# Motion Generation — Paper Notes

The papers here do not all mean the same thing by “generation.” Some learn a motion distribution, some generate reusable priors or discrete tokens, and others build an interactive controller around a generator. Each note therefore follows the paper's own central claim and asks whether its representation, conditioning mechanism, and evidence support that claim. Network and perception details are included only when they explain the contribution. Local PDFs were used for AMP, DeepMimic, MaskedMimic, OMG, Perpetual Humanoid Control, and SMP; primary paper/project material was used for the remaining entries.

## AMP Adversarial Motion Priors for Stylized Physics-Based Character Control

### The problem AMP changes

AMP learns goal-directed physics controllers whose motion resembles an *unstructured distribution* of demonstrations. Unlike a tracker, it is not told which clip or frame to imitate at the current instant. This removes phase synchronization and the motion-planning layer needed to select and splice references. A dataset defines style; a conventional task reward defines what the character must accomplish. The policy discovers how to combine running, jumping, rolling, striking, turning, or recovery while pursuing that task.

This distinction makes AMP a motion prior rather than a high-level sequence generator. It does not directly sample an entire animation. Physics, policy feedback, task commands, and the style reward jointly produce motion online.

### State, action, and network architecture

The policy observes root-relative link positions, link rotations in a 6-D normal–tangent representation, linear/angular velocities, and a task goal such as desired direction/speed or target position. Expressing state in a root-facing local frame removes irrelevant world translation and heading. The Gaussian actor has an input-dependent mean and fixed diagonal covariance. Its MLP uses hidden widths 1024 and 512 with ReLU and a linear action head. Actions are target joint positions: spherical joints use 3-D exponential-map rotations and revolute joints use scalar angles, with PD servos producing torque.

Bullet runs physics at 1.2 kHz and the policy is queried at 30 Hz. PPO trains the actor and a similarly sized value function. This separation between a slow neural target policy and fast stabilizing PD loop is a major reason the method handles high-dimensional characters reliably.

### Adversarial prior and reward

The discriminator receives pairs of consecutive local motion features `(s_t, s_{t+1})`, not task goals. Positive examples come from raw mocap/keyframe clips; negatives come from policy rollouts. A least-squares GAN objective provides more useful gradients than a saturated binary classifier. The resulting style reward is combined roughly equally with the task reward in the reported setup (`w_G = w_S = 0.5`). A gradient penalty on discriminator observation features, with reported weight 10, is critical—without it, learning fluctuates and motion quality collapses.

The discriminator’s limited transition window is intentional. It judges local movement statistics without requiring alignment to a full reference, allowing the actor to synthesize transitions and behaviors absent as exact clips. Demonstrations require no action labels, phase labels, skill segmentation, or task annotation.

### What the experiments establish

AMP reproduces single-clip motion competitively with tracking baselines but is most valuable on mixed datasets and new goals: characters steer at requested speeds, cross obstacle courses, strike targets, and adopt zombie, stealth, or other styles by swapping the reference corpus. The method works across humanoid and nonhuman/fictional morphologies. Composition emerges through the task objective rather than a hand-authored state machine.

### Technical judgment and next work

The enduring idea is the separation of *what to do* from *how motion should look*. It made unlabelled motion data reusable across tasks and inspired many humanoid style priors. But the discriminator is also the weakness: adversarial co-training is unstable, reward scale drifts with the current policy, and a classifier can reward dataset artifacts. Dataset likeness is not biomechanics, energy efficiency, safety, or contact correctness. Short transitions may miss long-term rhythm and intent, while a broad multimodal dataset can let the policy exploit implausible paths between modes.

Later score/diffusion priors address reuse and stability; terrain-conditioned or state-dependent discriminators address conflicting modes. A stronger successor should attach calibrated contact/force features, use longer multi-scale temporal windows, and test held-out tasks with fixed prior weights. Human preference and physical-load metrics should complement discriminator scores. For physical robots, actuator limits, latency, and domain randomization must be part of the prior’s training distribution rather than left entirely to downstream PPO.

## DeepMimic: Example-Guided Deep Reinforcement Learning of Physics-Based Character Skills

### Goal and historical contribution

DeepMimic showed that model-free PPO could turn a kinematic example into a robust feedback controller for running, backflips, cartwheels, spinning kicks, rolls, and terrain locomotion. It retains the visual quality and timing of the reference while allowing perturbation recovery and additional goals. In the progression represented by this folder, it is the canonical *explicit tracker*: later papers relax its phase/reference requirements rather than making it obsolete.

### Observations, action, and policy

The state is expressed in a character-centric coordinate system with the root at the origin and x-axis along facing direction. It includes every link’s root-relative position, orientation, linear velocity, and angular velocity. A normalized phase `phi in [0,1]` identifies the target frame; cyclic motions wrap phase at the end. Task policies additionally see commands such as travel direction, speed, or strike target.

The Gaussian actor uses two fully connected ReLU layers of 1024 and 512 units and a linear mean head; covariance is a fixed hyperparameter. A similar network estimates value. At 30 Hz, the actor outputs target orientations for PD controllers—axis-angle for spherical joints and scalar angles for hinges. PPO uses GAE and a TD-lambda value target. For terrain tasks, a uniform local height map passes through three convolutional layers (16 `8x8`, 32 `4x4`, 32 `4x4` filters) and a 64-unit FC layer before concatenation with body state/goal and the main MLP.

### Imitation objective and the two essential curricula

Reward contains weighted exponentials of joint-pose error, joint-velocity error, end-effector position error, and center-of-mass error, plus a task term. This decomposed metric communicates much more than a binary success signal but requires manual scales/weights.

Two training choices make dynamic motion learnable:

- **Reference-state initialization (RSI):** an episode starts from a random frame of the reference rather than always frame zero. The policy immediately visits aerial, inverted, and landing states that ordinary exploration would almost never reach.
- **Early termination:** a rollout ends when the character diverges substantially or falls. Compute is not spent collecting unrecoverable low-reward states, and the value function sees a sharper distinction between viable and failed motion.

Ablations show these are structural, not cosmetic: without RSI, exploration cannot discover difficult mid-clip states; without early termination, training is diluted by long fallen rollouts.

### Multiple clips and skill selection

The paper studies three extensions. A max-over-clips reward lets one policy imitate similar examples; a skill-selector command chooses among diverse motions; and a learned value function can decide which skill is best for a task/transition. These experiments foreshadow motion libraries and mixture models, although they still depend on explicit clip identity or reference phase.

### What is achieved

DeepMimic produces high-fidelity, disturbance-responsive motion on 3-D humanoids and other characters, including difficult intermittent-contact acrobatics. Terrain variants adapt a reference gait to irregular geometry, while goal terms allow running direction or ball/target interactions. No camera is used: “vision” in the terrain experiment is a perfect simulated local height map.

### Assessment and future directions

DeepMimic’s strongest lesson is that demonstrations can define dense exploration structure while physics supplies recovery and adaptation. Its weakness is equally clear: phase is an oracle clock, one policy remains strongly tied to its clips, and a dynamically necessary timing change can conflict with reference reward. The method also assumes reasonably retargeted, physically trackable examples.

Future designs replace phase with future-reference windows, masked constraints, motion tokens, or distributional priors. A valuable modern baseline would retain DeepMimic’s transparent error decomposition but infer phase/contact timing, expose reference feasibility, and report torque/impact—not only pose error. For robot deployment, state estimation, latency, actuator bandwidth, and fall safety must be integrated rather than treating simulation pose as directly observable.

## MaskedMimic: Unified Physics-Based Character Control Through Masked Motion Inpainting

### Control as a missing-data problem

MaskedMimic unifies full reference tracking, sparse keyframes, selected joint targets, joystick/path goals, text, object interactions, and combinations by treating them as partially observed descriptions of a complete motion. The policy “inpaints” a physically feasible full-body behavior through closed-loop simulation. This avoids building a separate controller/reward for every input interface and permits zero-shot combinations such as text-stylized path following.

### Fully constrained teacher

The first stage trains a complete motion tracker with A2C. It observes local-frame 3-D body pose/velocity, the next `K` full target poses, and scene height. Each future joint token contains rotation relative to both current joint and root plus position relative to current joint/root, with time-to-target. The actor is Transformer-based; a fully connected critic supplies value. Actions are Gaussian PD position targets with fixed covariance—no residual root force or meta-controller is allowed.

Rewards match global joint positions/rotations, root height, joint linear/angular velocities, and penalize energy. Training spans flat ground, irregular stairs/slopes/roughness, and an object playground. Early termination allows more deviation on irregular ground (0.5 m versus 0.25 m on flat) so terrain adaptation is not punished as failed imitation. Failure-prioritized motion sampling concentrates on difficult flat-ground clips without treating an impossible cartwheel up stairs as a learnable failure.

### Partially constrained conditional VAE

The student comprises a learned prior, posterior encoder, and action decoder. The prior sees current pose, `16 x 16` local height samples spaced 10 cm, five historical poses subsampled from the previous 40 steps, optional object/text tokens, and masked future targets. Object input uses eight 3-D bounding-box corners, 6-D direction, and category; XCLIP maps text to 512 dimensions. Training uses 11 future poses: ten immediate frames plus one random long-term target.

Each modality has a `[256,256]` token encoder. The prior Transformer has 512-D latent width, 1024-D feed-forward blocks, four layers, and four heads. Two heads map its first output token through `256 -> 128 -> 64` to Gaussian mean/log standard deviation. The posterior encoder is a `[1024,1024,1024]` MLP that also sees complete unmasked targets; the decoder is another three-layer 1024 MLP taking current pose, terrain, and a 64-D latent to deterministic PD targets.

DAgger-style collection lets the student act while the complete teacher labels actions. KL weight rises from `1e-4` to `1e-2`, gradually converting an action imitator into a usable generative latent. One noise sample is held fixed for an episode, preventing frame-to-frame style flicker. Structured masks remain temporally consistent 98% of the time so a limb stays observed or hidden over meaningful intervals.

### Scale and evidence

The 30 Hz policy is trained in 16,384 Isaac Gym environments across four A100s for roughly two weeks—about 30 billion teacher and 10 billion student steps. AMASS supplies broad motion/keyframes, HumanML3D supplies text, and SAMP supplies human–object interactions. The resulting single model handles sparse targets, long-horizon text, terrain, furniture, path steering, and novel condition combinations.

### Judgment and future work

The conceptual highlight is a common *constraint language*: every interface specifies what is known, masks specify what is free, and physics fills the remainder. Its cost is enormous training compute and dependence on mask/data coverage. A Gaussian 64-D latent can still average incompatible solutions; text describes motion but not necessarily physical scene semantics; bounding boxes omit fine contact geometry. Early termination also leaves the teacher unreliable in severe failures, which limits DAgger labels.

Future work should use contact-aware/discrete or diffusion latents for stronger multimodality, learn mask curricula from real user queries, and add uncertainty when constraints conflict. A robot version needs actuator/delay randomization and a safety layer. Evaluating constraint satisfaction separately from realism, energy, collision, and diversity would make clear whether inpainting obeys the user or merely produces a plausible nearby behavior.

## MuGen: Multi-Skill Generative Locomotion Controller for Humanoid Robots

### Dynamics-aware discrete skill generation

MuGen builds a reusable discrete motion vocabulary for a physical humanoid. Direct reference-to-action MLPs can memorize training clips, while a purely kinematic VAE can generate trajectories that look plausible but cannot be tracked. MuGen learns its Vector-Quantized VAE policy *through a differentiable learned world model*, so codebook entries are shaped by simulated robot dynamics and action feasibility.

### Teacher, world model, and VQ bottleneck

The learned world model predicts the change in robot state from current state and teacher action. A teacher encoder maps current/reference context into a continuous feature, nearest-neighbor vector quantization selects a codebook vector, and a decoder conditions on that code plus current state to output a distribution over joint targets. Backpropagation through predicted state rollouts supplies a model-based RL tracking objective. Reconstruction, codebook, and commitment terms keep codes informative and prevent arbitrary encoder drift.

The quantized bottleneck serves two purposes. It compresses hours of heterogeneous motion into recurring dynamically meaningful skills, and it prevents the smooth-but-blurry averaging common in continuous VAEs. A code can later be selected by a reference encoder, a proprioceptive prior, or another downstream task.

### Student distillation and deployment I/O

The privileged teacher and deployable student have different observations but share the codebook. The student matches both teacher actions and teacher code indices/features. A DAgger-style smooth-transition schedule initially executes the teacher frequently, then linearly anneals teacher use to zero. This avoids abruptly placing an immature student into states its behavior-cloning data never covered.

After distillation, a prior encoder maps proprioceptive history to a distribution over codebook entries, allowing generation/control without privileged future state. The Unitree G1 embodiment uses 23 controlled DoFs (12 legs, 10 arms, one waist). The network issues joint-position targets at 30 Hz and a 500 Hz PD loop realizes them. No camera or terrain sensor is central; this is reference/proprioception-conditioned motion generation and tracking.

### Dataset and results

The reported experiments use a one-hour selected LaFAN1 subset for broad evaluation plus a dynamic dance clip, measuring rollout survival, mean joint rotation error, and mean velocity error on seen/unseen references. Temporal context matters: removing history sharply reduces unseen success, while too much history introduces noise/capacity pressure. The history-enabled MuGen variant reports `0.9535` unseen-motion success in the relevant ablation versus `0.6070` for a direct MLP, alongside lower rotation/velocity error. Real G1 demonstrations validate executable tracking.

### Technical judgment

The important contribution is not “VQ-VAE for motion” alone; it is aligning the discrete representation with closed-loop physics and transferring the same latent semantics to a deployable student. This makes the codebook more reusable than clip IDs and more structured than an unconstrained hidden layer.

The learned dynamics model is the central risk. Multi-step error can teach policies to exploit model bias, while finite codes can collapse, leave rare motions uncovered, or produce discontinuities at code changes. Results emphasize locomotion/motion tracking rather than terrain-aware interaction, and a fixed G1 codebook may not transfer across embodiments.

Future work should report code utilization, transition matrices, and rare-skill coverage; use model ensembles or real/simulator consistency to suppress exploitation; and condition codes on contacts/terrain. Hierarchical generation over code sequences could provide long-horizon intent, while a continuous residual could restore fine detail lost to quantization. See the primary [arXiv paper](https://arxiv.org/abs/2605.24592).

## OMG: Omni-Modal Motion Generation for Generalist Humanoid Control

### Goal: one generator behind many interfaces

OMG builds a high-level motion “brain” that maps language, music/audio, human reference motion, sparse teleoperation keypoints, or mixtures of these signals into future Unitree G1 trajectories. A separate HoloMotion policy tracks the generated trajectory. The contribution is therefore not just another text-to-motion model: it is a common robot-motion distribution whose conditions can be attached, removed, or combined without training an independent generator for every interface.

This division also clarifies what OMG generates. It produces kinematic robot references at 30 Hz, not torques. The low-level tracker supplies disturbance rejection, contact handling, and motor commands. The system can synthesize motions that are absent as an exact training sequence, but its physical validity is inherited from both the robot-space training data and the tracker's envelope.

### OMG-Data and motion representation

OMG-Data aggregates more than 1,000 hours of heterogeneous motion. Human mocap and other sources are retargeted to G1, standardized to 30 Hz, annotated with the conditions available for each source, and filtered with simulation-in-the-loop tracking so grossly unexecutable sequences do not become generator targets. That filtering step is important: a large generative model can otherwise learn an attractive human-motion distribution that a robot with different proportions and limits cannot realize.

Each motion frame is a 125-D continuous feature vector containing canonicalized root position, root rotation, robot joint angles, and body-link positions. A root-centric coordinate frame is anchored to the last observed frame, reducing dependence on absolute world location while keeping the generated continuation geometrically consistent with recent motion. The model sees ten history frames and predicts a 60-frame future—about two seconds at 30 Hz. This gives substantially more temporal context than a reactive locomotion actor, although it still relies on receding-horizon regeneration for long behavior.

### OMG-DiT architecture and modality injection

OMG-DiT is an `x`-prediction diffusion transformer: it directly denoises continuous future motion rather than decoding from a learned motion tokenizer. Future-frame tokens interact through self-attention; history and text enter through cross-attention. Rotary positional encoding represents temporal position, a sinusoidal diffusion-timestep embedding tells the network the current noise level, and GELU nonlinearities are used throughout the transformer.

The conditions are deliberately heterogeneous:

- A frozen T5 encoder supplies semantic text features.
- Frame-aligned audio features are projected from a 35-D representation.
- Human reference features use a 66-D representation aligned to the output timeline.
- Pico-style sparse teleoperation keypoints use an 18-D input.

Frame-aligned modalities are injected with feature-wise linear modulation (FiLM), whereas language is naturally handled as a sequence through cross-attention. Random condition dropout trains classifier-free guidance and prevents the backbone from assuming that all modalities are present. A newly added modality receives a lightweight encoder and zero-initialized FiLM adapter, so initial behavior matches the pretrained model and changes gradually during fine-tuning. This is the mechanism behind the paper's few-shot modality extension claim, not merely concatenating a new vector to an input layer.

The paper scales the same design from roughly 50 million to 500 million parameters. Improvements with model and data scale support the claim that the shared distribution benefits from capacity; they do not establish that scaling alone solves physical grounding.

### Runtime data flow and perception

At runtime, recent generated/executed motion and the active condition are sent to OMG-DiT. A denoised two-second reference is produced, then HoloMotion receives the relevant reference window plus robot state and outputs joint-level control targets. Text and prerecorded audio require no visual perception. Human-following and teleoperation modes depend on an upstream body/keypoint estimator, but the DiT itself is not an image network. There is no central camera/LiDAR terrain encoder, occupancy map, or object-state branch in the reported generator.

### What is distinctive, and what remains open

The highlight is the *adapter-friendly shared motion space*. Text can specify semantics, audio can specify timing, and keypoints can constrain selected body parts while the same prior fills unconstrained joints. Random modality subsets create useful zero-shot composition instead of forcing an application to choose exactly one control mode. Direct continuous diffusion also preserves fine joint detail that a small discrete codebook might discard.

The hierarchy creates an equally important limitation: the generator does not observe tracking error, contact failure, terrain, or motor saturation. A trajectory can be statistically plausible yet leave the tracker an impossible transition. Much of the corpus is flat-ground motion, and simulation filtering only checks motions under the filter's tracker and simulation assumptions. Diffusion inference and offboard compute also introduce latency that is mostly hidden by the short-horizon tracker.

A strong continuation would close the loop by feeding tracker error, contact state, actuator margin, and local geometry back into generation; train with hard negative trajectories that failed execution; and report condition-conflict behavior when text, music, and keypoints disagree. Safety constraints, online latency distributions, and real-robot success should be measured separately from offline FID-style motion quality. Jointly adapting generator and tracker could improve feasibility, but should preserve the modular interface that makes OMG practically attractive.

## Perpetual Humanoid Control for Real-time Simulated Avatars

### Problem and achievement

Perpetual Humanoid Control (PHC) targets a simulated avatar that can follow the breadth of AMASS motion continuously, recover after failure, and resume a changing reference without an external root force or manual reset. A single fixed-size tracker tends to sacrifice rare, dynamic clips as the dataset grows; a conventional imitation controller also has little reason to learn useful behavior after it falls. PHC addresses these as capacity-allocation and data-curriculum problems. It reports successful tracking on 98.9% of AMASS—roughly 10,000 clips and 40 hours—and demonstrates 30 fps control from live monocular pose estimates or generated language motion.

### Humanoid, observation, action, and reward

The avatar follows the SMPL topology with 24 rigid bodies and 23 actuated joints. Body shape is represented by the ten SMPL shape coefficients `beta`. The policy observes proprioception—joint/body configuration `q`, velocity `q_dot`, and shape—and a goal encoding built from the difference between the current character and upcoming reference. The full-pose version uses relative positions, orientations, and velocities; the keypoint-only variant accepts reference joint positions, which is much better matched to noisy vision pipelines that cannot reliably estimate global joint rotations.

The Gaussian actor outputs a `23 x 3` set of desired joint rotations for PD servos. Crucially, these are direct targets, not residuals added to the reference pose, and PHC does not use residual root forces or a learned meta-PD controller. That makes success a stronger test of physical control, although it is still an idealized simulated body.

Reward mixes reference tracking with a learned natural-motion term. Tracking covers body position/orientation and velocity agreement. The AMP discriminator judges ten-step histories of proprioceptive motion, giving it enough temporal context to discourage locally correct but jerky transitions. In the reported objective, tracking and AMP terms have equal top-level weight (`0.5` each), with an additional energy cost. Relaxed early termination tolerates larger ankle and toe errors, since strict thresholds at these distal contacts would incorrectly terminate recoverable tracking.

### Progressive Multiplicative Control Policies

PMCP grows capacity only where the existing controller fails:

1. Train an initial primitive on the full motion set.
2. Evaluate every clip and collect the persistent failures.
3. Freeze the learned primitive, add a new primitive, and train it on the hard subset.
4. Repeat until failure coverage is sufficiently small, then add recovery experience.

Each primitive is a Gaussian policy. Primitive, critic, discriminator, and composition networks use two-layer MLPs with hidden sizes 1024 and 512. Rather than hard-switching between primitives, a learned multiplicative composer forms a product of their Gaussian action distributions. A primitive whose action distribution is sharp in a relevant dimension can influence that joint strongly, while overlapping competence can be blended. Freezing old primitives limits catastrophic forgetting; mining failures ensures extra capacity is not wasted on clips already solved.

Recovery is learned as another capability. Training initializes the avatar in fallen or strongly perturbed states and rewards returning to viable reference tracking, using a simple locomotion reference as the bridge. PHC distinguishes ordinary tracking failure, a fallen pose, and states that are both fallen and far from the reference. This teaches a behavioral path back to the motion manifold rather than teleporting or resetting the root.

### Live-control pipeline and evidence

For offline AMASS evaluation, the reference and simulator state are directly available. In the webcam demonstration, a separate monocular pose estimator turns RGB video into 3-D human keypoints; PHC receives those keypoints, not pixels. The same position-only interface can consume motion produced from text. Thus the controller has proprioceptive and kinematic-reference inputs but no learned visual encoder, object detector, depth map, or terrain perception of its own.

Ablations support the structural choices: progressive primitives cover difficult tails better than simply enlarging one monolithic policy, AMP improves recovery naturalness, and relaxed termination prevents excessive failures at feet. The most important result is continuity—fall recovery is embedded in the same reference-following system, so a user-controlled avatar need not be reset whenever the upstream pose estimate glitches.

### Technical judgment and next work

PHC's novelty is not a new generative model. It is an effective way to turn a huge, uneven motion corpus into a persistent controller, preserving earlier skills while allocating specialists to failure modes. Multiplicative composition is more graceful than a hard expert selector and the position-only mode makes the system genuinely usable as an avatar backend.

The price is a growing collection of frozen experts whose size and inference cost depend on the dataset's tail. Failure mining may create experts for sensor artifacts rather than meaningful skills, and multiplying confident but incompatible Gaussians can yield a poor compromise. Recovery is learned around the simulated avatar and simple locomotion distribution; it is not a guarantee against every contact configuration. Shape variation, imperfect ground contacts, and monocular keypoint noise are addressed more than actuator dynamics, latency, or physical hardware safety.

Useful follow-up work would distill the primitive ensemble into a compact conditional policy, learn when capacity growth is warranted, and expose uncertainty when reference input is implausible. For robots, recovery should be conditioned on terrain/contact sensing and constrained by self-collision, motor temperature, and impact limits. Evaluation should include recovery time, energy, contact impulse, and how accurately control resumes after recovery—not only whether the avatar eventually stands.

## SMP Reusable Score-Matching Motion Priors for Physics-Based Character Control

### Why replace adversarial motion priors?

SMP turns a motion collection into a frozen, reusable reward model. In AMP, the discriminator is trained against the current policy, so its scale and decision boundary move during every downstream task; the original demonstration data must remain available; and a prior trained for one policy cannot simply be handed to another. SMP instead learns the reference distribution once with denoising score matching. Downstream PPO controllers query that fixed model for a naturalness reward without accessing the mocap set.

This is a prior rather than a runtime sequence generator. The diffusion network scores short motion windows during training. Once a task policy is learned, the policy itself runs reactively and the score model can be removed from deployment.

### Motion representation and diffusion model

Training examples are ten-frame windows. Each frame describes local/root-invariant motion using root linear and angular velocity, local joint rotations in a continuous 6-D representation, and root-local end-effector positions. These features retain dynamics and contact clues while removing absolute heading and translation that should not define style.

The score model is a compact two-layer Transformer encoder of about three million parameters. A forward diffusion process adds noise over 50 timesteps, and the network predicts the injected noise `epsilon` from the corrupted window and timestep. Exponential moving average weights stabilize the learned model. Depending on dataset size, prior training takes roughly 400,000–800,000 iterations and up to about five hours on one RTX 4090; the experiments include the more-than-20-hour 100STYLE corpus. This is substantially cheaper than repeatedly adversarially fitting a discriminator for every task.

### From a denoiser to an RL reward

For a policy-generated window, the method adds a selected noise level, predicts its noise with the frozen network, and uses the squared prediction/score error as an energy: motion agreeing with the learned data distribution receives higher reward. A naive single noise level is brittle. Very low noise only judges tiny local deviations; high noise blurs styles and makes the reward weak. SMP therefore uses an ensemble of noise timesteps—reported as `K = {22, 15, 8}`—and combines their judgments.

The raw error magnitude differs across noise levels, so each component is adaptively normalized using its running statistics before exponential reward shaping. This calibration is central: without it, one timestep can numerically dominate even when it is not the most informative. Independent priors can also be combined with weights, enabling style composition without retraining one joint discriminator.

### Downstream controller and generative initialization

The downstream actor is an ordinary Gaussian MLP trained with PPO. It receives current humanoid proprioception plus task-specific commands or observations and outputs desired joint positions for PD control. Tasks include steering/target-directed locomotion and interactive skills such as striking or boxing. There is no camera built into the general prior; environmental input belongs to the task policy.

SMP also uses the diffusion model for generative state initialization. Instead of sampling starting states from the original clips, reverse diffusion synthesizes plausible motion/state windows. This both removes the dataset dependency after pretraining and exposes PPO to a wider support than always starting at recorded frames. It is an unusually practical consequence of choosing a generative score model rather than only a classifier.

### Evidence, strengths, and limitations

The paper finds motion quality competitive with adversarial priors across multiple control tasks while keeping one prior fixed. Prompting/composing priors changes style, and generated initialization remains effective after the reference corpus is discarded. The most significant advance is modularity: a studio or robot lab can distribute the frozen prior as a learned asset rather than distributing all motion data and rerunning an adversarial game.

SMP does not eliminate reward-model problems. A score is density under the demonstrations, not proof of balance, collision avoidance, low impact, or actuator feasibility. A policy may find high-score regions that are irrelevant to the task, and rare but valid behaviors can receive poor density. During training, evaluating several noisy windows adds multiple transformer passes per environment step. Results also depend on noise schedule, feature normalization, window length, and the running reward statistics; these are less visibly unstable than GAN training but still important knobs.

Future work should calibrate score reward against physical measures, learn multi-scale windows for both foot contacts and long-period style, and use uncertainty or an ensemble of independently trained priors to detect out-of-distribution behavior. Generative initialization should be rejected or projected when sampled states violate contact or joint limits. For a physical humanoid, the learned motion density should be paired with actuator/impact constraints and trained on robot-executable data, rather than assuming humanlike appearance equals safe execution.

## T-GMP: Terrain-conditioned Generative Motion Priors for Versatile and Natural Humanoid Locomotion

Source: [arXiv 2606.06944](https://arxiv.org/abs/2606.06944).

### Problem: naturalness must depend on the ground

T-GMP argues that one unconditional “humanlike locomotion” distribution is wrong for varied terrain. Level walking, balancing on a narrow beam, lowering the center of mass on a slope, and reaching an arm outward during a difficult step can have different whole-body statistics. A global discriminator may penalize exactly the adaptation needed to avoid falling. The paper instead learns a motion prior and discriminator conditioned on local terrain, then trains one humanoid policy across eight terrain families.

### Where the expert data comes from

The training set pairs robot state with the local height map that made that state appropriate. Privileged expert locomotion policies produce terrain-specific behavior, while retargeted human mocap from GMR provides natural coordination. The combined dataset is small by foundation-model standards—about 29.6 minutes or 88,800 frames—but explicitly covers slopes, stairs, beams, gaps, and other procedural geometry. Pairing is the crucial feature: the model is not asked to infer after the fact whether a crouched pose belonged to a slope or flat ground.

### Terrain-conditioned generative prior

A conditional beta-VAE reconstructs a short sequence of `T` consecutive expert states. State features include joint position/velocity and root-relative end-effector information. A two-layer convolutional encoder processes the local height map; the encoder uses motion plus terrain to infer a latent distribution, while the decoder reconstructs the expert sequence conditioned on latent and terrain.

At deployment, the decoder is deliberately conditioned only on the current local height frame rather than a privileged future terrain sequence. That design makes the prior compatible with an online height estimate, although it limits anticipation to the spatial field visible in the current map. The latent space provides a smoother family of terrain-appropriate behaviors than selecting one demonstration clip or imposing a single fixed style.

### RL actor, discriminator, and action interface

The PPO actor receives five observation frames. Each frame contains root angular velocity, projected gravity, joint state, and the previous action; a terrain CNN embeds the local height map. The command specifies desired movement, and the actor outputs 26 desired joint positions for PD control. It is therefore a history-aware proprioceptive/exteroceptive policy, not a camera-to-action model.

A separate terrain-conditioned adversarial discriminator uses a five-layer CNN for geometry and judges whether state transitions resemble experts *for that terrain*. The discriminator reward encourages coordinated style without suppressing necessary adaptations. A foothold penalty supplies more explicit contact guidance by discouraging placements inconsistent with the local support surface. Together, latent prior, conditional discriminator, and foothold term divide the problem into plausible motion, context-sensitive style, and local foot safety.

The real-robot pipeline reconstructs a robot-centered height map from LiDAR. The policy consumes that representation rather than a raw point cloud or RGB image. This is an important qualification: reported robustness depends on the mapping/preprocessing stack preserving step edges, beams, and gaps at the resolution assumed in simulation.

### Results and interpretation

Across the terrain curriculum, T-GMP improves success and smoothness relative to policies with weaker or unconditional natural-motion constraints. Qualitative behaviors—arms extending for balance and body height adapting to grade—support the claim that conditioning changes more than foot placement. The unified actor avoids a hand-authored gait/terrain mode switch and transfers to a physical humanoid using the LiDAR-derived map.

The strongest idea is to model `p(motion | terrain)` rather than `p(motion)`. This is broadly useful: the same visible pose may be natural on one surface and pathological on another. The compact paired dataset also shows that carefully structured expert coverage can matter more than raw mocap scale for environmental adaptation.

### Limitations and productive next steps

The prior can only learn correlations represented in the paired demonstrations. Privileged experts may inject their own unnatural biases; mocap retargeting may not match the terrain that produced the original human movement. A local height map says little about friction, compliance, moving supports, or whether a surface can bear weight. LiDAR occlusion and reconstruction delay can turn an apparently correct foot target into a miss, and a short-window VAE may smooth over distinct contact strategies.

Future variants should condition on terrain uncertainty, friction/material estimates, and predicted contact timing; use a temporal map memory rather than only the current height frame; and report performance under controlled map latency/noise. Actuator torque, slip, foot clearance, impact, and energy should accompany success rate. A mixture or discrete latent could preserve multiple valid ways to traverse the same obstacle, while online failure data could refine the expert-conditioned prior without allowing adversarial reward drift.

## TextOp: Real-time Interactive Text-Driven Humanoid Robot Motion Generation and Control

Sources: [paper](https://arxiv.org/abs/2602.07439), [project/code](https://text-op.github.io/).

### Goal and system boundary

TextOp lets a user revise a natural-language command while a Unitree G1 is already moving. Instead of generating a complete sequence once, it continuously generates small overlapping motion primitives and sends them to a universal tracking policy. This supports one uninterrupted stream that can move among locomotion, gestures, dance, martial arts, jumping, or instrument-playing motions without a prerecorded script or continuous human teleoperation.

The architecture is explicitly hierarchical. The high level is a text-conditioned latent diffusion model that produces kinematic robot references. The low level is an RL feedback policy that observes the physical robot and outputs joint targets. Language determines intent, while stabilization and disturbance response come from the tracker; the diffusion model does not directly command torque.

### Robot-specific motion representation

TextOp does not generate an SMPL pose and retarget it online. Its 69-D per-frame feature is defined directly for the robot's single-DoF joints. It contains continuous sine/cosine features for root roll and pitch, incremental root yaw, binary foot contacts, yaw-local root translation increments, root height, 29 joint positions, and 29 joint increments. Global planar position and heading are represented incrementally, making autoregressive continuation invariant to where the robot started.

This is more than an implementation detail. A human-skeleton representation permits rotations the robot cannot perform and adds a retargeting discontinuity between generator and controller. The ablations show better segment quality and transition smoothness from the DoF-local representation than generating human-format motion and retargeting afterward.

### VAE and latent diffusion architecture

At each generator update, two history frames condition the prediction of the next eight frames. With data at 50 Hz, a primitive covers 160 ms; generation runs at 6.25 Hz, so consecutive primitives tile the stream. The model follows DART and has two Transformer-based stages:

- A conditional VAE encodes the history-plus-future segment into a 128-D latent and reconstructs eight future frames from latent plus history. It uses hidden width 512, feed-forward width 1024, nine Transformer layers, and four heads.
- After the VAE is frozen, a latent diffusion Transformer learns the text-conditioned latent distribution. It also uses width 512 and feed-forward width 1024, with eight layers and four heads. A pretrained CLIP encoder supplies the text embedding.

The LDM predicts the clean latent from noise in only five denoising steps. Classifier-free guidance uses scale 5; text embeddings are randomly dropped during training to provide the unconditional branch. Losses supervise latent denoising, reconstructed features, body translation/rotation, joint angle/velocity, and contact-aware foot position. Thus the model is not relying on latent reconstruction alone to discover grounding.

Autoregression creates exposure bias because training histories are clean but runtime histories include previous predictions. TextOp uses self-rollout: over overlapping sequences, a segment's true history is randomly replaced with the preceding generated future, with replacement probability gradually increased. This directly trains recovery from the generator's own small errors.

### RL tracker: exact input and output

The tracker is an asymmetric actor–critic trained with PPO in 8,192 IsaacLab environments. Its actor and privileged critic are ELU MLPs with widths `[2048, 1024, 512]`. The actor receives 431 values:

- five future reference frames of joint positions and velocities (`290` values);
- five relative pelvis positions and 6-D pelvis orientations (`15 + 30`);
- projected gravity and pelvis linear/angular velocity (`3 + 6`);
- current joint positions/velocities (`58`); and
- the previous 29-D action.

The critic additionally sees positions and 6-D orientations of 14 key bodies. The actor outputs 29 joint-position targets. Simulation runs at 200 Hz and policy/control at 50 Hz; PD control converts targets to motor torque. Tracking rewards match pelvis and link pose/velocity, while penalties cover action rate, joint limits, unwanted contacts, sliding, hard landing, overspeed, and torque beyond actuator limits. Friction, restitution, joint zero offsets, torso center of mass, and external pushes are randomized.

The tracker is trained not only on filtered AMASS/private references but also on 5,368 generator rollouts totaling 31.48 hours. This is a particularly good design choice: online generated motion is noisier than curated mocap, and training only on clean data would make the hierarchy fail precisely at their interface.

### Dataset, deployment, and evidence

GMR retargeting and simulation filtering produce 12,296 AMASS clips (40.67 hours) plus 403 private dance/martial-arts clips (3.12 hours), all at 50 Hz. BABEL supplies frame-aligned action language; mirroring yields 83,478 text–segment pairs. The generator is trained for 200,000 VAE and 300,000 LDM steps, while the hardware tracker receives substantially longer PPO training.

On hardware, the tracker runs onboard through ONNX at 50 Hz. The generator runs with TensorRT on an external RTX 4090 at 6.25 Hz, and a motion buffer reconciles their rates. Reported module latency is about 7.6 ms for text encoding, 29.6 ms for generation, and 2.2 ms for tracking, but perceived command-to-motion response averages 0.73 s. Real trials demonstrate command changes and manual perturbations; random 30-second command streams succeed in 16 of 20 trials, while several fixed/looping commands reach 8–10 successes out of 10.

### Judgment and next directions

TextOp's highlight is not merely text conditioning; it makes *instruction replacement* a first-class temporal event. Short primitives, self-rollout, robot-native representation, and generator-augmented tracker training are a coherent solution to the seams that typically break a generator–controller hierarchy.

The language channel is still underspecified. “Punch” gives no safe target, collision volume, keep-out region, or social context. The high-level model observes generated history, not tracking error or the surrounding scene. No camera, LiDAR, object encoder, or terrain map is part of the policy described here, so collision-free or object-aware execution is not implied. Offboard generation and networking are additional failure points, and five denoising steps plus buffering explain why neural inference latency is much smaller than perceived response.

Next work should condition generation on measured execution error and contact state, add object/terrain tokens with explicit safety constraints, and degrade safely when communication stalls. Evaluation should test paraphrases and conflicting/rapid command changes, measure semantic reaction time automatically, and report both diversity and physical failure—not only offline text–motion scores. A small onboard fallback generator or command-independent stabilization buffer would make the interactive claim more robust outside a lab network.

## UniAct: Unified Motion Generation and Action Streaming for Humanoid Robots

Source: [arXiv paper](https://arxiv.org/abs/2512.24321).

### Goal: one autoregressive interface for several control modes

UniAct converts language, music, desired planar trajectory, or an imperfect reference motion into a single stream of robot motion tokens. Its motivation is the tradeoff between end-to-end language-to-action policies, which respond quickly but have limited semantic capacity, and offline diffusion-plus-tracker systems, which understand richer commands but wait too long for a complete plan. UniAct fine-tunes a multimodal language model for next-motion-token prediction, decodes causally, and begins streaming before the whole sequence exists.

The output of the high-level system is a sequence of 29 Unitree G1 joint positions, not torque. A PPO tracker consumes this reference and closes the physical feedback loop. The paper reports sub-500 ms response, more than 1,000 simulation trials, over 100 hours of physical operation, and a 19% gain in zero-shot success on low-quality reference motions relative to the compared methods.

### How each modality becomes tokens

Text uses the native Qwen2.5 vocabulary and embeddings. The remaining signals are discretized for insertion into the same autoregressive sequence:

- **Motion:** each frame is the G1's 29 joint angles. A temporal 1-D convolutional encoder with residual blocks compresses motion, Finite Scalar Quantization (FSQ) rounds bounded latent scalars, and a causal convolutional decoder reconstructs continuous joints. The allocated motion vocabulary has 15,360 codes.
- **Music:** sampled at 30 Hz, each frame contains a 35-D feature: amplitude envelope, 20 MFCCs, 12 chroma components, beat peak, and onset. A separate convolutional FSQ encoder produces one of 6,144 music tokens.
- **Trajectory:** root paths are sampled at 5 Hz, converted to displacement in the previous root frame, and represented by heading changes quantized into 6-degree bins, giving 60 tokens.

FSQ avoids vector-quantizer codebook lookup/collapse and bounds generation to joint patterns seen through the tokenizer. It does not make every emitted token sequence dynamically feasible: valid individual codes can still form a bad transition.

### Multimodal LLM and generation protocol

UniAct fine-tunes Qwen2.5-3B. It forms one vocabulary by reserving ranges for text, motion, music, and trajectory tokens; modality-specific delimiter tokens identify sequence boundaries. Available instruction streams are concatenated, followed by target motion tokens, and standard causal cross-entropy trains the model to predict each next motion token. Sliding windows expose the model to varying positions within arbitrarily long music and trajectory inputs.

This “early fusion” lets one Transformer handle text-to-motion, music-to-dance, trajectory-to-gait, motion completion/tracking, and multimodal combinations. It also leverages language pretraining for sequential instructions instead of reducing text to a fixed command embedding. The cost is a three-billion-parameter server model and an autoregressive failure mode: errors can compound, diversity is below some diffusion baselines, and the model may prefer linguistically likely motion over physically appropriate motion.

### Causal decoding and streaming system

The motion decoder is causal: output at time `t` uses only current and prior token features through a one-sided temporal convolution. It decodes in small chunks to reduce per-token overhead. When a command changes, ten prior motion tokens are kept as context so the continuation begins from the current behavior rather than a fresh pose.

Generation runs on a GPU server and is sent over WebSocket to the robot client. Because the server produces at a variable, usually faster-than-real-time rate while control must be periodic, the client maintains a motion cache and reads one reference every 20 ms. This buffer is what turns bursty LLM generation/network transfer into a stable 50 Hz reference stream; it also creates a latency-versus-underrun tradeoff that should be included when interpreting “sub-500 ms.”

### RL tracking network and physical interface

The modified BeyondMimic tracker observes reference joint position/velocity, current joint position/velocity, body-frame gravity, body angular velocity, and previous action. Removing global reference orientation focuses the policy on relative joint configuration and avoids forcing an unreliable world-heading estimate. It outputs joint-position targets, converted to torque by PD control.

PPO uses asymmetric actor–critic training: the critic additionally sees relative Cartesian body poses. Actor and critic are ELU MLPs with hidden sizes `[2048, 1024, 512, 256, 128]`, trained with 16,384 environments. Physics runs at 200 Hz and the deployed policy at 50 Hz. Rewards track root orientation, relative body-link position/orientation, and link velocities; penalties cover action rate, joint limits, unwanted contacts, torque, and slip. Randomized friction, joint offsets, and center of mass support transfer.

No camera, LiDAR, or raw scene representation enters the published generator/tracker. “Multimodal” means instruction modality, not general environmental perception. A path is a supplied abstract trajectory; it is not planned from obstacles by UniAct.

### UA-Net data and evidence

UA-Net contains roughly 20 hours of robot-compatible text, trajectory, and music motion. Human motion is retargeted with GMR and manually refined; FineDance contributes music-conditioned examples. Its semantic annotation breadth is as important as duration because the LLM needs aligned vocabulary, rhythm, paths, and robot joints.

Evaluations separate instruction alignment (FID, diversity, retrieval/MM-distance, genre, root path error) from physical success. Ablations show that reducing the FSQ vocabulary or temporal resolution harms results, consistent with discrete capacity controlling both detail and robustness. Real experiments demonstrate complex sequential language, beat-synchronized dance, and path following. The authors also identify rapid jumping and object manipulation as current failures—the former is limited by the tracker, the latter by the absence of object-centric representations.

### Technical judgment and next work

The strongest contribution is the complete latency-aware pipeline. Shared tokens turn unlike conditions into a problem an autoregressive MLLM can solve; causal decoding, history carry-over, network streaming, and a fixed-rate client cache make that model usable for control. The discrete bottleneck also provides a useful safety bias by avoiding arbitrary continuous output, though it is not a safety guarantee.

Important open issues are quantization loss, autoregressive drift, server/network dependence, and lack of scene grounding. Ten-token context may not preserve long phase or intent across a command boundary. A language request can be semantically valid yet unsafe near a person, and neither the LLM nor tracker observes that person. Future work should add object/terrain tokens, explicit constraint decoding, token-level feasibility critics, cache-aware latency benchmarks, and onboard fallback behavior. Hierarchical tokens or a continuous corrective residual could preserve coarse semantic robustness while restoring precise contacts and fast dynamic motion.

## Unified Walking, Running, and Recovery for Humanoids via State-Dependent Adversarial Motion Priors

Source: [arXiv 2605.18611](https://arxiv.org/abs/2605.18611).

### Problem and central idea

This work trains one Unitree G1 actor for forward/backward walking, running, and recovery from prone or supine falls. Conventional deployments often switch between separately trained stand-up and locomotion policies using a hand-built finite-state machine. A naive single-policy AMP alternative has a different problem: one discriminator must treat cyclic upright gait and multi-contact get-up motion as one distribution, producing an ambiguous style reward.

The paper keeps one actor but makes its *training prior* state dependent. Fallen transitions are judged against recovery motion; upright transitions are judged against walking/running motion appropriate to commanded speed. At deployment, both discriminators and the gate disappear. The exported actor is one ONNX policy at 50 Hz, so the method does not merely hide a runtime controller switch inside another network.

### Exact policy input and action

One observation frame is 96-D: body angular velocity, projected gravity, commanded velocity, relative joint positions, joint velocities, and the previous action. Four consecutive frames are concatenated, giving the three-layer MLP actor a 384-D input. This short history helps infer movement phase and recent command/action evolution without a recurrent state. The 29-D action specifies desired G1 joint positions, realized by low-level PD control.

The controller is entirely proprioceptive. Projected gravity comes from orientation/state estimation; joint encoders and IMU provide the remaining feedback. There is no RGB camera, depth sensor, LiDAR map, contact map, or terrain encoder. It therefore demonstrates behavioral unification on the tested surface, not perceptive all-terrain recovery.

PPO optimizes velocity tracking, smoothness, upright/postural terms, energy, and fall penalties plus AMP reward with weight `lambda_AMP = 0.5`. Standard simulator parameter randomization supports zero-shot hardware deployment; no separate real-world fine-tuning is reported.

### Two discriminators and their gate

The recovery discriminator receives transitions from the retargeted `fallAndGetUp2_subject2` LAFAN1 clip. The locomotion discriminator is one velocity-conditioned network trained with `walk1_subject1` and `run1_subject2`. Its condition is normalized forward command

`v_hat = min(v_x_command / v_max, 1)`.

For discriminator reference updates, a walk transition is sampled with probability `1 - v_hat` and a run transition with probability `v_hat`. The condition therefore provides a continuous walk-to-run training distribution instead of a binary gait label or two locomotion discriminators.

The routing rule uses the vertical component of projected gravity. Upright is approximately `g_z = -1`; recovery reward is selected when `|g_z + 1| > 0.6`, corresponding to about 37 degrees of body tilt, and locomotion reward otherwise. The authors justify the fixed boundary from a low-density gap in the empirical gravity signal. Its simplicity makes the method interpretable and prevents another classifier from becoming a source of error.

Only the selected discriminator produces style reward for a training transition. That matters near recovery: an unconditional locomotion prior would punish hands/knees contacting the floor, while an unconditional recovery prior could reward non-cyclic or crouched behavior after the robot is upright.

### Results and what they establish

Only three LAFAN1 clips regularize all behavior. Hardware velocity commands cover `[-0.5, 1.0] m/s` in the normal range and `[-1.5, 3.0] m/s` in a fast operator-selected range. The same frozen policy recovers from both prone and supine configurations, then transitions through walking to running without a recovery mode command. Qualitatively, the velocity condition produces heel-toe walking and longer-stride running with reduced double support.

The result is notable because it shows that separating incompatible *reward distributions* can be enough; separate execution policies are not inevitable. It is also data efficient in the narrow sense of needing only three clips. That should not be interpreted as broad motion coverage: the structural prior is doing more work precisely because the target behavior set is compact.

### Technical judgment and next work

The paper is a clean, practical extension of AMP. It isolates where mode specialization is helpful—the learned style critic—while preserving a simple actor and deployment path. The continuous speed condition is preferable to a discrete walk/run switch, and the gravity gate is easy to audit.

The gate can nevertheless chatter around the threshold because no hysteresis or temporal filtering is described. A crouched but intentional locomotion pose may be routed as recovery, while an upright early recovery state may receive gait reward too soon. Projected gravity alone cannot distinguish sitting, kneeling, bracing against a wall, or falling on stairs. The reference set also provides little coverage of lateral gait, turning, variable get-up contacts, or terrain, and “fast mode” remains an explicit operator safety decision even though gait execution is unified.

A next version should study soft probabilistic routing or hysteresis, learn the gate under interpretability constraints, and add contact/terrain context. More recovery examples should cover obstacles and asymmetric falls. Evaluation should report threshold sensitivity, repeated fall/recovery cycles, impacts and motor limits, and transitions near the gate—not only successful showcase sequences. A particularly useful test would compare one actor with routed priors against a genuinely matched multi-controller FSM on reliability, memory, latency, and edge cases.

## Real-Time Execution of Action Chunking Flow Policies

Real-Time Chunking (RTC) is an execution algorithm, not a learned policy. Standard action-chunk agents either block the robot during inference or keep executing an old chunk and then jump to a newly predicted one. RTC runs inference asynchronously while the 50 Hz control loop continues.

When a new observation arrives, RTC estimates how many actions will inevitably execute before inference finishes. Those prefix actions are held fixed. A flow/diffusion sampler inpaints only the remaining suffix, conditioned on the frozen prefix, so the new chunk is dynamically continuous with what the robot actually did. The method uses the existing conditional flow vector field and requires no retraining.

Experiments include an `H=8` chunk policy implemented with a four-layer MLP-Mixer and simulated delays up to four controller steps, plus dynamic Kinetix and real bimanual manipulation. A producer–consumer implementation separates the inference thread from the control loop and atomically replaces future actions when ready. Results show smoother action transitions and robustness even above 300 ms compared with naive chunk replacement.

RTC matters disproportionately for humanoids because pausing, duplicating, or discontinuously switching commands can destabilize balance. It does not change observations, proprioception, or the base model's semantics; it only reconciles timing. If latency exceeds the remaining horizon, if the old prefix is already unsafe, or if the policy proposes a semantically wrong suffix, inpainting cannot solve the problem. The method also assumes the action representation is suitable for conditioning/inpainting.

### The systems insight behind RTC

Action latency is often treated as a throughput statistic, but RTC shows that it changes the semantics of a predicted chunk. By the time inference ends, the robot is no longer at the observation state from which the chunk was generated. Naively beginning at action zero repeats stale commands; jumping forward can introduce a discontinuity because the new policy did not condition on the exact actions executed during inference. Freezing the inevitable prefix makes generation consistent with the real timeline.

This is especially elegant because the fix operates inside the generative sampler and requires no new demonstrations. It converts known future controls into an inpainting condition, using a capability flow/diffusion models already possess. The separation of inference and control threads also lets the actuator loop retain deterministic timing even when GPU latency varies.

RTC is not a safety controller. It assumes latency can be predicted well enough to choose the fixed prefix, and it preserves that prefix even if a new observation reveals imminent danger. Very long or highly variable latency can consume the whole horizon. Moreover, smooth continuity in joint space does not imply continuity of contact force, center of pressure, or task intent.

### Useful extensions

An adaptive version should model a distribution over completion time and reserve a conservative prefix under jitter. A safety monitor could override frozen actions when state constraints or collision risk are violated, while a fallback stabilizer bridges until the next valid chunk. Humanoid evaluation should report falls, foot slip, contact impulse, and support-polygon margin—not only action smoothness and task success. Conditioning on predicted force or contact mode could make suffix inpainting physically continuous as well as numerically continuous. Finally, dynamically choosing chunk horizon based on latency and environmental uncertainty could avoid committing equally long into both free-space and contact-rich phases.

### Assumptions an implementation must satisfy

RTC needs synchronized timestamps, a reliable estimate of which actions have or will execute, and a sampler that can condition a suffix on fixed prefix values. Queueing or network delay must be included, not just neural inference time. If the actuator consumes commands faster or slower than assumed, the inpainted boundary is still misaligned. Atomic replacement is needed so the control thread never reads a partially updated chunk.

There is also a representation assumption. Inpainting joint positions is straightforward numerically, while inpainting actions with hidden controller state or contact-mode semantics may not be. The fixed prefix must be expressed in precisely the same normalized action coordinates used during training. For hierarchical humanoids, RTC may be better applied to task-space motion chunks while a stabilizer owns high-frequency joints.

A strong systems evaluation would replay measured latency traces with jitter, GPU contention, and dropped observations rather than fixed artificial delays. Comparing wall-clock throughput, deadline miss rate, discontinuity, and physical outcomes would show when RTC's extra sampler constraints are worthwhile. The method is simple conceptually, but correctness depends on careful control-software integration.

### Comparative position

RTC is orthogonal to almost every learning contribution here. It can wrap π0 chunks, HuMI diffusion, SUGAR commands, or tactile pose plans if the sampler supports prefix conditioning. It does not improve perception or motor competence, but prevents failures caused by disagreement between wall-clock inference and control time. Every chunked humanoid policy should compare blocking execution, naive asynchronous replacement, temporal ensembling, and RTC under the same measured latency trace.
