# Loco-Manipulation — Paper Notes

Loco-manipulation papers differ mainly in where they place the hard problem: task planning, command interfaces, contact-rich control, data generation, or perception. Each entry centers that choice, explains how locomotion and manipulation are coupled, and comments on whether the reported evidence really supports the proposed level of generality. Local full texts are used where available and primary paper/project sources for newer entries.

## Accelerating and Scaling MPC-Guided Reinforcement Learning for Humanoid Locomotion and Manipulation

This paper is less about inventing a new humanoid policy architecture than about making model-predictive guidance cheap enough to use inside large-scale RL. Ordinary centroidal MPC can provide valuable future center-of-mass (CoM), momentum, contact, and constraint information, but solving thousands of horizon QPs independently during PPO training is prohibitively expensive. The paper's πnMPC solver avoids repeatedly constructing sparse QPs. It applies a three-block ADMM formulation whose horizon updates are parallelized and whose small matrices can be precomputed, allowing one MPC instance per simulated environment.

The important distinction is between **training-time guidance** and **deployment-time control**. During training, centroidal MPC predicts CoM and angular-momentum trajectories over the horizon. Those predictions become landmark-shaped PPO rewards rather than runtime policy inputs. At deployment the MPC is removed; the actor is a conventional feed-forward Gaussian policy with hidden sizes `(512, 256, 128)`, ELU activations, and a learned scalar action standard deviation. It outputs 29 joint-position targets consumed by the robot's joint-level PD loops.

The deployable actor is deliberately proprioceptive. Its observation contains trunk angular velocity from the IMU (3), projected gravity (3), 29 joint positions relative to the nominal pose, 29 joint velocities, the preceding 29-D action, commanded planar velocity and yaw rate `(vx, vy, wz)`, and a two-value gait clock `(sin φ, cos φ)`. No camera, LiDAR, height map, or object pose enters this policy. Uniform observation noise is injected during training—for example ±0.2 rad/s on angular velocity, ±0.05 on projected gravity, ±0.01 rad on joint angle, and ±1.5 rad/s on joint velocity. The asymmetric critic sees clean proprioception plus base linear velocity, foot clearance, and MPC-predicted CoM/angular momentum. Thus MPC affects value learning and reward shaping without becoming a hardware dependency.

PPO is implemented in `rsl-rl`. The MPC stack replaces ordinary velocity-tracking and angular-momentum shaping with future-aware supervision. The strongest evidence is not just nominal gait tracking: hardware trials include treadmill locomotion, pushes, an unseen 10 kg load/vest, and pushing a roughly 290 kg cart. These tests support the claim that the landmark reward teaches a more useful dynamic prior than instantaneous tracking rewards.

The limitation is equally clear. The learned actor has no exteroception and no explicit payload estimate; it reacts through proprioception after load/contact changes appear. MPC guidance is only as good as the centroidal model, contact schedule, and objective used during training, and removing MPC at deployment discards online constraint enforcement. This is best read as an efficient way to transfer model-based structure into a fast neural controller—not as an MPC safety guarantee at runtime.

### Why πnMPC is the actual contribution

Many “MPC-guided RL” papers are conceptually straightforward but computationally awkward: the MPC becomes the slowest operation and prevents the thousands of parallel environments needed for modern locomotion RL. The construction-free, horizon-parallel solver changes that scaling relation. This makes the paper useful beyond its particular reward: future work can use the same batched solver for contact scheduling, payload-aware objectives, or manipulation constraints without accepting CPU-QP throughput.

The choice to provide MPC through rewards rather than observations is also consequential. It prevents a train–test input mismatch and keeps deployment simple, but it asks the policy to compress a family of predictive solutions into static weights. An alternative would distill the MPC trajectory explicitly or keep a small online residual MPC. That could preserve adaptation to new payloads and constraints while retaining most of the neural speed.

### Assessment and next experiments

The load and cart tests are persuasive evidence of dynamic robustness, but they do not isolate whether improvement comes from predictive CoM guidance, angular-momentum shaping, or simply a better-tuned reward. A particularly valuable ablation would compare equal-compute learned rewards, explicit MPC-reference distillation, and online MPC residuals. Perception-conditioned contact changes would also test the “manipulation” claim more strongly: the present actor does not observe an object, desired hand wrench, or terrain. Extending the MPC state with object dynamics and distilling uncertainty or constraint margin—not just nominal landmarks—would make the method a more complete loco-manipulation framework.

### End-to-end information flow and practical failure modes

The training loop can be read as `command + simulated state → centroidal MPC landmarks → reward → PPO actor`, whereas deployment reduces to `command + noisy proprioceptive history at the current instant → 29 position targets`. There is no recurrent or transformer memory in the actor and no MPC trajectory encoded as an input. Any anticipation at runtime is therefore stored implicitly in the network weights and gated mainly by gait phase and command. The asymmetric critic can use privileged linear velocity, foot clearance, and MPC predictions to reduce variance without making the deployed observation larger.

That division predicts its characteristic failures. It should be strong when a disturbance resembles loads represented by randomized training and when recovery is inferable from current joint/IMU state. It should be weaker before an unseen contact occurs, when a camera or force estimate could have provided advance warning; during terrain changes not encoded in projected gravity; and when a task requires choosing a new hand or foot contact rather than merely resisting a load. The cart result is impressive dynamic evidence, but the policy is still a blind forceful locomotor. Calling the method loco-manipulation is reasonable for coupled whole-body force production, though not for perceptual manipulation or object-conditioned planning.

## CLONE: Closed-Loop Whole-Body Humanoid Teleoperation for Long-Horizon Tasks

CLONE addresses a specific teleoperation failure: local-pose imitation can look correct moment to moment yet drift meters away from the operator's intended global path during a long walk–reach–squat task. Its control interface is intentionally sparse. Apple Vision Pro supplies the 6-D poses of both wrists and the 3-D head position; those three points are the entire human command. On the robot, LiDAR odometry estimates global head/wrist locations through robot pose plus forward kinematics, closing the translational loop.

Training is teacher–student RL rather than behavior cloning from headset commands alone. The privileged teacher is an MLP trained with PPO/AMP. It observes every link's pose, linear and angular velocity, reference-state errors, the next reference state, and randomized environment parameters such as friction and mass distribution. Its output is a 29-D vector of joint-position targets for PD control. The student is distilled with DAgger and must operate from deployable signals.

The student input is unusually well specified. It receives 25 frames of joint positions, joint velocities, root angular velocity, and projected gravity from the onboard IMU, together with 25 previous actions. Its task block contains current-to-target errors for the three tracked points, target point positions and velocities, and current/target wrist orientations. This history lets the policy infer unobserved motion and compensate delayed/noisy global feedback. LiDAR updates at 10 Hz while the learned controller runs at 50 Hz; joint PD runs at 1 kHz. The policy consumes position error directly, so successive outputs reduce accumulated global drift rather than merely reproduce local body shape.

Architecturally, the student has three layer-wise mixture-of-experts blocks. Each block contains four feed-forward experts with dimensions `(2048, 512, 512, 256)`; a router selects the top two experts and combines their outputs. An AMP discriminator is a three-layer `(256, 256, 256)` MLP. The chosen 25-frame history and 3×4 MoE configuration are supported by ablations: shorter history loses temporal information, longer history increases dimension and optimization difficulty, and too many experts become redundant. The paper reports teacher training with 8,192 Isaac Gym environments and student training with 4,096.

CLONED expands AMASS through retarget augmentation, motion editing, wrist sampling, and 14 custom sequences so the controller sees hand orientation and task-relevant combinations absent from generic mocap. This matters for loco-manipulation: the resulting policy can connect walking, crouching, reaching, and object-oriented upper-body motion without a separate gait/arm state machine.

The central contribution is therefore closed-loop global correction inside a broad whole-body motion prior. The price is infrastructure and ambiguity. Sparse head/wrist targets do not specify elbow style, foot placement, contact force, or finger motion; the policy fills these from its training distribution. LiDAR localization loss or latency directly corrupts the error signal, and large corrections can demand motions outside the MoE's learned support. See `Teleoperation_Data/note.md` for the interface/data perspective.

### What CLONE changes relative to sparse pose tracking

The high point is not merely using an MoE. OmniH2O-like sparse tracking already demonstrates that head and hands can command a humanoid. CLONE adds the missing global feedback path and trains the policy to *correct* error smoothly rather than assuming its integrated local motion remains aligned. This is especially important for collecting long-horizon task data: a locally convincing trajectory is unusable for manipulation learning if the hands arrive at the wrong table.

Layer-wise routing is a reasonable match to heterogeneous motion because early and late representations can specialize differently. The ablation showing redundant experts beyond four is more informative than simply reporting a large MoE. However, top-2 routing can still switch discontinuously between modes, and the paper does not establish that experts correspond to semantically stable skills rather than statistical partitions.

### Research opportunities

The next step is to close more than position. Wrist force/tactile feedback could correct contact, hand observations could control fingers, and scene vision could prevent a globally corrected body from colliding with objects. Localization uncertainty should be fed to the policy so corrections are attenuated when LiDAR confidence drops. A learned operator-intent model could also replace some of the 25-frame history with a forecast of upcoming head/hand motion. Finally, evaluating downstream autonomous policies trained on CLONED trajectories would establish whether global correction improves data value, not only teleoperation appearance.

### Architecture interpretation and closed-loop behavior

The 25-frame state window and mixture-of-experts solve different ambiguities. History exposes recent velocity, tracking error, and command evolution; top-2 routing gives conditional capacity for regimes such as standing reach, stepping, crouching, and recovery. Three stacked MoE layers allow specialization at several representation levels rather than selecting one complete controller. Only two of four experts are active in each block, keeping inference cheaper than an equivalently large dense network.

The multi-rate loop is important: LiDAR updates at 10 Hz, the network at 50 Hz, and joint PD at 1 kHz. Geometry is stale between scans, so the policy relies on proprioceptive continuity. That suits static interiors but risks delayed reaction to moving people or objects. LiDAR supplies distance and shape independent of texture, yet does not identify grasp semantics or contact force.

CLONE should not be judged only by robot–operator pose error. Closed-loop remapping may intentionally alter pose to preserve balance or global hand arrival. Future evaluation should separate intent error, global drift, collision margin, and balance margin. Routing plots across tasks would also show whether experts acquire stable roles; without them, MoE specialization remains a plausible explanation rather than a demonstrated one.

## CEER: Compliant End-Effector and Root Control as a Unified Interface for Hierarchical Humanoid Loco-Manipulation

CEER starts from an interface-design observation: a high-level planner can readily specify where the pelvis and hands should go, but it is unreasonable to ask an LLM, diffusion planner, or teleoperator to synthesize a stable 29-joint trajectory. CEER therefore defines a 16-D command containing root `(x, y, z, yaw)` and end-effector position/orientation targets. The low-level policy converts that task-space command into a full 29-D joint-position command.

Compliance is learned through an impedance-consistent reference construction. During teacher training, external interaction is modeled as a spring-like disturbance and the reference is displaced according to stiffness `Kp^-1 f_ext`; the resulting compliant target is both privileged input and reward target. This encourages the controller to yield under contact rather than fight to restore an exact pose. The formulation is not a runtime analytical impedance controller: the final neural policy imitates the behavior induced by this training model.

The teacher is a general motion tracker trained with PPO from human-motion data. It sees a 29-D joint reference, deployable proprioception, and privileged state including the impedance-consistent target. The student sees only its 16-D EE/root command and deployable proprioception. The shared proprioceptive block contains projected gravity, root angular velocity, a `K`-step history of joint positions, and `L` previous actions. The paper defines these histories symbolically rather than giving their numeric lengths in the main method; they should not be guessed. Both teacher and student output 29 joint commands.

CEER uses two estimators to keep the student structurally compatible with the pretrained teacher. A privileged-state estimator predicts unavailable teacher information from proprioception, while a target-joint estimator maps `(proprioception, EE/root command)` to a 29-D implied reference. The student can then inherit teacher weights rather than relearn stable locomotion from scratch. Its reward combines end-effector accuracy, compliant interaction, motion imitation, and locomotion stability. This is an important architectural detail: CEER is not merely inverse kinematics followed by tracking; it learns the underdetermined posture and balance solution from the motion prior.

The system above CEER has three levels. Mid-level modules—VR/keyboard teleoperation, move-to-position, grasp/place, or learned diffusion/RL skills—run around 10–20 Hz and all emit the same EE/root command. An LLM task manager invokes and sequences those modules on a seconds-scale horizon. CEER remains the high-rate whole-body layer. Reported results include about 3.3 cm end-effector error, lower jerk than alternatives, stable hardware interaction, and up to 70% success on simulated room-scale single-object tasks.

The abstraction is genuinely planner-friendly, but it also throws information away. Hand force, contact mode, finger configuration, obstacle clearance, and a unique whole-body posture are not present in 16 dimensions. The learned prior resolves ambiguity only within its data distribution. Likewise, compliance is demonstrated empirically rather than guaranteed by a passivity or force bound. CEER is strongest as a modular motor interface, not as a complete manipulation planner.

### Why the command-space decision matters

CEER differs from dense motion trackers and velocity-plus-arm hierarchies by making both locomotion and manipulation communicate through one geometric task space. That is a genuine systems advantage: a teleoperator, diffusion planner, and scripted skill can share the low-level controller without agreeing on robot joints. The target-joint estimator is the technically important bridge—it restores the dense-reference structure expected by the teacher while letting the upstream module remain sparse.

The compliance construction is clever but should be interpreted as behavioral supervision, not impedance identification. A spring disturbance model teaches plausible yielding for the training stiffness and force distribution. It does not guarantee stable energy exchange with arbitrary tools, humans, or delayed planners. The reported jerk and hardware contact results support qualitative compliance, while a force–displacement or passivity analysis would support a stronger claim.

### What would make CEER more complete

A natural extension is an augmented command carrying optional wrist wrench, contact state, or compliance tensor while preserving backward compatibility with the 16-D interface. Command feasibility prediction could warn a planner before it requests mutually impossible root and hand goals. Training on multiple embodiments would test whether EE/root space is truly robot-agnostic or merely planner-agnostic. Finally, exposing confidence from the privileged and target-joint estimators could let the high-level system slow down, replan, or request perception when the inferred posture is outside the motion prior.

### How the command becomes a whole-body action

CEER compresses a high-dimensional task into 16 values without discarding the root variables that make hand goals feasible. The actor combines that command with proprioceptive history and estimator outputs, then emits 29 position targets. During training, the privileged teacher has exact simulated information; at deployment, the estimators reconstruct the dense joint reference and hidden teacher features needed by the inherited control network.

This is not arm IK followed by an independent walk controller. Such a stack can receive individually valid root and hand goals that are jointly inconsistent, with neither module owning the compromise. CEER learns that compromise under one return. Its compliance is selective learned deviation, not a guaranteed analytical impedance, so behavior depends on disturbance and reward coverage.

The symbolic but numerically unspecified history lengths `K` and `L` are consequential: longer windows reveal load and velocity but increase dimension and lag; shorter windows alias slow effects. Exact lengths, estimator errors, and update rates are needed for reproducibility. A revealing stress test would deliberately issue infeasible root/hand combinations and measure whether CEER yields smoothly, abandons accuracy, or loses balance.

## GRAIL: Generating Humanoid Loco-Manipulation from 3D Assets and Video Priors

GRAIL attacks the data bottleneck by reversing the usual video-reconstruction problem. Instead of recovering unknown object geometry, scale, camera, and human morphology from arbitrary internet video, it first assembles a simulator-ready 3-D scene with known assets, a robot-proportioned character, camera intrinsics/extrinsics, and metric background depth. A VLM writes an interaction prompt and a video foundation model such as Kling animates the rendered first frame. The generated video supplies a rich motion prior while the known scene removes much of monocular ambiguity.

The 4-D reconstruction stack estimates SMPL-X human motion and object pose, then jointly optimizes them with keypoint reprojection, metric-depth alignment, contact, regularization, and temporal-smoothness terms. MoGe-2 depth is aligned to rendered background depth; SAM2 separates human/object regions for point-cloud losses; known object meshes allow FoundationPose-style tracking. Contact optimization is especially important because independent human and object estimates otherwise float, penetrate, or disagree along camera depth. Failed video/reconstruction samples are filtered, leaving more than 20,000 sequences.

GRAIL does not use one controller for every interaction. For object manipulation it freezes a pretrained FSQ-based whole-body controller—encoder, quantizer, and action decoder—and trains an object-aware adaptor. The adaptor input combines robot proprioception with body-frame object pose, hand-to-object transforms, finger contact forces, a basis-point-set shape code, and differences between future reference object pose and current simulated pose. It outputs a 64-D residual added to the controller's latent token (scaled by 0.1 before quantization) plus two binary open/close primitives, each mapped to seven finger joints. Motion tracking, object pose, and contact-gated grasp rewards train the adaptor with PPO; an L2 residual penalty keeps it near the locomotion prior.

For terrain and chair interaction, the system instead adds a local height-map encoder built from 2-D convolutions and fine-tunes the controller on reconstructed scene trajectories mixed with original flat-ground data. A parallel kinematic decoder reconstructs target motion as an auxiliary latent-regularization loss. Both tracks use reference-state initialization and motion rewards over root pose, body pose/orientation, and linear/angular velocities. Training runs in Isaac Lab with 1,024 environments per GPU on 64 L40 GPUs for 30,000 iterations—evidence that the method's scale is computational as well as algorithmic.

Those privileged trackers are not the final autonomous policies. Separate egocentric visual policies are distilled for pickup and stair climbing. At deployment, a Luxonis OAK-D W head camera supplies RGB and robot proprioception is streamed to an RTX 5090 desktop; the visual policy outputs SONIC latent tokens at 10 Hz, which the low-level controller decodes into actions. The paper reports roughly 84% hardware pickup success and 90% stair success.

GRAIL's strongest idea is to use generative video only for what it is good at—interaction/motion diversity—while fixing metric scene variables by construction. Its limitations follow the same pipeline: video models can alter appearance or invent contacts, reconstruction degrades with occlusion/fast motion, filtering discards a nontrivial fraction, and the deployed RGB policy no longer receives the privileged object pose/contact signals used to train the tracker. The open/close hand primitive also simplifies dexterity substantially.

### Where GRAIL is genuinely different

Most video-to-robot systems begin with uncontrolled footage and spend their complexity resolving unknown scale, camera, object mesh, and human shape. GRAIL uses a video generator inside a known 3-D world, so appearance and motion are sampled while geometry remains anchored. This is a strong reframing of synthetic data: the video is not ground truth but a proposal that a geometry-aware optimizer must validate.

The two controller adaptations reveal that “loco-manipulation” is not a single data problem. Object skills need shape, contact, and object-state residuals; terrain/scene skills need spatial geometry and posture adaptation. Preserving the frozen locomotion decoder for the former is a good defense against catastrophic forgetting, while full fine-tuning for terrain recognizes that feet and base motion must change more fundamentally.

### Judgment and future directions

The 20,000-sequence scale and physical deployment are strong, but filtering makes the effective data distribution important: if difficult occlusions and unusual contacts are removed, the policy may learn only the generator's easiest modes. Reporting yield by failure type and performance versus retained-data diversity would clarify the real scaling law. Future systems could close the loop by using failed hardware rollouts to reweight video prompts, add tactile/force supervision to validate generated contact, and replace binary hands with dexterous finger trajectories. Jointly training the visual student with the privileged adaptor—rather than distilling after the fact—could also reduce the gap between state-based tracking and RGB deployment.

### Reading the latent and perception choices

FSQ supplies a compact discrete-like vocabulary without maintaining a learned codebook. That can suppress noise in reconstructed motion and promote reuse across assets; the 64-D residual restores task-specific precision while its scale penalty protects the locomotion prior. This combination is well matched to imperfect video proposals. It can also become restrictive when a generated interaction requires a motion mode absent from the frozen controller.

BPS encodes object geometry relative to fixed basis points, and the height-map CNN encodes local terrain. These are structured geometric observations, not raw perception. Consequently, the privileged controllers test whether geometry-aware control generalizes after pose/shape estimation has already succeeded. The later RGB student is what tests onboard perception, and the distillation gap between them should be reported explicitly.

The strongest compositional test would combine a new object, terrain, and ordering of familiar contacts. Latent traversal and code-usage analysis could reveal whether FSQ levels align with approach, grasp, transport, and release or simply partition clip identity. Failure accounting should separate video-generation error, 4-D reconstruction, contact cleanup, state estimation, and residual-control failure; otherwise an aggregate success rate cannot identify the pipeline bottleneck.

## HAF: Adapting Generalist VLAs to Humanoid Whole-Body Loco-manipulation via Hierarchical Action Flow and Spectral Latent RL

Source: [arXiv 2608.16837](https://arxiv.org/abs/2608.16837).

HAF asks how to reuse a pretrained flow-matching VLA on a humanoid without either generating all body joints in one unstable shot or fine-tuning the enormous backbone through risky real-world RL. It answers with two separable components: HAF-VLA changes how whole-body action chunks are generated, while HAF-Steer changes where post-training RL explores.

The VLA observation is an egocentric RGB image, robot proprioception, and a language instruction. It predicts a 100-step whole-body action chunk and executes the first 40 steps before replanning. Each action is explicitly partitioned into locomotion/skill mode, head orientation, waist posture, and bimanual manipulation. Three flow stages use nested action sets: stage 1 generates locomotion and head; stage 2 adds waist; stage 3 adds both arms/hands. Later stages may revise earlier dimensions rather than simply append them.

All stages share one action-flow expert. A visual-language prefix KV cache is computed once, while a learned stage embedding identifies the current generation stage. After each early stage, the clean denoised action is re-encoded into a cross-stage action KV cache that conditions the next stage. Training uses masked stage-wise flow-matching losses and teacher-forced ground-truth caches to avoid cascading errors; inference uses predicted caches and ten flow-denoising steps per stage. Only the final stage is executed. Reported three-stage latency is about 0.12 s on an RTX 5090.

HAF-Steer freezes that VLA and treats its initial flow noise as an RL action. Demonstrated action chunks are numerically inverted through the frozen flow field to recover their initial noise. A temporal discrete-cosine transform keeps the first eight coefficients, reducing exploration from `H × D` to `8 × D` and suppressing high-frequency jerking. A stochastic actor first behavior-clones these expert spectral codes, then mixed offline/online Soft Actor-Critic updates the actor and twin critics. Offline expert batches retain a BC penalty; the offline sampling ratio decays as real experience accumulates. Rewards are sparse—one on successful termination, zero otherwise—and the VLA weights never change.

The paper evaluates two physical humanoids on seven household loco-manipulation tasks and reports better whole-body coordination and success than single-stage VLA baselines, including out-of-distribution disturbances. This supports the hierarchical-generation claim more directly than a tabletop-only benchmark would.

The hierarchy is sensible but expensive: three sequential denoising passes increase inference latency, teacher forcing means later stages do not see the full distribution of early-stage errors during training, and a bad base decision can still poison manipulation through the cache. Spectral RL is safer and lower-dimensional than raw action exploration, but it can only steer behaviors the frozen flow model can express; eight DCT modes deliberately discard sharp temporal corrections. HAF is best understood as an adaptation mechanism for an existing generalist VLA, not a new low-level stabilizing controller.

### The architectural bet

HAF argues that humanoid coordination should be reflected in generation order: establish locomotion/gaze, shape the waist workspace, then solve bimanual detail. This differs from both a fully decoupled hierarchy, where later modules cannot revise base decisions, and one-shot action generation, where every joint competes inside one denoising problem. Nested action sets and editable earlier dimensions are the most convincing part of the design.

Spectral steering tackles a separate problem: online RL should alter the *sampling trajectory* of a competent foundation model rather than its billions of weights. DCT truncation is more expressive than repeating one latent across time and smoother than optimizing independent noise at every step. The offline inversion of demonstrated actions also gives SAC a sensible initialization instead of asking it to discover valid flow noise on hardware.

### Evidence, weaknesses, and what comes next

Seven real tasks on two humanoids are valuable, but comparisons should separate gains from three-stage generation, extra inference compute, and spectral RL. Evaluating equal-latency one-stage baselines and varying stage order would test the kinematic hypothesis. Student-forcing or scheduled sampling across stage caches could reduce train–test mismatch. Adaptive spectral bandwidth is another obvious extension: low frequencies for transport, more coefficients near impacts or grasps. Safety critics, uncertainty-aware exploration, and low-level feasibility feedback into the VLA would make real-world online improvement more defensible.

### Generation horizon, feedback, and spectral limits

HAF generates 100 steps and executes 40 before replanning. The long horizon gives the flow model space for coordinated walking and manipulation, while partial execution limits commitment. Ten denoising steps per each of three stages and a reported 0.12-second latency mean the low-level controller must maintain stability while the semantic policy computes. The DCT retains only eight temporal coefficients, strongly favoring smooth low-frequency changes; balance reflexes and brief contact transients must be handled below this representation.

The three stages encode a kinematic hypothesis: base and gaze establish the workspace, waist refines it, then hands solve manipulation. Because later stages may revise earlier coordinates, this is less rigid than independent modules. But teacher-forced clean cross-stage caches during training differ from predicted caches at inference. A wrong locomotion sample can therefore propagate into every later decision even when the hand stage is otherwise competent.

Frequency-domain ablations should report contact timing and action spectra, not only success. Eight coefficients may be ideal for transport and insufficient for catching or impact. Varying the executed prefix from 40 steps would reveal the feedback/throughput tradeoff. Real-world sample count, reset/intervention count, and safety violations are also necessary to judge spectral RL's practical advantage over fine-tuning or residual action correction.

## HANDOFF: Humanoid Agentic Task-Space Whole-Body Control via Distilled Complementary Teachers

HANDOFF treats the boundary between an agent and a whole-body controller as the research problem. Dense motion references are expressive but hard for an agent to construct. Its planner-facing command is only ten dimensions: planar base velocity, yaw rate, desired root height, and two pelvis-frame 3-D wrist targets. The controller must infer all 29 joint commands, gait phase, posture, and recovery behavior from that compact request.

No single teacher covers the command space well, so three PPO specialists are trained. A 29-DoF whole-body tracker learns reach, posture, and squat from retargeted human motion. Before training, dynamically unsafe squat frames are projected with a closed-form control-barrier-function correction that restores static center-of-pressure margin. A 15-DoF legs/waist locomotion teacher tracks velocity on flat ground; randomized/curriculum-blended arm motion makes it robust to upper-body CoM shifts. A 29-DoF AMP teacher learns locomotion plus paired fall/recovery clips, with some environments initialized in delayed fallen states.

The deployable student consumes the 10-D command plus an 11-frame proprioceptive history and outputs 29 joint targets. Its soft mixture-of-experts head has three experts—one per teacher—over a shared encoder. Supervision is action-sliced and context-dependent. For legs/waist, a sigmoid of commanded speed blends KL targets from the whole-body and locomotion teachers around 0.1 m/s. Arms follow the whole-body teacher. When a recovery flag is active, the fall-recovery teacher supervises the complete action. PPO, KL losses, MoE load balancing, and a routing loss are optimized together. This is not ordinary ensemble averaging: teacher authority changes by body part and regime.

The teacher observations clarify what is distilled away. The whole-body teacher sees 11 frames of deployable proprioception plus the full 29-D reference, while its critic also sees base velocity, reference root pose, key-body positions, and randomization parameters. The locomotion teacher actor uses commanded velocity, projected gravity, base angular velocity, joint state, previous action, and a four-value gait-phase block. The student retains only command and proprioceptive history.

For agentic demonstrations, natural language is decomposed into atomic tasks. A VLM operating around 0.1 Hz projects 2-D detections into an RGB-D point cloud to produce pelvis-frame waypoints; a waypoint follower generates base commands and wrist goals for HANDOFF. Thus RGB-D and language belong to the planner, not the low-level policy. The controller itself is vision-free and can be reused with other planners.

Hardware evidence on Unitree G1 includes competitive velocity tracking, a large robust wrist workspace, natural-language task rollouts, and fall recovery without task-specific controller fine-tuning. The compact interface is the achievement, but also the limitation: it specifies no wrist orientation, finger configuration, desired wrench, contact schedule, or obstacle geometry. Those must be supplied by a mid-level skill or silently inferred by the motion prior. The binary recovery flag also assumes some external failure detector or planner logic.

### Why complementary teachers are necessary

The paper demonstrates a real conflict hidden by “unified controller” language. Human-motion tracking provides expressive squats and reaches but poor command-velocity coverage; locomotion RL provides reliable travel but no arms; recovery data occupies a radically different state distribution. HANDOFF does not pretend one teacher is universally correct. Its action-sliced KL says which specialist should control which body region under which regime, which is more principled than indiscriminate averaging.

The CoP projection of unsafe reference frames is another highlight. It improves the *data* before policy learning rather than expecting reward shaping to overcome dynamically invalid demonstrations. Still, static CoP margin is only a proxy for dynamic feasibility and cannot validate fast angular momentum or contact transitions.

### Technical judgment and extensions

The 10-D interface is excellent for waypoint-level agents, but its success partly relies on the controller hallucinating unspecified motion from its priors. A useful next version would make command fields optional and masked: add wrist orientation, hand state, contact/wrench, or gaze only when the planner knows them. Learned feasibility/confidence could reject bad commands. Automatic recovery detection should replace the externally supplied flag, and scene-aware collision observations could prevent large wrist workspaces from intersecting the environment. More importantly, long-horizon task success should be measured against denser interfaces to determine when compactness helps planning enough to offset lost control authority.

### From compact intent to coupled motor behavior

The student maps a 10-D task command and 11-frame proprioceptive history into 29 joint targets. Three teachers cover distinct state distributions; action-sliced KL chooses which expert is authoritative for which body region and regime. This is more precise than averaging their full distributions, which could let locomotion supervision erase reach posture or motion imitation weaken recovery.

The fixed slices nevertheless impose an approximate modularity on a non-modular body. Arm acceleration shifts whole-body momentum, waist posture changes reach, and hand contact changes desirable foot stiffness. Shared features and PPO can reconcile some conflicts, but transitions between walking, reaching, contact, and falling are the decisive cases. Per-regime metrics and transition-conditioned failure plots would be more informative than aggregate tracking.

The RGB-D/VLM layer runs around 0.1 Hz and supplies waypoints rather than continuous feedback. It therefore handles semantic sequencing, while HANDOFF handles rapid physical execution. A feasibility value returned from the motor policy would let the planner revise unreachable wrist/root combinations. Visualizing expert routing, ablating each teacher, and testing contradictory commands would clarify whether the student actually combines competence or mostly selects one dominant expert.

## HDMI: Learning Interactive Humanoid Whole-Body Control from Human Videos

HDMI turns a single human–object video into a task-specific physical interaction policy. GVHMR recovers the human, LocoMujoco retargets it to the robot, and object pose/contact are post-processed into a synchronized reference. Each frame contains robot root pose/orientation and joint angles, object pose, contact flags, and desired contact points expressed in the object's local frame. The same structure supports rigid and articulated objects and contacts by hands, feet, or other end effectors.

The controller is trained with DeepMimic-style PPO. Episodes begin from randomized reference frames, receive a normalized phase `φ ∈ [0,1]`, and terminate after large robot/object tracking error or persistent contact loss. Runtime observation combines robot proprioception, phase, root-frame object pose, and root-frame reference contact points. The tracker itself consumes no RGB or depth; real deployment requires an external or task-specific estimate of object pose and contact targets.

Actions are residual joint-position targets: `q_target = q_reference + a`. Centering exploration around the reference is crucial for kneeling and configurations far from nominal standing. Rewards track local body pose, global root pose, body velocity, joints, and object pose, with penalties for action rate, limits, torque, foot impacts/slip, and air time. A general interaction reward activates only during annotated contact and combines end-effector proximity with sufficient-but-bounded contact force. Robot/object inertia and friction are randomized.

The elegance is that task semantics enter through a trajectory and contact annotation rather than a custom semantic reward per door, box, or ball. Hardware tests include varied box transport and 67 consecutive door trips. Yet HDMI remains phase-indexed reference imitation: it does not decide when or why to interact, and cannot autonomously recover object state from vision. Monocular reconstruction and manual contact cleanup are upstream bottlenecks; RL can robustify a reference but cannot repair a fundamentally wrong contact sequence.

### The most reusable idea

HDMI's unified contact reward is more consequential than any one task. Pose tracking alone can reproduce the appearance of a push while applying force at the wrong place or losing contact. By expressing desired contact in the object frame and gating it with reference contact state, the same objective covers doors, boxes, and foot interaction without naming the task. Residual action around the retargeted pose then makes physically difficult references learnable without erasing their style.

The 67-trip door test is convincing evidence of repeatability, not just a curated success video. However, most robustness is demonstrated under a known phase and object state. A fairer autonomy test would perturb timing—delay the door, move the box during execution, or interrupt contact—and measure whether the policy resynchronizes rather than merely survives.

### Next steps

Replacing phase with event/contact-conditioned progress would allow speed changes and recovery after interruption. An RGB-D or tactile estimator could provide object/contact state, ideally trained jointly with the controller so uncertainty affects action. Multi-clip skill models could share approach and recovery behavior across objects instead of training one policy per interaction. Finally, force-aware retargeting should be incorporated upstream: current reference videos specify kinematics, while the interaction reward must infer all physically meaningful force behavior during RL.

### End-to-end representation and evidence

Object-frame contact targets remove irrelevant room position and heading, so repeated interactions share a consistent representation. Phase, robot state, object-relative state, and contact targets enter the actor; it produces a residual around the retargeted joint reference. PPO therefore searches near a semantically meaningful motion rather than discovering a door push or box carry from scratch.

This prior is helpful when the human reference is approximately feasible and restrictive when the humanoid needs a qualitatively different stance or support transition. A phase clock can synchronize the clip, yet can also cause contact at the scheduled time even if the object is late. Because RGB/depth is absent at runtime, HDMI avoids visual latency after initialization but cannot directly recognize slip or displacement beyond the supplied object state.

The 67 consecutive door traversals provide strong repeatability evidence. They do not by themselves demonstrate recovery to changes in interaction timing. Tests should move the object after contact begins, delay resistance, vary hinge friction, or interrupt the robot and report resynchronization. Contact success, peak force, object error, and pose imitation should be separated: a visually accurate motion can fail physically, while a robust physical solution may deviate substantially from the video.

### Overall judgment

HDMI is strongest when a task has a stable interaction template but difficult whole-body physics. It turns one annotated video into a repeatable skill with little task-specific reward engineering. It is weaker when an environment requires semantic choice or a qualitatively different strategy. The controller receives the answer to “what phase and contact should happen now” and solves “how can this body realize it robustly?” A useful benchmark would connect a learned perceptual progress estimator to several HDMI skills and measure selection, switching, and recovery without external phase or pose input.

Compared with HumanX, HDMI stays closer to one recovered reference and emphasizes a reusable contact reward; HumanX expands the video into a wider distribution and studies privileged-to-blind execution. Compared with SUGAR or GRAIL, HDMI has a smaller data-generation stack but less autonomous visual generality. It is therefore a good choice when one wants a reliable, physically grounded specialist quickly and can provide object state and progress externally—not when one needs an open-ended household policy.

## HOMIE: Humanoid Loco-Manipulation with Isomorphic Exoskeleton Cockpit

Source: [arXiv 2502.13013](https://arxiv.org/abs/2502.13013).

HOMIE separates what a person can command precisely from what RL handles better. A foot pedal specifies forward speed, yaw rate, and desired torso height. Two seven-DoF isomorphic exoskeleton arms map joint angles directly to the humanoid upper body, and Hall-sensor gloves capture up to 15 finger DoFs. The lower-body policy stabilizes and walks around these externally imposed configurations; it does not generate the operator's manipulation motion.

The PPO locomotion policy observes six frames (`O[t-5:t]`). One frame contains the three-value command, torso angular velocity, projected gravity, all joint positions and velocities, and the preceding action. Outputs correspond one-to-one with lower-body joints and become targets in a PD torque law. There is no camera, LiDAR, object pose, or motion-reference input to this policy. First-person robot video closes the human loop, not the learned-controller loop.

An upper-body curriculum gradually expands randomly changing arm/torso poses; targets are resampled each second and interpolated so the legs tolerate dynamic mass shifts. A knee/height reward teaches continuously commanded squats, with one-third of environments assigned squat training and the rest stand/walk transitions. Mirrored rollout transitions plus actor/critic symmetry losses reduce left/right bias. Unlike motion-prior trackers, this policy needs no mocap corpus.

The hardware is part of the contribution: isomorphic correspondence avoids iterative IK and vision-pose latency, while the pedal frees both hands. Collected demonstrations can train an autonomous policy that later predicts pedal/body commands and upper-body targets. Precision comes from embodiment matching, which is also the limitation: a new robot may need redesigned hardware. Pedal commands omit natural leg/contact intent, and the vision-blind lower body cannot independently avoid terrain or obstacles.

### Why HOMIE is not “just another teleoperation rig”

The key systems insight is to avoid solving problems unnecessarily. Joint-isomorphic arms eliminate pose estimation and IK; a pedal communicates the few locomotion variables the operator actually wants; RL absorbs balance coupling. Compared with full-body visual shadowing, this gives high-frequency manipulation and an effectively unbounded walking workspace without requiring the operator to physically walk in place.

The six-frame input is short and practical, but it mainly estimates local dynamics. Because upper-body targets are imposed directly, the lower-body policy must tolerate arm configurations it cannot veto. The curriculum demonstrates that random pose exposure can work, yet collision, self-contact, or torque-infeasible arm commands remain supervisory concerns.

### What should follow

A portable mapping layer between similar but non-isomorphic robots would trade a little precision for wider reuse. Adding foot-pressure or estimated payload to the lower-body policy could improve stability while carrying. Scene depth could modify pedal commands for obstacle avoidance without removing human authority. The most important evaluation would compare autonomous policies trained from HOMIE, VR, and video demonstrations at equal data time; that would establish whether faster teleoperation actually produces more learnable data rather than merely quicker task completion.

### Detailed control loop and likely bottlenecks

HOMIE closes two loops at different conceptual levels. The human closes the semantic and manipulation loop by watching the robot and moving an isomorphic upper-body rig; the learned policy closes the balance/locomotion loop by translating pedal commands, six-frame proprioception, and upper-body configuration into lower-body joint targets. Because arm angles map directly, the interface avoids accumulating inverse-kinematics and hand-pose-estimation error. Because the operator commands velocity rather than absolute base position, the robot can traverse farther than a room-scale motion-capture setup.

The cost is that the operator and lower-body policy share the same physical body without an explicit negotiation channel. Fast arm motion or a heavy payload changes angular momentum and center of mass, but the leg policy sees these effects only through configuration and recent proprioception. Six frames can expose fast local trends; it is less suited to identifying slowly changing payload mass or planning around future arm motion. Direct joint correspondence also transfers poorly when shoulder axes, link lengths, joint limits, or hand kinematics differ.

A useful safety extension would let the locomotion controller return a feasibility margin to the operator interface, slowing or projecting upper-body commands that threaten balance while preserving transparency. Evaluation should report command latency, operator workload, foot slip, intervention count, and demonstration diversity in addition to task time. Those measures would reveal whether isomorphism improves the quality of collected control data, not only the subjective ease of teleoperation.

### Overall judgment

HOMIE is a data-acquisition and human-control interface, not an autonomous manipulation policy. Its contribution is a transparent allocation of authority: the operator owns semantics and fine upper-body motion, the pedal owns mobility intent, and RL owns lower-body balance. No vision model must guess the intended grasp. The open scalability question is whether one morphology-specific cockpit remains worthwhile at dataset scale. Calibration time, operator learning curves, fatigue, and cross-operator variance should be reported with collection rate, because they determine whether the speed advantage persists beyond demonstrations by expert authors.

Relative to Vision Pro keypoint systems such as CLONE, HOMIE sacrifices hardware portability for direct joint precision and low latency. Relative to HuMI, it occupies the robot during collection but observes the actual embodiment and contact outcome. These are not minor implementation differences; they decide which errors enter the dataset. Isomorphic teleoperation minimizes mapping error, while portable or sparse interfaces minimize setup and robot time. A fair comparison must price both the apparatus and the downstream cost of correcting noisy or infeasible demonstrations.

## HOIST: Humanoid Optimization with Imitation and Sample-efficient Tuning for Manipulating Suspended Loads

HOIST studies an externally suspended load that the humanoid can influence only through intermittent whole-body contact. Delayed pendulum motion makes “push, then stop at target” qualitatively different from grasped pick-and-place. Pure imitation is safe but inaccurate; real-world RL from scratch is unsafe. HOIST therefore learns only the high-level command generator and keeps stabilization fixed.

The VLA observation contains onboard egocentric RGB and depth, an external side-view RGB image showing payload versus target, joint/base/IMU proprioception, language, and the previous navigation command. It assumes neither force sensing nor explicit payload-state input. The side camera is thus an important experimental dependency, not an ego-only solution.

Output is an `H`-step chunk of increments to head pose, left/right hand targets, planar navigation command, and desired base height—not torque or joint angles. The first increment updates the accumulated planner command, and a frozen GR00T whole-body controller maps that command plus robot state to motors.

The high-level model modifies GR00T N1.6. Most VLM layers remain frozen; the final four VLM layers, flow/diffusion-transformer action expert, and new depth/ego-motion encoders are fine-tuned on PICO VR demonstrations. After imitation, autonomous rollout batches receive a simple terminal translation/yaw reward. Logged initial flow noise trains a deterministic offline actor–critic steering module; cached VLM features are reused and the VLA itself remains frozen.

Reported refinement reduces translation error by 19.9 cm and raw angle error by 3.56°. The contribution is a sample-conscious route from safe demonstration to task optimization. It is not end-to-end humanoid RL: the external view and fixed WBC absorb much of the problem. New cable lengths, masses, suspension points, and occluded side views remain important tests, while the terminal reward says little about peak force or hardware wear.

### What is novel about the learning loop

The notable idea is to steer the initial noise of a frozen flow policy. This keeps the safe imitation manifold and avoids backpropagating through or catastrophically changing GR00T. Iterative batched collection is closer to conservative policy improvement than conventional online exploration: deploy, label a small batch, update the compact steering actor, and repeat.

The multimodal input is also well matched to the physics. Ego depth sees upcoming hand contact, the side view resolves global payload swing, proprioception reports the humanoid's ability to respond, and previous navigation command disambiguates current base motion. Removing any of those changes observability rather than simply model capacity.

### Critical reading and future work

The paper should be judged as a specialized systems result, not a general hoisting foundation model. Its strong error reduction is meaningful, but the sample count, safety interventions, and variance across load dynamics matter. Future work should learn a latent payload state from temporal images, remove the external camera, include force/torque or tactile feedback, and use risk-sensitive rewards for impact and cable tension. A model-based swing predictor could also provide dense counterfactual targets to the noise-steering critic, improving learning from few physical trials.

### Signal path, observability, and evidence boundaries

At execution, images, proprioception, language, and the last navigation command enter the high-level GR00T-derived network; the resulting chunk updates head, hands, navigation, and height references; the frozen whole-body controller then turns those references into stable motion. The RL component changes the distribution of the flow model's initial noise, so it selects among behaviors already expressible by imitation rather than learning a new low-level reflex. This is why few real batches can improve terminal positioning without destroying walking.

The side camera deserves particular attention. Suspended-load control depends on payload displacement and velocity relative to the target. Ego vision may see the hands but lose the global swing; the side view supplies exactly that missing state. Accordingly, the result does not yet show that the humanoid can infer swing from onboard sensing. Similarly, the previous navigation command helps explain self-induced image motion but is not the same as a measured base trajectory.

Terminal translation and yaw improvements show better settling, yet they do not fully characterize a suspended system. Peak swing angle, settling time, cable tension, collision count, energy, and worst-case error across masses would expose whether the tuned policy is physically gentler or simply more accurate at the endpoint. Comparing noise steering against last-layer fine-tuning and residual command correction at the same real-data budget would also isolate why this particular adaptation mechanism is sample efficient.

### Overall judgment

HOIST gives a careful answer to policy improvement when direct physical exploration is dangerous: retain the imitation manifold and learn a compact selector around it. The suspended load is a strong benchmark because delayed dynamics expose reactive imitation. Generality still requires adapting across cable length, attachment point, mass, and target geometry without reinstalling an external camera. A latent system-identification module could infer these variables from motion history, with noise steering conditioned on a posterior over payload dynamics rather than treating every rollout as the same plant.

HOIST is closest in spirit to HAF-Steer: both preserve a large flow policy and optimize the initial-noise distribution rather than rewriting the backbone. HOIST, however, targets a narrow physical system with a terminal accuracy reward and strong external observability; HAF emphasizes a broader humanoid action hierarchy and spectral exploration. Comparing the two suggests a useful combination: infer a compact payload state, steer only low-frequency chunk components for safety, and allow a small high-frequency residual for contact correction.

## HumanoidMimicGen: Data Generation for Loco-Manipulation via Whole-Body Planning

HumanoidMimicGen is a demonstration generator rather than a new online controller. It segments a few successful simulator demonstrations into object-centric end-effector skills, adapts each skill to new object poses, and represents precedence/concurrency constraints as a DAG. A greedy topological scheduler executes compatible bimanual skills together—for example, both hands pick before either begins placing.

Ordinary MimicGen assumes task-space control, which is unsafe for balancing legs. Here arms, hands, and torso receive joint-position commands while a pretrained HOMIE policy accepts planar base velocity, yaw rate, torso height, and current/target upper-body configuration and produces feasible leg positions. All joints ultimately use position control.

For each skill, whole-body IK finds a configuration satisfying active end-effector targets. Planning then separates locomotion toward an intermediate configuration from stationary manipulation, after which the transformed source segment is replayed. Perturbed objects/motions widen coverage; failures are rejected. Generated episodes contain privileged state, camera images, proprioception, and joint actions, but the final policy behavior-clones deployable observations only.

The reported downstream model is a flow-matching transformer/VLA policy from one egocentric RGB camera. Real hardware uses a Luxonis OAK-D; upper-body commands run at 25 Hz, lower-body inference at 50 Hz, and position control at 200 Hz. Training uses batch 128 for 25,000 steps. Across nine tasks, sim-plus-real data improves success by about 20% over real-only data.

The method extends MimicGen to a walking body, but its phase decomposition favors stop-and-act behavior. It inherits source contact modes, IK/planner completeness, collision models, and privileged success checks. A generated state may also be visually ambiguous even if it is physically valid, leaving a gap between data generation and ego-camera imitation.

### Why whole-body planning is needed for data generation

For a fixed arm, transforming an end-effector trajectory with an object pose can be enough. For a humanoid, the transformed grasp may lie outside the balanced workspace; reaching it changes pelvis pose, which changes what locomotion must do. HumanoidMimicGen's IK–locomotion–stationary sequence explicitly repairs this missing connection. The partial-order graph is likewise important for bimanual data because naïve independent adaptation can separate synchronized contacts.

The reported sim-plus-real gain supports synthetic data utility, although aggregate success does not reveal which generated variations transfer. A diversity-versus-validity curve would be informative: more object perturbation expands coverage but increases planning rejection and visual/physical mismatch.

### Extensions worth pursuing

A contact-implicit whole-body trajectory optimizer could allow locomotion and manipulation to overlap rather than hard phase boundaries. Planner uncertainty and failure reason should be saved as training metadata so the visuomotor model learns where data are reliable. Active generation could target visual states where the current policy is uncertain instead of sampling object poses uniformly. Finally, replacing privileged success filtering with perception-based checks would reduce the mismatch between which demonstrations are retained and what the deployed policy can recognize.

### What is learned, what is planned, and what is copied

This division is essential for interpreting HumanoidMimicGen. Object-relative end-effector segments and their order are copied and transformed from seed demonstrations. Whole-body IK and locomotion planning generate a feasible connection between transformed segments. HOMIE supplies a learned balance-aware realization of base motion. Only after this synthesis is the ego-camera policy trained to reproduce behavior from deployable observations. Thus the final visuomotor model can look end-to-end, but its supervision embeds substantial privileged geometry and planning.

The DAG offers more than bookkeeping. It represents causal constraints such as “grasp before lift” while preserving concurrency when two hands can move together. Still, a greedy scheduler is not a task-and-motion planner: it does not search broadly over alternative grasps, contact orders, or recovery branches. Rejection filtering makes the retained dataset look clean but can hide regions where the generator repeatedly fails, biasing the policy toward easy arrangements.

The nine-task and sim-plus-real results support data amplification, but the most revealing analysis would report generated success rate, rejection reason, state coverage, and downstream benefit per synthetic hour for each task. It should also compare equal-sized real and synthetic sets and evaluate visual perturbations that do not alter geometry. These experiments would tell whether the gain comes from new physical configurations, extra image diversity, or simple dataset volume.

### Overall judgment

HumanoidMimicGen is persuasive because it amplifies a few successes while respecting locomotion feasibility, which arm-centric MimicGen does not. Its modularity makes failures attributable to transformation, scheduling, IK, locomotion, or imitation. The cost is dependence on designer-provided segmentation and a bias toward locomote-then-manipulate behavior. The next major step is automatic phase discovery plus contact-aware planning that overlaps walking and manipulation when safe, while retaining the DAG's interpretable causal constraints and explicit failure filtering.

Compared with OASIS, HumanoidMimicGen primarily expands *physical object arrangements and skill executions*, whereas OASIS can replay one execution under many visual conditions. Compared with GRAIL, it starts from successful simulator demonstrations instead of generative video. Those approaches are complementary: planned object-pose augmentation can create valid behavior, offline re-rendering can diversify observation, and a video prior can propose strategies not present in the seeds. Their rejection filters and provenance should remain attached so a learner knows which supervision is trustworthy.

## HumanX: Toward Agile and Generalizable Humanoid Interaction Skills from Human Videos

HumanX separates interaction-data construction (XGen) from robust execution (XMimic). XGen decomposes a video into pre-contact, contact, and post-contact phases, reconstructs/retargets human and object motion, and synthesizes plausible variations in mesh, size, initial state, and trajectory. Non-contact flight can be generated through physics; transition windows are interpolated so independently treated phases remain continuous.

XMimic uses asymmetric actor–critic PPO. A privileged teacher sees proprioception, body/reference features, object state, and external-force information and predicts a Gaussian joint-action distribution. A student combines PPO with teacher distillation but excludes privileged body/dynamics variables. It receives proprioception and optional object observations.

A distinctive design is external-force inference from joint-velocity history. Contact changes acceleration, so recent velocity makes part of the missing impulse observable. In No External Perception mode the policy removes object input and performs fast interactions “blind”; in MoCap mode a 14-camera system supplies object state for precise feedback. Policy/mocap run at 100 Hz and low-level PD at 1 kHz.

Disturbed initialization, interaction-prioritized sampling, transition emphasis, and object/frame noise prevent literal replay. Tests cover ten G1 skills including basketball, cargo handling, ball exchange, and reactive fighting. The key claim is that structured physical augmentation can turn sparse video into a useful control distribution.

“Blind” interaction is nevertheless prior-driven: the robot cannot know an unseen object's pre-contact position, only infer force after contact. MoCap results require an instrumented room. Reconstruction and chosen augmentation still bound generality, and the student may reproduce a physically robust maneuver without understanding its task semantics.

### HumanX's important conceptual split

XGen treats video as a seed for a *family* of physical interactions, not a trajectory to imitate exactly. XMimic then learns that family. This differs from residual tracking systems whose robustness comes mainly from motor noise: HumanX perturbs the object dynamics and phase structure that define the interaction itself. The No External Perception experiments probe how much can be encoded as timing plus proprioceptive reaction.

The force-from-velocity-history argument is physically plausible and useful, but only after contact. It should not be confused with object perception. Strong blind performance may also exploit repeatable launch timing or initialization, so tests with randomized unseen approach trajectories are especially important.

### Future research

An estimator combining vision before contact and proprioception/tactile signals after contact would provide both anticipation and robustness. Instead of fixed phase segmentation, a hybrid state model could infer pre-contact, impact, sustained contact, and release online. Dataset generation could maintain multiple plausible reconstructions from ambiguous video and let physics choose among them rather than commit early. Cross-object and cross-robot evaluation would also clarify whether the learned policy understands contact patterns or memorizes a particular morphology's response.

### Reading the two execution modes correctly

NEP and MoCap mode test different scientific questions. NEP asks whether a motion prior plus proprioceptive history can execute and react to a roughly timed interaction without object tracking. It is attractive for fast impacts because an external perception pipeline could add delay. MoCap mode asks how much accuracy returns when object state is measured. The gap between them is therefore an estimate of the value of feedback under the tested initialization distribution, not a contest between two otherwise identical autonomous systems.

The asymmetric teacher/student setup also deserves nuance. Privileged force and object information can shape a better action distribution during training, but distillation cannot give the student information that is absent at test time. It can only teach the statistically best response under ambiguity. If two visually hidden object trajectories require opposing reactions before impact, no amount of teacher quality resolves them from the same proprioceptive observation.

For agile interaction, averages can obscure rare catastrophic misses. Future evaluation should report phase-conditioned success, impact timing error, peak joint torque, recovery after deliberately mistimed contact, and sensitivity to object-trajectory entropy. These would clarify whether the history encoder estimates physical state or merely recognizes a familiar temporal script. Adding an abstention or safe-guard behavior when state is ambiguous may be more valuable than forcing a single aggressive action.

### Overall judgment

HumanX's central achievement is converting scarce video into a distribution of physical interactions instead of treating reconstruction as ground truth. Fast blind skills show how much can be carried by a prior and proprioceptive reaction, but also expose their information limit: hidden object state cannot be recovered before contact from robot motion alone. The promising design is therefore a hybrid that transfers authority from vision-based anticipation to tactile/proprioceptive feedback as contact begins, with uncertainty deciding when aggressive motion is safe.

HumanX is particularly suitable for short, fast interactions where perception latency can be comparable to the useful reaction window. It is less suitable for long manipulation sequences in which object state drifts unpredictably. HDMI offers tighter reference/contact tracking, while VAIC offers onboard visual feedback; HumanX explores the middle ground of strong physical priors and optional external feedback. A unified benchmark varying object unpredictability and perception delay would reveal where each design crosses from advantage to liability.

## Humanoid Manipulation Interface: Humanoid Whole-Body Manipulation from Robot-Free Demonstrations

Source: [arXiv 2602.06643](https://arxiv.org/abs/2602.06643).

HuMI collects demonstrations without occupying the humanoid. Portable UMI-style grippers/cameras record two-hand motion and vision, while a real-time robot IK preview lets the demonstrator adapt to target morphology. This removes robot reset and wear from task-data collection.

A high-level Diffusion Policy runs at 5 Hz. It receives two 224×224 RGB images from gripper-mounted GoPro Hero 10 cameras and three 20 Hz frames of lower-body joint angles. DINOv2 ViT-B/14 encodes vision. A flow-matching action model predicts 48 steps at 20 Hz—keypoint positions/orientations and gripper width—with ten denoising steps. The visual horizon is one frame; temporal planning resides in the action chunk.

The 50 Hz low-level controller outputs joint positions for built-in PD. A privileged PPO teacher sees joint pose/velocity, angular velocity, projected gravity, previous action, and dense whole-body reference/link errors. DAgger distills a student that receives 25 states and previous actions plus ten future waypoints sampled across two seconds. Visible end effectors get localized target pose/error; “blind” pelvis and feet use chunk-relative displacement. Vive trackers provide pelvis/global-height reference.

Adaptive end-effector rewards tighten tolerance for slow precision motion but relax during fast transitions; a constant tight reward destabilized squatting. Variable-speed augmentation teaches timing flexibility. Reported results include roughly threefold collection efficiency and 70% success on unseen scenes/objects.

The hierarchy is carefully aligned because the diffusion output exactly matches the student tracker's interface. Limitations include camera/gripper correspondence between collection and robot, IK-preview quality, reliance on global trackers, and 5 Hz replanning that may be slow for unexpected fast contact.

### The highlight: decoupling semantics from embodiment

HuMI's best design choice is that task demonstrations teach end-effector intent, while a separately trained motion controller teaches humanoid coordination. The person need not wear a full-body suit, and task-specific collection does not need paired robot motion. This is more scalable than teleoperating every joint and more physically grounded than asking a diffusion policy to output raw humanoid joints.

The adaptive tolerance is also a meaningful negative result: maximizing hand precision everywhere can reduce task success because the body needs freedom during fast balance transitions. This exposes why whole-body manipulation cannot be evaluated on end-effector error alone.

### What remains unresolved

The UMI grippers and robot grippers are deliberately matched, so “robot-free” does not mean embodiment-free. New hands, camera baselines, or tool geometries require calibration and perhaps new data. Future versions could learn equivariant view/hand representations, predict controller feasibility for each keypoint chunk, and use tactile feedback to correct contact. Moving from external Vive localization to onboard visual-inertial state would make the system genuinely portable. Faster asynchronous replanning or RTC could reduce the 5 Hz high-level reaction bottleneck.

### End-to-end timing and information contract

The hierarchy succeeds because its representations line up. Portable demonstrations provide two-hand visual observations and keypoint/gripper trajectories. The 5 Hz diffusion policy predicts 2.4 seconds of those targets (48 samples at 20 Hz). The locomotion student consumes ten future waypoints spanning two seconds plus 25 steps of state/action history and turns them into 50 Hz joint-position targets. The hardware PD loop then runs faster still. Each layer sees enough future context for smooth coordination while the higher layer avoids direct responsibility for balance.

That temporal contract also locates the risks. A long predicted chunk supports coordinated squats and reaches, but the high level observes only one visual instant and reacts at 5 Hz. The low level can stabilize tracking error but cannot change task intent if the object slips. Three lower-body frames given to the high-level policy provide only a short clue about body motion, whereas the student receives much richer history. External Vive tracking supplies the global pelvis/height reference that neither wrist image reliably contains.

The most informative ablations would vary action horizon, observation history, and replanning rate independently; compare DINOv2 features with task-trained visual features; and inject calibration shifts between portable and robot cameras. Reporting feasibility rejection, hand-error versus balance-error tradeoffs, and failures by perception versus control would show whether adaptive tolerance is solving true motor conflict or compensating for upstream uncertainty.

### Overall judgment

HuMI makes a strong scalability trade: collect task semantics with portable hand-centric hardware, then rely on a reusable controller for embodiment. It is more transferable than recording every joint, but less embodiment-neutral than “robot-free” first suggests because camera, gripper, and IK correspondence remain important. The next evaluation should hold demonstrator hours constant against direct robot teleoperation and compare task success, state coverage, reset burden, hardware wear, and how often the feasibility prior must distort human intent. Those measures would establish the real economic and data-quality benefit of collecting away from the robot.

HuMI complements HOMIE almost exactly: HOMIE maximizes embodiment fidelity during collection, while HuMI maximizes collection freedom and delegates embodiment later. CLONE uses an even sparser head/wrist command and global correction. The right choice depends on whether downstream tasks demand exact finger/contact detail, broad locomotion, or rapid data volume. A common dataset collected with all three interfaces would be unusually valuable because it could separate operator bandwidth, morphology mismatch, and controller-induced style.

## MotionWAM: Towards Foundation World Action Models for Real-Time Humanoid Loco-Manipulation

MotionWAM uses a pretrained video world model without fully decoding future video at every control cycle. A temporal VAE and flow-matching Video DiT process language and egocentric visual dynamics; an intermediate denoising activation at a fixed flow time becomes predictive context for a separate Motion DiT.

Input is a language goal, robot proprioception, and one RGB stream from a head-mounted RealSense D435i. The Motion DiT interleaves self/cross-attention over video features, proprioception, and noisy motion tokens. Per-embodiment input/output projectors wrap a shared trunk so heterogeneous robot data can share the model.

Output is a unified whole-body motion latent decoded by a pretrained controller at 50 Hz. Tokens cover locomotion, torso posture, height, feet, and hands—rather than arm actions plus a coarse base command. PICO demonstrations are recorded at 50 Hz. Training progressively adapts the video prior to egocentric robot dynamics, aligns it with action generation, and then performs multi-embodiment adaptation.

The model is about 2.5B parameters and reports 4.9 Hz inference. Nine G1 tasks include bottle pickup, kicking, cart loading, tossing garbage, retrieval, and basket lifting; success exceeds same-demonstration VLA baselines by more than 30 points. Task-driven feet are important evidence for the unified action-space claim.

MotionWAM conditions on predictive visual features but does not maintain an explicit geometric world state. A single moving head camera loses objects during close manipulation; object exit and viewpoint drift are reported dominant failures. At 4.9 Hz, fast stabilization still belongs to the downstream motion controller.

### Why the unified latent matters

Most humanoid VLAs leave legs inside a velocity tracker, so the semantic policy cannot decide to kick, kneel, brace, or reposition a foot for manipulation. MotionWAM makes every body part part of one generated motion token. The soccer and task-driven foot results are therefore more diagnostic than another hand pickup benchmark: they demonstrate control authority that a base-command hierarchy structurally lacks.

Using intermediate Video-DiT features is a pragmatic world-model choice. The model gains representations shaped by future visual prediction without paying to render pixels. But it is difficult to inspect what physical state those features encode, and action learning may exploit appearance correlations rather than reliable geometry or object permanence.

### Judgment and next work

The over-30-point same-data improvement and nine hardware tasks support the representation, though parameter count and pretraining make attribution difficult. Ablations should compare frozen versus adapted Video DiT, explicit depth/3-D memory, and identical Motion DiT without world features. Multi-camera or wrist-camera inputs would directly address object loss. A persistent object-centric memory, uncertainty-aware replanning, and tactile/contact tokens could convert the present short-horizon visual prior into a stronger world action model. Faster distilled inference would also narrow the gap between 4.9 Hz semantic updates and dynamic contact events.

### A closer look at the world–action coupling

MotionWAM taps an intermediate Video-DiT activation instead of rendering future RGB. Motion DiT attends to that predictive representation, proprioception, and noisy motion tokens. This saves decoding cost and exposes action learning to features shaped by future prediction, but leaves the modeled physical state implicit: object pose, velocity, contact mode, and uncertainty cannot be inspected or constrained directly.

Per-embodiment input/output projectors are what make a shared 2.5B-parameter trunk possible. They map different robot sensors and motion spaces into the common latent. Projector-only adaptation and leave-one-embodiment-out evaluation would test whether the trunk truly contains embodiment-general world-action structure, rather than gains arising mainly from more data and capacity.

At 4.9 Hz, high-level predictions are about 204 ms apart, whereas decoding/control runs at 50 Hz. The pretrained motion controller must reject disturbances inside that gap. The head camera can also lose an object behind the robot's hands or torso exactly during contact. Wrist views, persistent 3-D memory, and tactile tokens address different parts of that problem. Faster distillation, cached features, or asynchronous chunk inpainting would improve responsiveness while retaining the video prior.

### Overall judgment

MotionWAM is one of the clearest attempts to let task semantics recruit feet, torso, and hands rather than placing a VLA above a fixed walking API. Its main scientific uncertainty is attribution: gains may arise from predictive features, unified action space, model scale, diverse pretraining, or their combination. Equal-data and equal-parameter ablations are important. Exposing an explicit object/contact belief and calibrated uncertainty beside the implicit video latent could combine foundation-model flexibility with the diagnosability and safety hooks required for dynamic whole-body deployment.

Relative to π0, MotionWAM is more explicitly humanoid: it learns a unified body-motion latent and delegates stabilization to a pretrained motion controller. Relative to HAF, it emphasizes predictive world features rather than staged generation and online noise-space RL. The strongest hybrid would use MotionWAM's world representation, HAF's structured body-generation order, and a feasibility-aware low-level controller. Such a comparison should equalize demonstrations, backbone scale, cameras, and inference budget; otherwise “world model” benefits are difficult to disentangle from resources.

## OASIS: From Simulation Data Collection to Real-World Humanoid Loco-Manipulation

OASIS moves the expensive part of demonstration collection into simulation. Photos are converted to textured 3-D assets; a VLM proposes dimensions/material classes used to populate physical properties. An operator teleoperates the humanoid in simulation with PICO devices and robot-head video. State trajectories are then replayed offline under randomized texture, illumination, clutter, and camera extrinsics to create many visual observations without recollecting actions.

The high-level model is a Transformer action-chunk policy trained with flow matching. A frozen CLIP encoder processes text; frozen DINOv2 encodes three views (head and two wrists); an MLP encodes two frames of recent reference-motion commands. The denoiser predicts 32 future frames of a 67-D motion representation: trigonometric root roll/pitch, local yaw and translation changes, root height, 29 body joints, and joint increments. Ten Euler denoising steps generate each chunk.

OASIS intentionally uses prior **reference commands**, not measured robot proprioception, as the history input. This avoids feeding tracking noise back into the planner and keeps simulation/deployment conditioning identical, but also hides actual tracking error from high-level feedback. Curriculum rollout training addresses autoregressive model error: after the first clean segment, later segments increasingly reuse predicted history, with rollout probability rising from zero after the first 20% of training to 0.8.

Teleopit converts reference motion to 29 body angles; with 14 hand joints the complete output is 43 DoF. On G1, a D435i head camera and two D405 wrist cameras provide RGB. The planner runs at 25 Hz on an RTX 4090 and the low-level tracker at 50 Hz. The simulation-only planner transfers to real tasks such as cup placement, basket lifting, monitor wiping, and kneeling under a table.

The clever element is decoupling state/action collection from rendering, allowing aggressive visual randomization and instant resets. The central risk is physical mismatch: inferred friction/material and contact geometry may be wrong even when rendered RGB looks convincing. Using commanded rather than measured history improves distribution consistency but prevents direct high-level compensation for accumulating motor error.

### What OASIS scales—and what it does not

The strongest idea is not simply “train in simulation,” but to separate the expensive human decision sequence from the cheap visual observation sequence. Once a teleoperator has produced one successful state/action trace, OASIS can replay that same trace with different textures, lighting, clutter, and camera poses. This is a much better use of operator time than recollecting essentially identical motions to obtain visual diversity. It also gives unusually clean experimental control: visual appearance changes while the intended physical trajectory remains fixed.

That same separation exposes a boundary. Replay can diversify what the robot sees, but it cannot invent a physically different recovery, grasp strategy, or contact transition. If the original trace barely succeeds, randomized rendering produces many views of the same marginal behavior rather than a broader solution distribution. Likewise, a VLM-proposed material class is not a substitute for accurately modeled compliance, mass distribution, joint friction, or contact geometry. OASIS primarily attacks the **visual sim-to-real gap**; it only partially addresses the **dynamics and contact sim-to-real gap**.

The use of reference-command history is an interesting stability choice. It prevents real tracking noise from shifting the high-level model off its training distribution, but makes the planner closer to an open-loop reference generator. Two robots could have received identical commands yet be in meaningfully different physical states after a slip or collision. The 50 Hz tracker must absorb that discrepancy without the command generator being told it exists.

### Judgment and useful next experiments

The real contribution is a scalable data engine plus a deployable hierarchy, not evidence that photorealistic simulation alone solves loco-manipulation. The most informative next ablation would cross visual randomization and physical randomization independently, measuring which one explains transfer for each task. A hybrid history containing both issued commands and compressed measured-state error could preserve training consistency while allowing closed-loop correction. Force/tactile feedback, online object-pose correction, and failure-triggered replanning would be especially valuable during wiping and basket lifting. An active randomization loop could also select render or physics variations that maximize real-world model disagreement instead of sampling them uniformly.

### Deployment lens and evidence to request

OASIS has three distinct generalization boundaries: visual appearance handled by re-rendering, object physics handled by approximate simulation/randomization, and motor execution handled by the tracker. A deployment failure should be attributed to one of these before retraining the entire stack. For example, a visually missed cup suggests representation/randomization failure; a slipping cup with correct localization suggests friction/contact mismatch; a correct command followed by body drift suggests low-level tracking.

The multi-camera setup—head plus two wrists—is an important part of reported observability. Wrist cameras preserve close-up object evidence when the head view is blocked, but their rapid ego-motion complicates correspondence. CLIP language and DINOv2 vision remain frozen, so task learning primarily occurs in the action generator. Per-view dropout and camera-calibration perturbations would reveal whether the model fuses complementary views or over-relies on one.

Success should be reported against the number of unique teleoperated trajectories, not just rendered frames. That distinguishes genuine behavioral scaling from appearance augmentation. Tracking error conditioned on rollout length would quantify the consequence of command-only history. Finally, evaluating the same high-level policy with multiple low-level trackers could show whether OASIS data learn task intent or accidentally encode one controller's dynamics.

## OmniContact: Chaining Meta-Skills via Contact Flow for Generalizable Humanoid Loco-Manipulation

OmniContact proposes **contact flow** as the interface between planning and motor execution. At each control time it supplies sparse body-motion targets plus four binary end-effector contact states at nonuniform future offsets `{0,1,2,3,4,8,12,16,24,32,50}`. This carries near-term contact timing and longer-term intent without transmitting a dense whole-body trajectory.

CF-Track is one AMPPPO policy shared across locomotion, carrying, pushing, kicking, and other meta-skills. Its input concatenates contact flow with five frames of history. Each historical frame includes joint kinematics, base orientation, end-effector positions, previous action, and object-relative 6-D pose/bounding box. The policy outputs low-level motor actions at 50 Hz. Its actor–critic uses a Transformer rather than a plain MLP; the AMP discriminator sees ten frames (410 dimensions) and uses a `[256,256]` MLP. Training normalizes observations, randomizes dynamics, and uses motion/contact targets derived from captured interactions.

CF-Gen is not a monolithic learned motion generator. It assembles phase templates, object-centric anchors, trajectory parameters, IK keyframes, and contact timing into a new flow; a 50 Hz monitor detects execution failure and replans. A VLM may choose objects and meta-skills from scene/language input, but is constrained to object-level planning rather than emitting joint motion.

This division explains the paper's skill chaining: the planner can replace a failed anchor or concatenate flows while the tracker supplies continuity and balance. Tests include box carry, suitcase push, interaction, and longer combinations. The interface is far more execution-aware than symbolic skill names and far cheaper than dense trajectory synthesis.

Binary contact still omits desired wrench, compliance, friction, and detailed finger configuration. CF-Track also uses privileged/externally estimated object pose rather than raw onboard vision in its core formulation. Recovery works when failure detection and template alternatives cover the event; novel contact mechanics remain outside the abstraction.

### Why contact flow is a meaningful interface

OmniContact sits between two common but unsatisfactory extremes. A symbolic planner that says “push the suitcase” leaves timing, stance, and which limb should contact unspecified. A planner that emits every joint angle at every instant is expensive, embodiment-specific, and brittle to execution delay. Contact flow retains the task events that most strongly constrain physical interaction—where contact should occur and how the body should evolve around it—while allowing a reusable tracker to fill in dynamically plausible motion.

The nonuniform prediction times are a particularly good detail: dense near-term samples support accurate contact entry, while sparse long-term samples preserve intent without inflating the command. This resembles model-predictive control in spirit even though CF-Gen is template/anchor based. The single multi-skill CF-Track also means transitions are resolved inside one learned motion manifold instead of by switching between unrelated controllers.

The cost of this abstraction is that “contact” is reduced to a binary state. Pushing a light carton and bracing a heavy door can have identical contact bits but require different normal force, tangential force, impedance, and support polygon. The planner also assumes that object anchors and failure conditions are already observable. Therefore, successful chaining demonstrates the usefulness of the interface on covered skill families, not unrestricted compositional manipulation.

### Next research directions

A natural extension is **contact-wrench flow**: augment contact probability with desired force ranges, surface normals, compliance, and slip risk. Learned CF-Gen could propose flows from scene geometry while a feasibility critic rejects commands outside the tracker's experience. Calibrated uncertainty should determine when to replan rather than relying only on hand-coded failure thresholds. Finally, replacing privileged object pose with onboard RGB-D and tactile state estimation would test whether contact flow remains robust when the planner's anchors themselves are noisy.

### Control semantics and failure diagnosis

The ten future offsets give CF-Track a compact temporal plan extending much farther than its five-frame state history. Near offsets govern the immediate transition into or out of contact; distant offsets communicate the next interaction phase. A Transformer is appropriate because it can relate nonuniform future targets to recent state without assuming a fixed Markov vector. The AMP discriminator adds stylistic plausibility over ten frames, while task rewards make the object/contact outcome matter.

Failure can occur at three layers. CF-Gen may choose a wrong anchor or phase sequence; state estimation may place the object incorrectly; or CF-Track may be unable to realize an otherwise valid flow. The 50 Hz monitor can replan only if its detector distinguishes these cases sufficiently. A generic threshold may repeatedly issue new plans when the true problem is a systematic pose bias or unavailable force.

Evaluation should include unseen *transitions* between known skills, because chaining success on trained adjacency patterns can come from memorized boundaries. It should also compare contact flow with an equal-bandwidth dense waypoint command and with symbolic skill IDs. That would isolate whether the contact representation itself—not the Transformer or replanning—creates robustness and compositionality.

### Comparative position

OmniContact's interface is richer than HANDOFF's wrist/base targets because it represents future contact, but much smaller than HDMI-style dense references. It is therefore attractive when a planner knows *which interaction events* should happen without knowing the entire joint motion. Compared with CEER, it trades continuous compliance-oriented task space for temporal contact structure. A combined interface—root/end-effector goals plus uncertain contact-wrench flow—could serve both free-space reaching and sustained physical interaction without requiring a separate controller family.

## OmniH2O: Universal and Dexterous Human-to-Humanoid Whole-Body Teleoperation and Learning

Source: [arXiv 2406.08858](https://arxiv.org/abs/2406.08858).

OmniH2O uses kinematic pose as a universal control interface: VR, RGB pose estimation, language/frontier models, or an imitation policy all reduce to target human/robot keypoints. A PPO teacher first tracks 14,000 retargeted/augmented AMASS sequences with privileged rigid-body pose, velocity, reference errors, and previous action. The dataset is deliberately biased with standing and squatting variants; otherwise the robot continually shuffles while manipulating.

DAgger distills a deployable student. The teleoperation goal is only three points—head and two hands—with current-to-target position/velocity differences. Proprioception comprises 25 steps of joint positions, joint velocities, root angular velocity, projected gravity, and previous actions. No global linear velocity is supplied; history implicitly estimates it and removes the MoCap dependency of H2O. The policy outputs target joint angles for PD control. Dexterous fingers are handled separately through VR hand pose and IK.

Reward curricula are important: regularizers such as foot height can cause stomping if applied too strongly early, so a maximum-foot-height term is scheduled to distinguish standing from intentional stepping. Simulation success is 94.1%, close to the privileged teacher's 94.77%; history-length ablations show modest accuracy differences but large training/observation cost changes.

The same controller supports VR, third-person RGB shadowing, GPT-4o-generated goals, and autonomous policies learned from OmniH2O-6. That dataset pairs first-person RGB-D, three-point commands, and whole-body motor actions for six tasks.

OmniH2O's innovation is the robust sparse motor interface, not autonomous scene understanding. Three points underdetermine elbows, feet, contact force, and collision clearance, so the motion prior chooses them. Direct finger IK is not whole-hand dynamics control. Any upstream visual/language system must still generate safe and reachable goals.

### Why three points and long history work together

The head-and-hands command is deliberately underdetermined. That is a feature when the motion prior is strong: an operator need not specify pelvis position, footsteps, elbows, or balance reactions, and different input devices can all share the same interface. The 25-step proprioceptive history complements this sparsity by supplying evidence about velocity and recent motor response without a motion-capture root-velocity estimate. In effect, the command says what matters semantically while history lets the policy infer the unobserved dynamic state.

The dataset construction is an equally important, less glamorous contribution. A universal policy trained mainly on walking motion can satisfy hand targets by continually stepping or adopting awkward postures. Adding standing and squatting variants changes the prior over valid solutions. This shows that the learned null-space behavior—the motions not fixed by the three points—is controlled as much by dataset balance as by reward design.

The impressive cross-interface demonstrations should therefore be interpreted carefully. They show that many upstream systems can drive one robust motor layer, not that the motor layer understands objects or tasks. Its prior may choose a visually plausible elbow or foot configuration that collides with an unseen table. Finger IK further separates hand shape from contact dynamics, limiting claims of dexterous whole-body control.

### What should come next

Variable-cardinality commands would make the interface more general: a planner could specify only a hand for reaching, add the head for telepresence, or add feet/pelvis/contact targets for constrained maneuvers. Masked-goal training is one route. A feasibility and uncertainty predictor could warn an upstream VLM before an unreachable three-point request destabilizes the body. Scene-aware collision constraints, tactile hand feedback, and integrated finger policies would close major gaps. It would also be valuable to test whether the same student can preserve quality across robots with different proportions rather than retargeting only at the input layer.

### Training evidence and practical interpretation

The teacher-to-student gap is unusually small in the reported simulation metric, which supports the claim that long proprioceptive history replaces much privileged velocity information. It does not mean every teacher variable is reconstructed accurately; multiple hidden states can yield the same successful action. Estimator probes for base velocity, contact, and phase would reveal what history actually encodes.

The standing/squatting augmentation is a useful warning for dataset builders. A controller can have excellent average tracking yet be unsuitable for manipulation because its learned null space produces unnecessary footsteps. Reporting foot-motion statistics during stationary hand tasks is therefore as important as end-effector error. Likewise, the three-point interface should be tested with adversarially rapid or conflicting commands to measure its reachable command envelope.

OmniH2O is best used as a stable motor “API.” Upstream modules should operate below a certified command rate and receive reachability feedback. Downstream finger IK should be evaluated under object force, since geometrically matching a VR hand does not ensure stable grasp. These interfaces, rather than headline tracking success, determine whether the system scales into dependable long-horizon autonomy.

### Comparative position

OmniH2O and CLONE both exploit sparse head/hand goals and long proprioceptive history. CLONE adds global LiDAR correction and a layer-wise MoE for long-horizon teleoperation; OmniH2O emphasizes a universal upstream interface, large retargeted motion corpus, and several goal sources. HANDOFF uses even fewer wrist variables but explicitly distills locomotion, whole-body, and recovery teachers. Together they show that sparse control works only because a powerful learned prior resolves missing feet, pelvis, and elbow motion. Differences in prior data and feedback matter more than the raw number of command dimensions.

## OmniRetarget: Interaction-Preserving Data Generation for Humanoid Whole-Body Loco-Manipulation and Scene Interaction

OmniRetarget treats embodiment transfer as deformation of an **interaction mesh** joining body, object, and environment keypoints. Laplacian coordinates preserve local spatial relationships while kinematic limits, collision constraints, and contact terms adapt the mesh to a robot. This is more appropriate than independently matching human joints because object/support relations survive changes in limb length and morphology.

The same representation supports augmentation: object pose/scale and terrain geometry can change while attached mesh relationships move relevant body parts consistently. Additional objectives prevent trivial transforms and encourage useful diversity in height, depth, and approach. The output is more than eight hours of reference trajectories for pickup, carrying, pushing, parkour, and scene interaction.

Downstream policies use PPO and a single shared minimalist design. Actor observation is purely proprioceptive: pelvis linear/angular velocity, joint state, projected gravity, and previous action. The critic may receive privileged reference/state information, but no camera, LiDAR, or object pose is required at deployment. Policies output joint-position targets and are trained with only five common rewards plus four robot-domain-randomization terms. Observation noise and physical randomization support zero-shot transfer.

This result is evidence for the paper's real thesis: high-quality interaction-preserving references can reduce downstream reward and observation complexity. But the policy is primarily reference/skill execution, not open-world autonomous manipulation. The kinematic optimizer is expensive and can still produce dynamically marginal motion; contact/friction is not fully determined by geometry. Pure proprioception also means the robot cannot correct an object that deviates substantially from the scripted interaction.

### The important distinction from pose retargeting

Conventional retargeting asks whether the robot resembles the human pose. OmniRetarget asks whether the robot preserves the **relationships that make the interaction work**: hand relative to handle, feet relative to support, body relative to obstacle, and object relative to environment. The interaction mesh is valuable because these relations survive morphology changes better than absolute joint angles. Laplacian coordinates provide a local, structured objective rather than a bag of unrelated point distances.

The downstream minimalist policy is a strong diagnostic experiment. If a simple proprioceptive controller with a small common reward set can execute diverse behaviors, then much of the intelligence has indeed been moved into the reference data. This is preferable to demonstrating transfer only with a highly task-specific policy, where it would be unclear whether retargeting or reward engineering produced the result.

However, preserving geometry does not guarantee preserving physics. A kinematically valid lift may demand excessive torque; a hand-object relation may look correct but use the wrong force or friction cone. Augmentation can also generate configurations that satisfy mesh constraints while lying outside the controller's recoverable distribution. The pure-proprioceptive executor works best when the environment follows the expected script and cannot directly observe object drift.

### Judgment and extensions

OmniRetarget is best viewed as a high-quality **data compiler**, not a complete autonomy stack. The next version should incorporate approximate torque, momentum, support-polygon, and contact-wrench feasibility into optimization, preferably with a learned dynamics surrogate to retain throughput. Each generated trajectory could carry a confidence or difficulty score for curriculum learning. Closing the loop with onboard object perception would reveal how much reference quality survives realistic state-estimation noise. A further test across substantially different humanoid morphologies would establish whether the mesh representation is truly embodiment-general or still needs extensive robot-specific tuning.

### What to measure in a retargeting system

Joint-space similarity to the human is not the decisive metric. More relevant measures are contact-point error, penetration, object-trajectory error, support feasibility, required torque, and downstream policy success. OmniRetarget's strongest evidence is that a simple common controller can use its output, because this integrates many small geometric errors into an execution-level test. Still, controller success can conceal systematic reference bias if PPO learns to ignore difficult parts.

Augmentation quality should therefore be evaluated before and after control. Before control, report mesh distortion, constraint violation, dynamic-feasibility proxies, and optimizer convergence. After control, report success as a function of how far scale, pose, and terrain depart from the seed. This would expose whether “more data” expands useful support or mostly creates near-duplicates.

Interaction meshes also offer an avenue for interpretability. Visualizing which edges carry the highest deformation cost can show whether a grasp is preserved through hand–object structure, foot–ground structure, or global posture. Learning those weights from successful physical rollouts could improve the manually designed objective while preserving the mesh representation's geometric clarity.

### Comparative position

OmniRetarget complements rather than replaces video-based systems. HDMI and HumanX reconstruct a small number of interactions and rely on RL to repair them; GRAIL constrains generated video with known scene assets; OmniRetarget concentrates on transferring and augmenting already recovered interactions across embodiments and environments. Its interaction mesh is especially useful when relative geometry carries the task. It is less decisive when success depends on hidden material properties or precise force. Pairing mesh preservation with a learned dynamics/contact feasibility score would cover both halves.

## SplitAdapter: Load-Aware Humanoid Loco-Manipulation via Factorized Adaptation

SplitAdapter adapts a frozen AMP locomotion/manipulation policy to unknown payload and robot dynamics. A standard history encoder mixes box mass/contact with motor/friction changes in one latent; SplitAdapter begins with a 50-step history of 123-D observations and separates an object/load branch from a robot-dynamics branch.

One actor frame contains 108 proprioceptive values plus a 15-D task block: local box position (3), 6-D rotation, size (3), and goal position (3). The critic adds 3-D base linear velocity. Base actor/critic are `[512,256,256]` MLPs and output 29 joint-position targets for PD.

The adapter applies `Conv1d(123→64,k6,s2)` then `Conv1d(64→32,k4,s2)` with ELU, yielding 320 features. Separate heads produce 16-D object and dynamics latents; a 128-D FiLM generator injects them by soft-split routing. Auxiliary heads predict mass/loaded state, object delta, and a 79-D robot transition. Gradient reversal makes each branch poor at predicting the other's target, while KL/L1 terms regularize the codes.

PPO uses 4,096 environments, randomized 0.5–5 kg boxes, robot parameters, friction, observation noise, and disturbances; tests include unseen heavier loads. Rewards stage approach, lift, carry, placement, release, and stabilization. Factorization improves transfer and interpretability, but payload and robot dynamics are physically coupled. Object/goal poses are externally supplied, and history adapts only after load effects become observable.

### Why the factorization is useful

Carrying a heavy object changes motion for two different reasons: the object has task-relevant state, and its load changes the robot's effective dynamics. A single opaque adaptation vector may encode both but offers no guarantee that the learned representation recombines correctly when a familiar robot carries an unfamiliar object. SplitAdapter explicitly allocates capacity to object/load variables and robot variables, then uses auxiliary prediction and gradient reversal to encourage separation. FiLM injection lets those estimates modulate the frozen policy without relearning its entire locomotion prior.

The 50-frame window is important because neither payload nor friction is directly observable from one configuration. Their effects appear through acceleration, tracking error, and contact response. But identifiability remains imperfect: a weak motor and a heavy box can produce similar histories, and payload changes the robot-object coupled dynamics. Gradient reversal discourages information leakage; it cannot make the underlying physical causes uniquely observable.

The externally supplied box pose and goal also narrow the autonomy claim. The paper studies load-aware control under clean task state, not perception under occlusion. Adaptation is reactive: the robot must experience the new load before its history reveals it, which can be dangerous during the first lift.

### How to strengthen the result

Future evaluation should construct deliberately confounded cases—heavy object/strong motors versus light object/weak motors—to test whether the two latents really recombine. Mutual-information diagnostics and latent interventions would be more convincing than auxiliary prediction accuracy alone. Online uncertainty could cause conservative lifting until mass is identified. Wrist force/torque or tactile sensing would expose load earlier, and an RGB-D object tracker would remove the external-pose assumption. A broader multi-task test could determine whether the split is a reusable principle or mainly tailored to box transport; deformable or sloshing loads would be particularly revealing.

### Adaptation timing and safety implications

A 50-step history gives the encoder substantial temporal evidence, but adaptation necessarily lags the first effect of a new payload. At lift onset the controller acts mainly from the base policy and task state; only after acceleration/tracking error appears can the load latent change. This is a classic dual-control issue: actions both perform the task and reveal dynamics. An explicit probing or cautious preload phase could identify mass before committing to transport.

FiLM modulation is a conservative adaptation mechanism compared with replacing the policy. It scales/shifts internal features while the frozen AMP controller retains its learned motion structure. Auxiliary mass/load and transition predictions encourage useful codes, and gradient reversal discourages cross-contamination. Still, prediction performance should be reported over time from first contact so readers can see how quickly the correct regime is identified.

The unseen heavier-load tests support extrapolation, but safety depends on failure behavior outside training. Does the policy slow down, lower the box, or continue with saturated torque? A confidence-aware adapter should gate commands and expose an overload flag to the planner. Reporting latent trajectories alongside motor tracking error and estimated mass would make this mechanism much more interpretable.

### Comparative position

SplitAdapter addresses a narrower but deeper partial-observability problem than general visual policies: hidden payload and actuator dynamics. Unlike π0/HAF, it leaves task semantics unchanged; unlike MPC-guided training, it estimates online how the plant differs from nominal. Its factorized latent could condition CEER, HANDOFF, or a reference tracker. The idea becomes broadly convincing if the same object/dynamics split transfers across carrying, pushing, tool use, and human contact without redesigned auxiliary targets.

## SUGAR: A Scalable Human-Video-Driven Generalizable Humanoid Loco-Manipulation Learning Framework

SUGAR constructs skills through a reconstructed demonstration, a privileged physical Refiner, and a deployable Command Tracker plus Command Generator. Human/object trajectories are aligned using pose/depth cues and a VLM labels required contact. PPO Refiner rollouts repair physically infeasible video motion rather than requiring task-specific semantic rewards.

Refiner and Tracker use three-layer actor MLPs `[512,256,128]` with asymmetric critics. The deployable Tracker gets five frames each of root angular velocity, 29 joint positions, 29 joint velocities, 29 previous actions, and projected gravity. It also receives current root-frame object position/orientation and commands for joint position, root linear/angular velocity, and contact label. Privileged inputs add 14 body poses, object/root velocity, and five future reference frames.

A 12-block Diffusion Transformer generates commands. Object state, previous command, and optional goal pose are embedded by separate MLPs; the DiT predicts eight future commands, executes four, replans, and interpolates output to the 50 Hz Tracker. This removes the need for dense reference motion at deployment.

Tracking, interaction/contact, and regularization rewards are combined with robot/object randomization and impulses applied during active contact. Six G1 tasks show scaling with more videos, recovery, and goal transfer. The architecture is rich but multi-stage: reconstruction errors can propagate, the Refiner may erase rare strategies, object pose remains an external sensing requirement, and five-frame memory may miss slow hidden load changes.

### What SUGAR contributes beyond video imitation

Raw human video is abundant but not directly executable: depth is uncertain, contacts are ambiguous, human proportions differ, and reconstructed trajectories can violate robot dynamics. SUGAR's key insight is to treat video reconstruction as a proposal and use a privileged RL Refiner to turn it into physically successful interaction data. The Command Generator then compresses dense repaired motion into a receding sequence of targets, while the common Tracker supplies balance and disturbance recovery. This hierarchy makes adding another video cheaper than hand-designing another task reward from scratch.

The progressive state pool is a significant training detail. If refinement always starts from the original reconstructed state, the policy rarely sees the deviations caused by its own corrections. Feeding refined states back into initialization broadens the state distribution and makes repair iterative. Similarly, applying impulses during active contact targets the moments when object motion and robot stability are most tightly coupled.

Still, each stage can hide another stage's error. A Refiner may “repair” an unusual but valid human strategy into the easiest familiar motion, reducing behavioral diversity. The Command Generator may then learn the Refiner's bias, and the Tracker can mask command errors until a difficult contact. Reported scaling with video count is encouraging, but does not isolate whether diversity, reconstruction quality, or total transition count produces the gain.

### Judgment and future work

SUGAR is best understood as a scalable skill-acquisition pipeline, not an end-to-end perceptual policy. Joint or alternating training could allow downstream failures to improve the Refiner and command representation, though it would sacrifice some modularity. A vision-based student should replace externally measured object state, and tactile/force inputs should distinguish geometrically correct contact from mechanically useful contact. Failure-driven video selection could prioritize demonstrations that add new strategies rather than near-duplicates. Longer recurrent state or explicit load estimation would help with slowly revealed object dynamics. Finally, preserving multiple refined solutions per video would let a planner choose between fast, stable, or low-force strategies instead of collapsing each demonstration to one canonical behavior.

### Scaling claims and pipeline accounting

SUGAR's scaling claim should be evaluated against the full conversion cost, not only raw video count. Each clip requires human/object reconstruction, alignment, contact labeling, physical refinement, command extraction, and tracker training. Useful metrics include acceptance rate, Refiner simulation steps per minute of video, manual correction time, and downstream success per processed clip. These reveal whether the pipeline scales linearly or whether difficult videos create a growing curation burden.

The deployable Tracker's five-frame history is tuned for fast local state, while the DiT's eight-command horizon carries higher-level temporal intent. Executing four commands before replanning balances smoothness and correction. Object pose remains an explicit high-level input, so real deployment needs a tracker whose noise/latency should appear during training. Perturbing only robot dynamics does not cover this perception error.

The best evidence for broad generalization would hold total refined transitions fixed and vary the number of source videos, objects, viewpoints, and strategies independently. If diverse videos outperform more frames from one clip, the method has learned reusable interaction structure. If not, the gain may mostly be conventional data scaling. Reporting errors at every pipeline stage would turn SUGAR from a successful recipe into a more predictive framework for choosing new video data.

### Comparative position

SUGAR sits between HDMI's one-video specialist and GRAIL's large generative-video engine. It uses videos at scale, repairs them in physics, and trains a command hierarchy, but still depends on structured object state. Compared with HumanoidMimicGen, it obtains proposals from videos rather than simulator demonstrations and relies more on RL refinement. A shared benchmark measuring processing yield, simulation cost, labor, and real success per source minute would make these pipelines comparable.

## Thor: Towards Human-Level Whole-Body Reactions for Intense Contact-Rich Environments

Thor targets sustained high-force behavior where the waist must transmit ground reaction force between feet and hands. Three PPO actor–critics separately control lower body, waist, and upper body. They share observations, output disjoint joint slices, and train jointly with a total action/energy coordination penalty to prevent one component from dominating.

Actor observation contains 29 joint positions and velocities, base angular velocity, projected gravity, previous 29-D action, planar velocity/yaw commands, locomotion mode, desired hip height, and 14 upper-body reference angles. Critics additionally receive base linear velocity, global quaternion orientation, and a 6-D external wrench. Thus force is privileged during learning but not required by the deployed actors.

Training begins with low-force disturbances to establish gait/posture, then raises force magnitude and randomizes direction. Rewards cover command/pose tracking, support and balance, end-effector force production, torso counter-tilt, and energy. Actions run at 50 Hz with a 500 Hz actuator loop.

G1 tests include heavy doors, racks, wheelchair-like loads, and wiping; reported peak pulls reach roughly 168 N backward and 146 N forward. Waist specialization is a useful inductive bias, but disjoint agents can conflict and coordinate only through shared return/penalties. Thor assumes supplied motion/force task context and does not perceive object geometry. Performance is learned from contact simulation rather than protected by a formal force or balance guarantee.

### Why the waist deserves its own controller

In high-force loco-manipulation, the waist is neither merely part of locomotion nor merely part of arm tracking. It transmits ground reaction forces from the legs into the torso and hands, changes the moment arm against an external load, and counter-rotates to keep the center of mass supportable. A conventional upper/lower split risks assigning these responsibilities inconsistently. Thor's three-way decomposition encodes this mechanical role directly and gives the waist policy a focused optimization problem.

The curriculum is also physically sensible. Learning intense interaction before reliable stance and gait encourages unstable shortcuts; gradually increasing disturbance lets the policy first establish a viable support strategy. The peak-force demonstrations make the work stand out from manipulation papers where the hands make contact but transmit little sustained load.

The decomposition does not automatically produce cooperation. All three policies observe related state, but each controls only its joint subset and cannot explicitly negotiate a desired wrench or center-of-pressure plan. Shared rewards and energy penalties encourage coordination indirectly. This can work in training distributions while still producing agent conflict under a novel force direction. Privileged critic wrench input improves learning but does not tell the deployed actor whether a real contact is slipping or unexpectedly compliant.

### Assessment and extensions

Thor convincingly argues that forceful tasks deserve architecture and training different from gentle pose tracking. It does not yet establish safety near hardware limits or humans. A higher-level wrench/contact command shared by all three policies could make coordination explicit; online wrist/foot force sensors would reveal whether that command is achieved. Constrained RL or a control-barrier safety layer should enforce torque, friction-cone, support, and fall-risk limits. Scene geometry and object-state perception are needed to select stance and hand placement autonomously. Comparing the three-agent design with an equal-capacity monolithic policy and a centralized policy with structured auxiliary losses would clarify whether specialization itself—not parameter count or curriculum—causes the gain.

### What the measurements should reveal

Peak pull force is useful, but it conflates strength, stance, friction, and transient impulse. Sustained force over time, impulse, center-of-pressure margin, foot slip, electrical/thermal load, and success after force-direction change provide a fuller picture. Force normalized by robot mass would also make comparison across platforms meaningful. The actor lacks runtime wrench input, so tests with unexpected compliance or a suddenly released load are particularly important.

Thor's actor observation mixes commanded locomotion, desired height, mode, and a 14-angle upper-body reference. This means task intent is still externally specified; the learned contribution is force-capable coordination. The critic's privileged wrench can shape learning but cannot resolve partial observability at deployment. Histories or force sensors may be necessary when identical joint configurations experience different external loads.

A useful diagnostic is to record how much mechanical work each module contributes and whether one compensates for another's saturation. If the waist policy carries most improvement, its representation may be reusable inside a centralized controller. If gains emerge only through coordinated specialization, that supports the multi-agent design more strongly.

### Comparative position

Thor differs from most reference-tracking work because the primary outcome is sustained external force rather than geometric imitation. Compared with CEER's learned compliance, Thor optimizes the opposite regime: deliberately transmitting large force while remaining stable. Compared with SplitAdapter, it trains robustness across disturbances but does not explicitly infer payload/dynamics. Combining force-aware specialization with online dynamics adaptation and explicit wrench commands could support tasks ranging from gentle human contact to heavy pushing within one controller family.

## VAIC: Vision-Guided Humanoid Agile Object Interaction Control via Decoupled Commands

VAIC learns agile object interaction without motion capture at runtime. Stage 1 trains a privileged PPO teacher using exact object pose/velocity, future reference/contact, clean robot state, and a canonical object point-cloud template; depth is structurally present but zeroed. Stage 2 activates vision, inherits teacher weights, and lets the student explore with PPO while distilling privileged behavior.

Deployable proprioception contains five frames of joint positions, velocities, base angular velocity, and projected gravity plus three prior actions. Commands are a quantized 3-D velocity target and binary interaction-phase flag. Exteroception is a temporal stream of 64×36 depth from a torso-mounted RealSense D435i pitched downward 48°, captured at 60 Hz, plus the canonical object template.

A CNN encodes depth and a GRU fuses visual tokens, proprioceptive history, velocity, and phase to infer a latent object/environment state. This recurrent Object Adaptation module conditions the student actor, carrying information through temporary self-occlusion. Camera-pose jitter, ray resampling, missing pixels, and depth corruption are randomized.

The policy runs at 50 Hz and joint control at 200 Hz. Demonstrations include carrying, pushing/pulling, skateboarding, kicks, strikes, throws, and catches. The achievement is onboard closed-loop interaction rather than external tracking.

VAIC still needs a known object template and a phase command from an operator/planner. Low-resolution depth struggles with fast, reflective, transparent, or never-visible objects. Recurrence provides a learned belief, not calibrated uncertainty, and extreme unseen dynamics can invalidate its history.

### Why the command and perception split works

The velocity command specifies *how the object should move*, while the binary phase tells the controller whether it should interact or move freely. This is substantially more reusable than a full reference trajectory: the same motor policy can push, carry, strike, or release at different speeds without replaying one captured motion. It is also more informative than a base velocity alone because the phase resolves whether visual proximity should lead to avoidance, approach, or forceful contact.

The teacher–student schedule isolates two hard problems. The privileged teacher first discovers a dynamic whole-body strategy with clean object state. The vision student then learns to infer the hidden interaction state while continuing PPO exploration, rather than being limited to behavioral cloning on teacher observations. The CNN–GRU architecture is appropriate for intermittent depth: temporal memory can preserve object direction and contact phase during short self-occlusions. Camera corruption and pose jitter make that belief less dependent on ideal simulation pixels.

The canonical object template is both a strength and a limitation. It supplies shape context that sparse depth may not reveal, but presumes object identity and geometry are known. A single binary phase compresses rich contact progression—approach, preload, stick, slip, release—into two modes. Quantized velocity commands may also hide strategy differences at transitions.

### Critical judgment and next work

VAIC is one of the stronger closed-loop perception results in this collection because onboard depth directly affects agile whole-body interaction. Its main open problem is trustworthy state estimation. Automatic phase inference, calibrated belief uncertainty, and a recovery state triggered by visual/contact disagreement would reduce dependence on a perfect upstream planner. RGB, event cameras, and tactile/force sensing could complement low-resolution depth, especially for transparent objects and impact timing. A learned object encoder should replace the fixed template and generalize across shape families. Longer transformer or state-space memory could model slow payload effects, but should be compared carefully with the GRU to show that added history—not merely capacity—improves dynamics adaptation.

### How to evaluate the learned belief

Because the GRU's latent is not directly supervised as object pose, task success alone cannot show whether it tracks geometry, dynamics, phase, or a memorized response. Linear probes for object position/velocity and controlled latent perturbations could clarify what information is retained during occlusion. Plotting failure probability against occlusion duration would measure memory rather than visual quality.

The teacher has exact object/reference/contact state, while the student receives corrupted depth and history. Joint PPO plus distillation lets the student deviate when its observation demands a different robust action, which is stronger than pure imitation. But teacher confidence should be reduced when privileged behavior depends on information fundamentally absent from the camera. Otherwise distillation can penalize the best uncertainty-aware response.

Velocity quantization and the binary phase are convenient planner APIs. Their resolution should be ablated: coarse bins may stabilize training while creating jerky transitions, and a single phase bit may be insufficient for pre-impact versus sustained contact. Reporting behavior across command boundaries and unseen intermediate speeds would indicate whether the controller interpolates physically or just switches among trained modes.

### Comparative position

VAIC is distinguished by putting onboard depth inside the agile controller rather than assuming mocap/object pose or staying blind. HumanX offers faster prior-driven interactions with optional external tracking; HDMI follows explicit object/contact references; VAIC learns a recurrent visual belief and accepts compact velocity/phase commands. It is preferable when closed-loop object response matters and a depth-visible template is available. Extending its belief to RGB and tactile sensing would broaden the object classes while retaining its strong perception–control coupling.

## Weave: Learning Whole-Body Dexterous Loco-Manipulation from Human-Object Interactions

Source: [arXiv 2609.16683](https://arxiv.org/abs/2609.16683).

Weave converts captured human–object interaction into robot/object references through contact-aware retargeting and approach-motion completion. One policy controls 29 body and 12 finger joints across objects and clips rather than reducing grasp to an open/close primitive.

At 50 Hz the asymmetric PPO actor receives base angular velocity, projected gravity, joint positions/velocities, previous action, root-frame object pose, fingertip-to-surface vectors, binary contact flags, and a BPS-SDF geometry descriptor. A short future reference includes robot joint/pelvis motion, object pose, and contact labels. The critic adds base linear velocity and tracked-body pose. Actions are PD position targets; underactuated Inspire distal/intermediate finger joints follow fixed mimic couplings.

Rewards track pelvis/body/object pose and shape grasp geometry. An opposition term places thumb and fingers on different object sides; contact matching compares labels with normalized measured force. Foot sliding, action variation, and limits are penalized. Reference-state initialization, early termination, friction/restitution/CoM/finger randomization, and SimBaV2 networks support training; Muon optimizes matrix weights and AdamW other parameters.

Across nine objects, success is 92.5% on trained interactions and 65% on unseen sequences. The released ~9,000 physical rollouts (~23 hours) include contact annotations. The remaining gap is perception: the policy assumes object pose/contact/geometry features rather than raw onboard sensing. Finger coupling limits dexterity, and autonomous task planning is outside the reference-conditioned system.

### The standout contribution

Weave treats the object, body, and fingers as one coupled control problem. Many “whole-body manipulation” systems stop at arm targets or binary grippers; many dexterous-hand systems assume a fixed arm and base. Here, locomotion changes the reachable grasp, finger contacts change object motion, and object motion changes balance. A single policy over 41 body/hand joints allows these dependencies to be learned rather than mediated through independently tuned controllers.

The BPS-SDF descriptor and fingertip-to-surface vectors are important for cross-object transfer. They express local geometry in a representation more reusable than object identity, while contact-aware retargeting supplies which regions should meet. The opposition reward captures a basic physical property of stable grasping without prescribing exact finger angles. The drop from 92.5% on trained interactions to 65% on unseen sequences is therefore informative: meaningful generalization exists, but the reference/contact distribution still matters greatly.

There is a gap between the controller's input and a real autonomous sensing stack. Accurate object pose, signed-distance geometry, contact labels, and short future references are powerful structured supervision. Errors in any of them can place fingertips on the wrong surface or destabilize the body. Fixed mimic coupling also means the nominal 12 finger outputs do not correspond to fully independent distal control, limiting adaptation around irregular objects.

### Where this should go next

The clearest next step is a perception student that infers object pose, BPS-SDF/contact features, and confidence from head/wrist RGB-D plus tactile sensing. Training with perturbed or delayed estimates would expose whether the motor policy is robust to realistic perception. Adaptive hand synergy or independently actuated hands would test how much performance is capped by hardware coupling. A foundation geometry encoder could allow previously unseen household objects without precomputed exact meshes. Finally, the released physical rollouts could train a residual dynamics/contact model or offline critic, using real failures to improve policy selection rather than serving only as evaluation data.

### Generalization and dataset interpretation

The difference between trained interactions and unseen sequences is a useful reminder that “unseen object” and “unseen motion” are different axes. Geometry descriptors may transfer shape, while contact order and whole-body timing remain close to reference data. A factorial benchmark should separately hold out object shape, scale, mass, friction, grasp region, motion sequence, and their combinations.

Future-reference input gives the controller privileged intention. This is appropriate for tracking but means the policy is not independently deciding how to grasp. The core scientific result is motor generalization conditioned on a plan. A planner or VLA still has to generate geometrically and dynamically valid future references, and its errors may not resemble the clean held-out references used in evaluation.

The physical rollout dataset is especially valuable if it includes failed attempts, synchronized contacts, perception, and motor state. Failure diversity can support offline value learning and robustness analysis; success-only release would be less informative. Contact labels should include sensor thresholds and uncertainty so later users do not treat them as exact ground truth.

### Comparative position

Weave is closest to OmniContact in treating contact structure as central, but it resolves that structure through detailed body–finger–object tracking rather than a sparse future-flow command. It is closer to dexterous imitation than a semantic planner. Compared with GRAIL's binary hand primitives, Weave models much richer grasp geometry, though hardware coupling still restricts independent fingers. A natural stack would let a planner propose contact flow, a geometry module refine fingertip contacts, and Weave's policy execute the resulting whole-body reference.

## WT-UMI: Tactile-based Whole-Body Manipulation via Force-Supervised Contact-Aware Planning

WT-UMI makes tactile sensing part of both demonstration and learned planning. Sensors on two hand/contact sites and chest record tactile images and calibrated normal force during teleoperation or portable human demonstrations. A correction model uses force to convert human-derived target poses into robot-executable labels, while TactAlign reduces the human–robot tactile embodiment gap.

The planner runs at 50 Hz with 400 ms observation and prediction horizons—20 past and 20 future steps. Each frame contains left/right/chest tactile images and two 9-D hand poses (translation plus continuous 6-D rotation). A ViT encodes channel-stacked tactile images; a two-layer MLP encodes poses. Concatenation forms 320-D observation tokens plus separate pose tokens.

A Transformer flow/diffusion denoiser predicts a `20×18` bimanual pose chunk. A separate two-layer, four-head Transformer decoder uses 20 learned future queries and cross-attention to the history to regress two-hand normal-force trajectories. Softplus enforces nonnegative force; SmoothL1 controls contact-onset spikes, and force gradients jointly train the shared encoder.

The low level splits the body. A pretrained RL policy controls 12 legs and three waist joints from pelvis velocity. A 200 Hz tactile admittance controller corrects palm SE(3) targets from measured-versus-predicted force/contact centroid; optimization IK produces 14 arm commands, followed by PD and gravity compensation. Tactile at 100 Hz and proprioception at 500 Hz are synchronized to 50 Hz; generative inference uses RTC.

Tasks include yoga balls, pillows, loaded buckets, and human–robot beam/table transport. Predicting force supplies anticipatory contact intent that position-only imitation lacks. Limitations are calibration drift, sensor wear/coverage, mainly normal-force supervision, and no vision for global localization or obstacles. Current tactile coverage also omits legs/back and fine distributed fingers.

### Why future-force prediction is the key idea

Tactile feedback alone is reactive: the controller learns that force is wrong only after contact has already become too weak or excessive. WT-UMI predicts a future force trajectory from recent tactile and pose history, giving the admittance controller an anticipatory reference. This matters for large compliant objects and cooperative transport, where stable behavior depends on maintaining load over time rather than merely reaching a geometric hand pose.

The architecture cleanly separates responsibilities. The generative planner models multimodal bimanual pose futures; the force decoder shares its history representation but makes an explicit supervised prediction; the high-rate admittance loop corrects contact faster than generative inference; and the pretrained leg policy maintains locomotion from pelvis commands. RTC prevents asynchronous chunk updates from introducing pose discontinuities. This is a thoughtful systems design because each loop operates at the rate appropriate to its signal.

Force supervision also makes the latent more interpretable, but the present signal is incomplete. Normal force does not reveal tangential shear, torsional moment, incipient slip, or the full spatial pressure distribution. A scalar contact centroid can miss two very different pressure patterns with the same total load. The absence of vision means the system assumes pose/tactile history is sufficient once the task is initialized and cannot reason about remote obstacles or object identity.

### Judgment and future directions

WT-UMI's highlight is turning tactile sensing from a late corrective cue into a planned variable. The strongest follow-up would jointly predict spatial force maps, shear, and slip probability, with uncertainty propagated to the admittance gains. Visual–tactile fusion could handle global approach and occlusion while tactile dominates after contact. Variable-compliance and sensor-aging tests are needed to measure calibration robustness; online recalibration or self-supervised force consistency could reduce maintenance. Wider tactile coverage on fingers, forearms, chest, and possibly shoulders would support richer carrying modes. Finally, cooperative tasks should evaluate human comfort and peak interaction force, not only completion, because accurate task execution can still be unsafe or unpleasant for a partner.

### Multi-rate execution and safety interpretation

The 20-step history covers 400 ms, long enough to observe contact loading trends while remaining responsive. The predicted 20-step pose and force horizons cover the next 400 ms. Tactile sensing at 100 Hz and proprioception at 500 Hz are synchronized down to the 50 Hz planner; the 200 Hz admittance/IK layer reacts between generative updates. This hierarchy avoids asking a transformer to close the fastest force loop.

Its safety depends on the interface between predicted force and admittance. Overpredicted force may make the robot press harder; underprediction may cause slip or dropped load. Softplus guarantees nonnegative force but not a safe upper bound. Confidence-aware clipping, passivity constraints, and a reflex based on raw high-rate tactile signals would protect against generative error.

The portable human-to-robot tactile alignment is also a likely source of distribution shift: skin compliance, contact patch, sensor mounting, and object curvature differ. Evaluation should report force calibration error before and after TactAlign, performance under sensor replacement, and degradation when one tactile stream is missing. Those tests would establish whether force supervision is a robust control signal or a laboratory-specific calibration achievement.

### Comparative position

WT-UMI complements vision-heavy systems by observing the variable they often infer poorly: physical contact. Unlike Thor, which produces large force without runtime wrench observation, WT-UMI explicitly predicts and regulates a future force profile. Unlike Weave, it does not require precise object geometry but needs tactile coverage and task initialization. The most capable future system would use vision for global scene and approach, tactile prediction for sustained interaction, and a force-aware whole-body controller for balance—switching emphasis continuously rather than treating modalities as separate phases.


## I-BFM: Reward-Conditioned Robust Humanoid Interaction via Unsupervised Reinforcement Learning

I-BFM extends behavioral foundation models from body-only motion to coupled humanoid–object interaction. Instead of tracking one prescribed human–object trajectory or switching among task-specific policies, it learns one reward-addressable interaction space for carrying, pushing, kicking, getting up, goal reaching, motion tracking, style control, and multi-stage chaining. The central practical result is closed-loop recovery: the controller can abandon a failed nominal motion, regain balance or re-approach a displaced object, and continue the objective.

### State, observation history, action, and RL architecture

The privileged interaction state contains humanoid state, object pose and velocity, object-to-goal displacement and presence, plus bilateral hand-contact indicators, hand positions, and hand-to-object displacement vectors. The deployed actor receives a finite history (h_t=(o_{t-H:t},a_{t-H:t-1})) of proprioceptive observations, available object/contact measurements, and previous actions; the critic may see full simulation state during training. The paper does not use an onboard camera, LiDAR, VLM, or language tokens as the policy interface. Hardware operates in a motion-capture workspace, so object-state availability is an important deployment assumption. The output is joint-level targets executed by low-level PD control at 50 Hz on Unitree G1.

Like BFM-Zero, I-BFM uses off-policy unsupervised reinforcement learning with forward–backward (FB) representations. The forward encoder models discounted future occupancy conditioned on state, action, and latent command; the backward encoder embeds reached states. Their factorization creates a Q-function for the reward induced by a latent. A new downstream reward is converted into a command by averaging backward state features weighted by reward and projecting to the latent sphere. The same actor then runs closed loop with no task-specific policy optimization. Because object and contact variables are part of the represented state, identical body poses can demand different actions when the box has shifted or contact has broken.

Pretraining combines the interaction-aware FB loss with a style discriminator and auxiliary stabilization/safety objectives, jointly using object-interaction and ordinary locomotion/motion data. Carry, push, and kick data deliberately share one task ID, discouraging the policy from memorizing separate task labels. Motion tracking is another interface: backward features from reference interaction trajectories supply latent commands, while the policy retains freedom to recover rather than rigidly reproduce every pose.

### LOGO and temporal structure

The novel Local–Goal Objective Geometry Operator (LOGO) resolves a temporal ambiguity in latent conditioning. A baseline averages backward features across an eight-step future window, which can merge states requiring different immediate actions. LOGO instead derives a local target from the next state and a goal target from the state eight steps ahead. Both are mapped to tangent vectors on the spherical latent manifold: direction indicates how behavior should change, while magnitude represents geodesic distance. A shared actor is trained with both local and goal intents using a 4:3 loss ratio and a small residual weight, preserving immediate contact feasibility without losing long-horizon progress.

This is not a transformer, diffusion model, or planner that rolls out explicit trajectories. It is a latent-conditioned RL actor with successor-state representations. Long-horizon task chaining is performed by changing reward specifications when subgoals finish; the motor policy remains the same. That distinction explains the low reaction latency and ability to deviate from a reference, but it also means an external task manager still decides when “push,” “carry,” or “place” is complete.

### Results, judgment, and future work

In simulated Carry, I-BFM reports 94.3% nominal success and 89.3% after a force-induced robot fall, compared with 1.3% for the cited planning baseline under that perturbation. Removing LOGO lowers success to 51.0%, a much larger change than its small effect on joint tracking error, supporting the claim that multi-timescale intent—not better imitation—is responsible. Hardware demonstrations use a 0.7 kg, 0.35 m cube and show retry after failed grasp/handling, recovery after robot or object disturbance, pushing, kicking, and push–carry–place chains without task-specific retraining or online replanning.

The highlight is representing the physical consequence of whole-body action, not only the humanoid pose. That makes I-BFM closer to an interaction dynamics model while retaining direct reactive control. However, “zero-shot task” still depends on manually specified rewards, measurable object/contact state, training support, and external phase logic. One box geometry, motion-capture perception, qualitative hardware trials, and one-day-old preprint evidence are not yet enough to establish broad object or environment generalization. Comparisons also differ in native inference and training data, so matched-data baselines are needed.

Next work should replace motion capture with egocentric RGB-D and tactile/contact estimation, report the exact observation history and network sizes, and measure latency from sensing through PD execution. Tests should vary object geometry, mass, friction, grasp type, clutter, and contact loss, with held-out interaction families rather than new target positions alone. A coverage/uncertainty estimator should reject rewards outside the learned occupancy, while a safety shield limits latent commands and contact forces. Automatic semantic reward generation from language or vision would make the prompt interface useful to higher-level agents, but only if grounded rewards cannot exploit unobserved state or unsafe shortcuts.
