# World Action Model — Paper Notes

This folder contains models that explicitly predict future environment or robot state together with actions, or learn action-conditioned dynamics that directly train or guide a controller. Behavioral latent policies without a predictive world/state model and semantic world models that only choose navigation goals are excluded. The notes preserve each paper's original task context while emphasizing what is predicted, how action enters the model, the control interface, rollout horizon, uncertainty, and model-bias limitations.

## BeyondMimic: From Motion Tracking to Versatile Humanoid Control via Guided Diffusion

BeyondMimic treats the motion tracker as a broad motor prior rather than a fixed reference follower. A diffusion-based guidance mechanism proposes/regularizes motion targets while an RL controller makes them physically executable and robust. This supports unseen motions, command variation, and recovery beyond strict clip playback. The generative layer expands versatility but adds sampling/conditioning complexity; behavior can leave the reference distribution, and physics plausibility still depends on the tracker and simulation transfer.

### Two stages with different responsibilities

BeyondMimic first learns a highly general reference tracker and then fits a latent state–action diffusion model to its physically grounded rollouts. The tracker is deliberately simplified: rather than accumulating many motion-specific rewards and privileged corrections, it uses a compact reference representation and a small common tracking objective across the corpus. PPO and an asymmetric actor–critic supply feedback robustness; joint-position targets are realized through PD control. The broad tracker is therefore the data generator that converts heterogeneous kinematic motion into dynamically valid robot experience.

The diffusion stage models *joint sequences of future state and action*, not pose alone. This is critical for control: action variables retain how the tracker realizes a motion, while predicted future state permits costs on where the robot will go. A latent autoencoder reduces trajectory dimension and a conditional denoising network operates in the latent sequence space. Recent state/history initializes receding-horizon prediction; at each update, only the near-term action is executed and the horizon is replanned.

### Guidance as a control interface

Classifier guidance differentiates user-defined objectives through the denoising process at inference. The same frozen model can be biased toward desired velocity, spatial keyframes, obstacle clearance, upper-body targets, or combinations. Inpainting fixes known portions of a trajectory and lets diffusion fill the rest. Demonstrations include smooth walk–run control from velocity, cartwheel keyframes injected while walking, and collision mitigation, without retraining a task policy for each interface.

This differs from “diffusion generates a reference, RL tracks it” in an important way: state and action are optimized together inside a short predictive horizon. It resembles model-predictive control with a learned trajectory prior, while the denoising distribution prevents arbitrary optimizer solutions. Ground-reaction-force comparison also probes whether humanlike appearance corresponds to physical contact, although robot morphology—notably missing toe articulation—still creates systematic differences.

### Assessment

The reusable insight is that a high-quality tracker can be distilled into an editable behavior distribution. Guidance gives more flexibility than a fixed command-conditioned actor and avoids new RL runs for modest objectives. However, online denoising costs more and is less deterministic than one MLP pass. Strong guidance can pull samples off the learned manifold, and a short horizon may satisfy a keyframe by creating later instability. The diffusion model inherits all tracker/data biases; it does not independently verify torque, collision, or balance.

Future work should report wall-clock control latency and missed deadlines, add feasibility critics or projection during every denoising step, and feed measured execution error back to the sampler. Longer hierarchical plans could set intent while a fast latent diffuser repairs local motion. Hardware evaluation should separate guidance satisfaction, tracker error, energy/contact load, and recovery after deliberately infeasible goals.


### Minimal tracker interface and implications for compliance

The base tracker uses no stacked observation. Its robot-centric vector contains a reference-motion phase cue, a 9-D anchor pose error, six-dimensional IMU twist, joint position relative to default, joint velocity, and previous action. The phase/reference joints indicate progress but are not directly copied. Output `a` is scaled per joint and added to the default configuration before low-level PD; it is intentionally treated as a torque-shaping intermediate rather than a precisely enforced kinematic plan. This allows lower impedance than trackers that overpower natural dynamics with stiff servo gains.

The task reward averages position, rotation, linear-velocity, and angular-velocity errors uniformly over target bodies. Only joint-limit, action-rate, and self-contact penalties are added. Randomization is similarly compact—contact material, joint zero offsets, torso COM, and velocity perturbations—because excessive randomization made motion conservative. Adaptive segment sampling raises probability where empirical failure is high and relaxes toward uniform once mastered. These choices explain why the generated rollout distribution is smooth enough for a VAE/diffusion prior.

For guidance, the VAE models short state–action segments and the diffusion model predicts a state/latent horizon. A differentiable new cost acts on predicted future state; denoising gradients modify the latent, whose decoder supplies corresponding joint actions. The limitation is model-mediated causality: if the latent decoder's state–action relationship is wrong under hardware contact, guidance can optimize a fictitious future. Incorporating online residual dynamics or rejecting gradients outside training-density support would make this learned MPC analogy more trustworthy.

### Control-stack accounting and a stronger evaluation protocol

For BeyondMimic: From Motion Tracking to Versatile Humanoid Control via Guided Diffusion, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

## GigaBrain-WBC-0.5: A Behavior World Model for Robust Humanoid Whole-Body Tracking with Environment Interaction

Source: [arXiv 2608.18234](https://arxiv.org/abs/2608.18234).

A causal transformer jointly predicts next action, next proprioceptive state, and a mixture distribution over feasible next latent commands. Automatic motion-to-terrain annotation recovers contact geometry at corpus scale. At runtime, commands outside a Mahalanobis feasibility region are retracted toward learned behavior modes, enabling best-effort execution, contact interaction, and unified fall recovery. Reported success includes 81.3% terrain interaction and 99.3% recovery. Its learned feasibility distribution is not a safety certificate and reconstructed geometry covers mainly observed contacts/static supports.

### Behavior World Model and command retraction

Training augments retargeted motion with reconstructed 3-D support/contact geometry, so the causal Transformer observes proprioceptive/action history, latent behavior commands and environment-conditioned transitions. Shared sequence features feed three heads: next joint action, next proprioceptive state, and a mixture distribution over feasible next command latents. Joint prediction forces the controller to model consequences rather than only imitate an action label.

At deployment, the requested latent is scored under the predicted mixture. A Mahalanobis-distance test detects commands inconsistent with current body/environment state and retracts them toward a likely behavior mode. This lets the robot respond “best effort” to a missing support or impossible request instead of blindly tracking. The same state-conditioned model covers terrain contact, disturbance and recovery; reported success is 81.3% for terrain interaction, 83.1% for implausible commands and 99.3% recovery, with limited G1-to-L01 fine-tuning.

The feasibility distribution is empirical and can be confidently wrong outside annotated geometry. Retraction also changes user intent without necessarily explaining how. Future work should calibrate likelihood against physical failure, expose modified commands, add vision/geometry uncertainty and hard actuator/collision constraints, and compare mixture retraction with constrained predictive control.


### Joint prediction as an environment-conditioned consistency test

A conventional tracker only learns `observation, command -> action`; it can assign a plausible action to a command even if the requested next behavior has no support in the current contact geometry. GigaBrain adds next-state and next-command-distribution prediction to the same causal representation. Action quality is therefore constrained by whether the hidden state can also explain the resulting proprioceptive transition and what behavior commands tend to remain feasible afterward. The mixture distribution preserves several alternatives—for example step up, step around, or recover—instead of averaging them into one latent.

Automatic terrain annotation reconstructs support geometry implied by retargeted motion and places it in simulation, converting large flat-ground corpora into paired behavior/contact transitions. This is scalable, but reconstructed support is causally ambiguous: a raised foot does not uniquely specify stair height, a hand pose does not prove load-bearing contact, and missing geometry may reflect motion-capture conventions. Geometry provenance and contact confidence should accompany each training transition.

At runtime the Mahalanobis threshold and mixture covariance determine how readily user commands are declared implausible. Too tight a threshold suppresses novelty; too loose leaves the original unsafe behavior. Calibration curves should plot command likelihood against failure, task deviation after retraction, and false-rejection rate for valid unseen behaviors. Because retraction occurs in latent space, the system should decode and expose the altered short-horizon intent to the operator, especially during teleoperation or object contact.

### Foundation-model criteria, latent coverage, and deployment accounting

For GigaBrain-WBC-0.5: A Behavior World Model for Robust Humanoid Whole-Body Tracking with Environment Interaction, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

A behavioral foundation model should document the source motion hours and filtering, how kinematic observations become physical robot rollouts, replay or on-policy transition count, network and latent dimensions, temporal horizon/history, privileged training inputs, deployable inputs, action representation, control rate, and total simulation compute. Model size alone is not scale when larger policies receive more environment interaction. Data and compute-normalized curves should vary capacity, behavioral diversity and optimization separately, with several seeds and per-family results.

Latent quality requires closed-loop tests. Reconstruction or action likelihood can be high while decoded contacts drift and the robot falls. Report rollout survival, pose and velocity error, contact timing, diversity, latent utilization or collapse, interpolation behavior, prompt sensitivity, inference latency and hardware reliability. New goals, demonstrations or rewards should be structurally held out, not merely different clips from the same corpus. Compare frozen latent control, weight fine-tuning, task-specific RL, retrieval and random/optimized latent search at equal downstream interactions. A coverage estimator should predict when a requested behavior lies outside the pretrained occupancy.

The practical control stack also needs an authority boundary. A high-level prompt or planner selects latent behavior; the decoder produces joint targets; servo and safety layers constrain execution. Prompt computation, denoising or optimization steps count toward latency, and stochastic samples need temporal consistency. “Zero shot” must state what remains optimized after prompting. My judgment is that foundation status is earned by reusable improvement across different command interfaces and tasks with predictable failure, not by one very broad tracker. Cross-embodiment transfer, uncertainty-based abstention, contact/terrain perception and shielded latent planning are the decisive next tests.

## Hybrid Internal Model Learning Agile Legged Locomotion with Simulated Robot Response

### From privileged environment inference to response prediction

Hybrid Internal Model (HIM) is a proprioception-only locomotion method intended to generalize across robot platforms and disturbances without a camera, LiDAR, explicit terrain estimate, or teacher–student privileged-state transfer. Its insight is that a deployable controller need not identify friction, payload, terrain height, and actuator parameters separately. What matters is predicting how the robot will respond. A latent inferred from recent observation/action history is therefore trained to contain both an estimate of the next proprioceptive state and a contrastive code for underlying dynamics.

### Network, inputs, and outputs

At every step, the policy receives the usual command and proprioceptive state—base angular velocity and gravity direction, commanded velocity, joint positions and velocities, and previous action—plus the hybrid internal embedding extracted from a history window. The encoder is a three-layer MLP with widths 512, 256, and 128. A source encoder processes simulator response targets; contrastive learning brings the history embedding near the correct response embedding and separates it from other batch samples. A prediction head simultaneously reconstructs the next observation. The locomotion policy concatenates the latent with current observations and outputs one residual target per actuator, scaled and added to a nominal pose; a PD loop produces torque.

Training alternates Hybrid Internal Optimization and PPO rather than using a fixed pretrained estimator. HIO updates the latent model from the newest rollouts, and PPO updates actor/critic while holding the internal model fixed. This prevents the control loss alone from driving the latent toward an arbitrary shortcut. It also avoids an oracle teacher, extra distillation stage, and explicit parameter labels. The reported training budget is about 200 million simulation frames; policies execute at 50 Hz with a 500 Hz PD controller.

Rewards largely follow robust command-tracking locomotion practice: linear/angular velocity tracking, uprightness, height and gait/contact terms, plus penalties on torque, joint acceleration, action changes, and collisions. Terrain and dynamics curricula expose slopes, stairs, rough ground, payload changes, pushes, friction, motor-strength variation, and delay.

### Achievement and interpretation

Experiments span quadruped and humanoid embodiments and show robustness to disturbance and varied terrain using only internal sensing. The deeper contribution is representational: next-state regression captures short-term mechanics, while batch contrastive learning encourages a stable domain-level signature. Together they outperform using either a plain history encoder or direct regression alone in the paper’s ablations.

### Limits and useful follow-up

HIM is still blind. History reveals a surface only after interaction, so it cannot plan a gap, isolated stone, or head-height obstacle before contact. The contrastive negatives are batch-dependent and do not guarantee a physically meaningful or calibrated latent. Abrupt dynamics changes can also make a history-conditioned estimate stale. A strong extension would fuse this internal response latent with exteroception while retaining modality dropout, expose predictive uncertainty, and train multi-step rather than only next-step response. That would distinguish “I know how the body is reacting” from “I am uncertain because the terrain ahead is unobserved,” making the latent more useful for safety supervision.

## Learning Humanoid Locomotion with Perceptive Internal Model

### Adding terrain to an internal dynamics model

Perceptive Internal Model (PIM) extends the authors’ earlier Hybrid Internal Model from blind response inference to humanoid terrain-aware control. Rather than render millions of camera images or train a separate depth encoder through teacher–student distillation, simulation supplies a robot-centric elevation grid directly. At deployment, LiDAR or RGB-D points are registered into the same grid. This keeps the policy’s training representation sensor-independent and makes it possible to swap hardware perception without retraining the controller.

### Observations, latent, and alternating optimization

Current proprioception includes commanded velocity, base angular velocity and projected gravity, joint position/velocity, and the last action. A history of these observations is combined with the local elevation map by the PIM. The latent must encode both hidden robot dynamics and terrain context, and the actor concatenates it with current state to output joint-position targets. The critic has privileged simulator information during PPO. As in HIM, optimization alternates: the perceptive model is frozen while PPO updates actor/value networks, then accumulated rollout data trains the internal model. This prevents a moving latent target from destabilizing every PPO minibatch.

An action curriculum initially holds less important upper-body joints near a nominal configuration and progressively releases them. This reduces the early exploration dimension for high-DoF humanoids while eventually allowing natural whole-body balance. The method does not use a reference motion; command tracking, posture, foot/contact, smoothness, torque, and collision rewards shape the gait.

### Sensor realization and scope

Two real perception stacks instantiate the same grid. One uses a Livox Mid-360 for both point clouds and FAST-LIO odometry. The other combines a RealSense D435 depth camera with T265 visual–inertial odometry. This is a meaningful systems experiment: the learned controller is tied to a geometric interface rather than a specific camera image distribution. Tests cover Unitree H1 and Fourier GR-1 humanoids on stairs, gaps, platforms, and outdoor terrain.

The paper positions PIM as a one-stage, computationally modest alternative to image rendering and multi-stage privileged-to-vision distillation. It also supplies anticipation unavailable to blind HIM: the elevation map shows an obstacle before contact, while proprioceptive history still captures unmodeled response and observation delay.

### Assessment

The clean geometric interface is the strongest design decision. It improves sensor interchangeability and makes the latent’s role clearer than an unconstrained recurrent visual policy. Yet the controller is not truly localization-free: both demonstrated stacks need odometry to register points, and map error can accumulate through motion. A 2.5-D grid cannot represent overhangs or distinguish traversable grass from rigid support. The latent remains difficult to audit, and alternating optimization offers no guarantee that it preserves rare safety-critical states.

Next work should include map uncertainty and age, train with correlated odometry drift rather than only point noise, and decode the latent into interpretable quantities such as support height and base velocity. A multi-layer/voxel extension would address confined spaces. Cross-sensor testing without policy fine-tuning should be reported quantitatively to support the claimed sensor independence.

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

## MuGen: Multi-Skill Generative Locomotion Controller for Humanoid Robots

### A dynamics-aware discrete motion vocabulary

MuGen learns a generative humanoid controller from heterogeneous human motion rather than a separate reward-engineered policy per skill. A VQ-VAE maps behavior into a finite codebook. Unlike a kinematic autoencoder, its decoder is trained through a differentiable learned dynamics model, encouraging codes whose actions are physically executable.

### Teacher, world model, and VQ policy

The world model predicts next-state change from current state and teacher action. Backpropagation through predicted rollouts trains reference tracking. An encoder maps state/reference context to a vector, quantization selects a code, and a decoder conditions on that code plus current state to output joint targets. Commitment/codebook losses organize the latent while physical tracking supplies meaning.

The student lacks privileged reference/state information. It shares the frozen codebook and learns action behavior cloning plus latent-code matching. A DAgger-style schedule mixes teacher and student execution, annealing teacher selection to zero and reducing one-shot distribution shift. A proprioceptive prior encoder can later predict code indices, enabling motion generation without a future reference.

Experiments use a selected one-hour LaFAN1 subset and a 23-DoF Unitree G1. The network emits targets at 30 Hz; a 500 Hz PD loop follows them. Moderate history improves continuity and unseen-motion success, while excessive history hurts capacity. MuGen reports 0.9535 unseen-motion success in its history ablation versus 0.6070 for a direct MLP baseline, with lower joint/velocity error.

### Novelty and limits

The main contribution is learning the skill representation *inside physics*. Discrete codes avoid continuous-VAE averaging and can be reused for tracking or generation; latent plus action distillation preserves semantics and immediate control.

The world model is also the largest risk: rollout bias can produce codes exploiting model error, and a finite codebook may collapse or omit rare contacts. Results emphasize tracking more than perceptive rough terrain, so inclusion here reflects controller relevance.

Future work should quantify code utilization and transitions, use ensembles or real rollouts against model exploitation, and condition generation on terrain/contact affordances. A vision-conditioned selector would demonstrate autonomous use of the learned vocabulary.

## Agile-WAM: An Agile Tactile World Action Model for Contact-Rich Robot Control

Agile-WAM asks whether a world-action model can retain predictive physical reasoning without the large pretrained video generator that makes many WAMs too slow for contact control. Its main result is a compact imitation-learning policy that jointly predicts an action chunk and future visual/tactile representations. The important distinction from simply concatenating touch to a visuomotor policy is that tactile evolution is itself a supervised prediction target, so the representation must encode how contact is likely to change under the generated motion.

### Inputs, outputs, and network design

At time (t), the deployable input is one current RGB observation, a tactile force field (TacFF), and robot proprioception (q_t). The simulation experiments use a 256×256 wrist image and a 10×14×3 TacFF on a 7-DoF Franka Panda; real experiments use a 320×240 Intel RealSense D405 wrist view at 30 FPS and a 32×32 piezoresistive gripper sensor at 30 Hz on a 7-DoF Flexiv Rizon 4. This is wrist-camera plus local tactile perception, not LiDAR, language, or an external scene map.

Separate ResNet-18 encoders process vision and touch—the visual encoder is ImageNet-pretrained while the tactile encoder is trained from scratch. Their features and proprioception are concatenated and linearly projected into a shared source latent. An action autoencoder supplies a structured latent for the demonstrated action sequence. A lightweight MLP parameterizes an ODE velocity field that transports the observation latent directly to a joint target containing an action latent, future visual latent, and future tactile latent. Because sensory context is the flow's starting distribution, it is encoded once rather than repeatedly injected through cross-attention. Six explicit-Euler integration steps produce the output, after which the action decoder returns a horizon-(H) action chunk and only the first (h) actions are executed before replanning.

The temporal design is unusually deliberate. Touch is predicted one step ahead, (h_{tac}=1), because forces can change abruptly at contact. Vision is predicted at the longer executed-action offset, (h_{vis}=h), because adjacent images are often nearly duplicates. The training loss combines flow matching, L1 action-autoencoder reconstruction, L1 supervision on the ODE-generated action, and L2 prediction of future visual and tactile latents. This is behavior cloning with conditional flow matching; it is not reinforcement learning, and the future branches predict latent features rather than photorealistic pixels.

### Evidence, novelty, and judgment

Training uses only 20–50 demonstrations per simulated task and 50 Meta Quest 3 teleoperation demonstrations per real task. Evaluation covers nine ManiFeel simulation tasks and five physical insertion/assembly tasks. Across the five hardware tasks, Agile-WAM reports a 29.4% relative overall success improvement over the strongest baseline and 11.9 ms inference latency. Its recovery examples are especially meaningful: after visual alignment becomes poor or occluded, the robot maintains light surface contact and uses touch to search for the insertion. Ablations indicate that joint action–future learning, both sensory prediction targets, and modality-specific horizons each matter.

The highlight is not merely speed; it is matching model horizon to sensor physics. Slow appearance change and fast contact transients should not receive identical prediction targets. The compact direct-flow design also challenges the assumption that useful world modeling requires a giant video foundation model. However, it remains a per-task imitation policy trained from modest demonstrations, not a broadly pretrained world model. The fixed wrist camera and gripper tactile array cover local manipulation but do not establish transfer across viewpoints, tools, sensors, robots, or unseen task semantics. Predicting latent futures may improve control without yielding an interpretable or rollout-accurate dynamics model.

The strongest next tests would hold data and backbone capacity constant against a non-predictive flow policy, evaluate sensor dropout and tactile calibration drift, quantify force safety rather than success alone, and test whether predicted tactile latents detect slip or impending jamming before failure. Longer closed-loop tasks need uncertainty estimates and a rule for aborting when imagined futures conflict with observations. Cross-sensor pretraining, variable-delay fusion, language/task conditioning, and adaptation to unseen objects would determine whether Agile-WAM is a reusable tactile world-action model or an excellent high-frequency contact policy.


## Efficient-WAM: A 1B-Parameter World-Action Model with Low-Cost Future Imagination

Efficient-WAM targets the main deployment weakness of large world-action models: they spend most of their latency rendering visually convincing futures even though the controller only needs future features that improve action choice. The paper's claim is deliberately action-centric—coarse prediction can be sufficient if it preserves object motion, contact layout, and robot dynamics. The 1B-parameter system reaches roughly 98 ms per action chunk on an RTX 4090, reported as more than 30× faster than the Motus WAM while remaining competitive on manipulation success.

### Conditioning, coupled experts, and generated outputs

The policy conditions on the current visual observation (o), a language instruction (l), and robot state (s). Its outputs are an (H)-step continuous action chunk and explicit future visual latents. It is therefore a genuine VLA-style command interface coupled to a predictive video branch—not merely a video model used offline. The two outputs are generated by coupled conditional-flow-matching experts inside a Mixture-of-Transformers (MoT): a video expert denoises future scene tokens, an action expert denoises the action sequence, and joint attention lets action tokens use imagined scene structure. Both branches use independently sampled noise levels during training and mean-squared velocity-field objectives.

The video expert is transferred from WAN-2.2-5B rather than trained without a visual prior. Structured compression reduces the teacher from 30 transformer layers to a narrower 12-layer, approximately 0.8B video expert. Distillation matches selected hidden-state anchors and temporal feature changes so the smaller branch retains appearance and motion knowledge. A dedicated action expert brings the total model to about 1B parameters. This design separates visual-world capacity from motor generation more clearly than a monolithic transformer, but its action quality is still conditioned by the learned visual tokens.

Two additional mechanisms reduce computation. First, the current observation remains high-resolution while future frames are represented by downsampled latents, reducing future-token density to one quarter at the same temporal horizon. Current and future grids receive separate positional encodings before concatenation. Second, asymmetric denoising gives the expensive video stream only two updates while the action stream receives five or ten. The first two MoT passes jointly update video and action and cache video keys/values; later iterations update only action queries against those cached features. The cache is rebuilt for each action chunk. Efficient-WAM-RT uses the aggressive configuration for physical deployment.

### Evaluation and what the result means

On 50 RoboTwin 2.0 tasks, Efficient-WAM reports 86.7% average clean and 85.7% randomized success; the real-time variant reports 83.1% and 82.0%. On four real Astribot S1 manipulation tasks—pipette-tray grasping, reagent-bottle transfer, LEGO color sorting, and bimanual pen uncapping—Efficient-WAM-RT averages 66.25% success, 98 ms chunk latency, and 6.1 ms amortized step latency. The full futures are visibly sharper than the real-time model's, yet the coarse variant retains task-relevant motion and contact geometry. This is strong evidence that photorealism is a poor proxy for control value.

The most useful idea is allocating compute by downstream utility: preserve high-resolution evidence about the present, aggressively compress the hypothetical future, and spend extra refinement on actions. Still, “30× faster” is a comparison with a particularly heavy baseline on one GPU and should not be read as universal real-time performance. A 98 ms planning pause can remain important for fast contact, and amortized per-step latency assumes safe open-loop execution of the chunk. Results also combine architectural compression, distillation, token reduction, and sampling changes; each saves cost but can fail differently under occlusion or distribution shift.

Future work should measure end-to-end camera-to-actuator latency, chunk interruption behavior, calibration of imagined futures, and whether low-fidelity predictions preserve rare safety-critical events. Adaptive computation could increase video steps only when uncertainty or observation–prediction disagreement is high. Tests with unseen instructions, novel objects, camera shifts, deformables, and longer-horizon error recovery would clarify whether compressed imagination preserves generalization rather than only benchmark success. A valuable ablation would replace video tokens with equal-cost learned latent dynamics to determine whether recognizable future imagery is needed at all.
