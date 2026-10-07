# Whole-Body Control — Paper Notes

Whole-body control is not one uniform task: this folder includes actuator control, motion tracking, compliance, safety filters, teleoperation interfaces, foundation models, and contact-rich loco-manipulation. Each note is shaped around the particular control problem and evaluates the paper on its own claim. Details such as data, networks, perception, or command spaces appear only when they clarify that claim. Cross-listed papers have fuller task-specific discussion in their other tagged folders.

## Actuator Control for the NASA-JSC Valkyrie Humanoid Robot A Decoupled Dynamics Approach for Torque Control of Series Elastic Robots

Develops torque control for Valkyrie’s series-elastic actuators by decoupling motor, spring, and link-side dynamics. Feedforward/feedback terms compensate actuator effects so a higher-level whole-body controller receives a more linear, responsive torque interface. The method improves tracking and disturbance response, but depends on parameter identification and reliable deflection sensing.

### Decentralized torque-source abstraction

Valkyrie's whole-body controller models rigid bodies actuated by ideal joint torques and outputs a desired torque per joint. Each embedded actuator controller then owns motor, gear, spring, sensing, friction, and load-side dynamics and converts that request to motor current. This hierarchical boundary is the paper's central engineering choice: higher-level inverse dynamics does not carry a coupled model of every series-elastic actuator, communication bandwidth is reduced, and one joint can be tuned before assembling the full humanoid.

The joint loop estimates spring torque from deflection and uses feedback/feedforward plus a disturbance observer (DOB) to reject unmodeled load and actuator effects. A frequency-domain analysis treats variable reflected load inertia explicitly, showing which disturbances the DOB attenuates and where sensor noise, phase margin, and saturation limit bandwidth. Inputs are desired joint torque and local motor/link/spring measurements; output is motor current realizing an approximately ideal torque source.

The highlight is a clean abstraction that made many-DoF SEA control practical. Passive compliance improves shock tolerance and force sensing, while decentralization aids debugging. Yet decoupling is approximate: multibody coupling appears as disturbance, and aggressive DOB gains amplify noise or flexible modes. Online friction/load identification, explicit thermal/saturation protection, delay analysis, and whole-body passivity monitoring are valuable extensions.

### Loop design, identification, and whole-body consequences

Series-elastic torque is inferred from spring deflection, so encoder resolution, spring stiffness calibration, backlash, and structural vibration directly set usable feedback bandwidth. The motor-side plant contains motor inertia, gearing, friction, current-loop dynamics, and the elastic element; the link-side load changes with posture and contact. The decoupled controller treats much of this variation as a disturbance and combines model feedforward with torque feedback and a DOB. A low-pass filter inside the observer creates the familiar trade: increasing bandwidth rejects friction and load disturbance but admits sensor noise and flexible resonances.

The important experimental quantities are closed-loop torque bandwidth, phase margin, steady and transient tracking, apparent output impedance, disturbance rejection, and response under changing reflected inertia. Locked-output testing characterizes the actuator, while free-link and whole-body contact tests show interaction stability. Saturation must be included because a high-gain observer can request unavailable current and then invalidate its linear analysis. Thermal limits, communication delay, quantization, and spring hysteresis are equally important on Valkyrie's large joints.

At the robot level, better joint torque tracking improves inverse-dynamics consistency, but independent loops do not remove cross-joint coupling. A foot impact can excite structural modes shared across the limb, and an ideal-torque whole-body model may command energy that local controllers cannot safely deliver. A useful successor would expose torque capacity, temperature, bandwidth, and uncertainty per joint to the whole-body optimizer. Passivity or energy tanks could bound interaction even when the DOB model is wrong. Online identification should update friction and effective inertia slowly, with a certified fallback to conservative gains.

### Reproducibility, stability margins, and whole-body integration

For Actuator Control for the NASA-JSC Valkyrie Humanoid Robot A Decoupled Dynamics Approach for Torque Control of Series Elastic Robots, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

A reproducible actuator result must state motor and load inertia, gear ratio, spring or transmission stiffness, torque-sensor location and resolution, current-loop bandwidth, servo rate, filters, delay, saturation, friction compensation, and controller gains. Reference tracking should include swept-sine bandwidth and phase, steps and reversals; interaction tests should include several reflected inertias and environment stiffnesses, impacts, and sustained load. RMS error alone hides noise amplification, active output impedance, current peaks, heating, backlash, and energy injection. Frequency-domain prediction should be checked against measured Bode plots and time-domain contacts, with raw traces and thermal state disclosed.

Whole-body consequences need a second level of evaluation. The higher controller assumes a torque source, but every joint has different bandwidth and capacity and multibody contacts couple otherwise decentralized loops. Report joint torque residuals during walking, impacts, manipulation and saturation, and show how they affect task-space tracking, balance, contact wrench and passivity. A policy trained with ideal actuation should be tested with the measured closed-loop actuator model, delay and limits. The actuator layer should expose available torque-speed envelope, temperature derating, estimated error and fault status to whole-body control.

My assessment criterion is not which method wins one nominal trace, but which maintains an explicit stability and energy margin over uncertain load, contact and wear. Online identification or learned residuals can improve compensation, yet must be bounded and fall back to a conservative verified loop. Passivity observers, energy tanks, torque-rate limiting, anti-windup and fault detection belong in the evaluation because they determine whether improved bandwidth remains safe around people and hardware.

## Agility Meets Stability: Versatile Humanoid Control with Heterogeneous Data

Combines heterogeneous motion sources—mocap, retargeted clips, optimized trajectories, and task-generated behavior—to train one versatile humanoid policy. Balanced sampling and difficulty-aware objectives prevent easy data from dominating while robustness training preserves stability. It expands skill breadth, though conflicting data quality and embodiment feasibility remain central curation problems.

### Synthetic balance data and hybrid PPO objective

AMS observes proprioception plus a reference motion state and outputs 23 desired joint angles. A privileged PPO teacher is trained before deployment-oriented student distillation. Ordinary mocap supplies dynamic human motions but under-represents poses that exploit robot-specific support limits, so a controllable generator synthesizes balance-critical targets. General tracking/stability rewards apply across all data, while balance priors are applied only to synthetic samples; imposing those priors on natural agile motion would suppress the dynamics the method is trying to preserve.

Adaptive learning mines low-success motions and changes sampling/reward pressure per sequence, preventing easy walking clips from dominating rare balance and agility. One policy consequently performs dynamic skills and extreme quasi-static balance on hardware rather than switching controllers. The key insight is that heterogeneous data require heterogeneous supervision, not merely concatenation.

Synthetic balance can encode the simulator's support/contact assumptions, and per-motion shaping risks a growing set of heuristics. Success also says little about torque/impact margin. Further work should learn data provenance/quality weights, calibrate contact uncertainty, report energy and fall/intervention rates, and test smooth interpolation between agile and balance regimes rather than only replaying represented targets.

### Dataset provenance, network interface, and decisive ablations

AMS's heterogeneous bank should be viewed as several conditional distributions, not one shuffled replay buffer. Natural mocap contains expressive transient dynamics and human morphology artifacts; retargeted data adds robot joint-limit/contact error; optimized trajectories are feasible but often stylistically narrow; synthetic balance poses deliberately stress the support polygon. Provenance-aware rewards let natural data preserve agility while generated data receives balance-specific supervision. Adaptive sampling increases probability for clips whose success or tracking return remains low.

The deployable actor takes proprioception—root attitude/angular motion, 23 joint positions and velocities, previous action—plus reference state and outputs 23 desired joint angles for PD execution. A privileged teacher/critic can use exact base velocity, body/contact state, and dynamics parameters; student distillation removes unavailable signals and uses history to infer them. Controller rate, reference look-ahead, and history length determine whether rapid motion is anticipated or merely reacted to and should be reported alongside architecture dimensions.

The central ablation is factorial: homogeneous versus mixed data, shared versus provenance-conditioned reward, uniform versus adaptive sampling, and teacher versus student observations. Per-family success and error are necessary because one average can improve by sacrificing rare balance or aerial clips. Hardware evaluation should include torque/temperature, fall and intervention, contact impulse, and transitions between data families. The paper's strongest idea is conditional supervision for conditional data; the next step is learning quality/reward weights from feasibility and uncertainty instead of assigning them manually, with a policy that can reject an infeasible reference rather than producing a damaging approximation.

### Control-stack accounting and a stronger evaluation protocol

The decisive chain is data provenance → provenance-specific objectives → adaptive hard-motion sampling → teacher/student deployment. Its claim depends on retaining both dynamic agility and quasi-static balance rather than improving an average dominated by easy clips.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

## Any2Any: Efficient Cross-Embodiment Transfer for Humanoid Whole-Body Tracking

Any2Any seeks one tracking framework that transfers motions between different human sources and robot bodies without training every source–target pair. It learns embodiment-aware motion representations and separates common task intent from morphology-specific decoding/control. A privileged teacher supplies dynamically feasible behavior; deployable students use robot proprioception and reference features. The value is reduced retraining and broader reuse of motion corpora. Transfer still depends on compatible contact semantics, joint/workspace coverage, and good source retargeting; morphology abstraction cannot create capabilities the target robot physically lacks.

### What is actually transferred

The starting point is a complete PPO whole-body tracker already trained for a source humanoid. Its actor consumes proprioception—base angular velocity, projected gravity, joint positions/velocities, and previous action—together with reference-motion features, then outputs offsets to desired joint positions. An asymmetric critic may use base velocity, contact force, body parameters, and other simulation-only state. Any2Any freezes most of this source policy and adapts it to a target with a different number/order of joints and different dynamics.

The method explicitly separates two mismatches. **Kinematic alignment** makes target observations and actions semantically compatible with the frozen source network. First it reorders global observation blocks. At joint level, a sparse scattering matrix inserts target joints into the source layout, discards unmatched redundant coordinates, and zero-pads missing ones. A special hip-decoupling transform corrects inclined/coupled hip axes before source-policy inference; the inverse mapping gathers source actions back into target order. This preserves the meaning of learned features instead of asking fine-tuning to discover a joint permutation through closed-loop trial and error.

**Dynamics adaptation** then uses parameter-efficient fine-tuning. The paper tests MLP, causal Transformer, and SONIC-style backbones and compares LoRA, bottleneck adapters, and prefix tuning. LoRA is inserted not only into the actor backbone but also into proprioceptive input and action-output projections, because morphology changes both state encoding and action decoding. Only about 5.26% of parameters are trainable in the highlighted setting. It reaches performance comparable to aligned full fine-tuning while increasing reported training throughput from 34.8k to 111.7k FPS and using much less adaptation compute/data.

### Evidence and judgment

Experiments transfer between Unitree H1 and LimX Oli/Luna-like embodiments and vary dataset scale, GPU sampling rate, backbone, alignment, and adaptation location. The important ablation is that LoRA without kinematic alignment is not enough: semantic correspondence must be solved before low-rank weight changes can compensate dynamics. Prefix tuning is least stable, plausibly because adding context tokens does not directly repair proprioceptive/action interfaces.

This is a pragmatic alternative to training a universal multi-robot foundation policy from scratch. Its scope is narrower than the name suggests: the robots still share a recognizable humanoid topology, and correspondences plus hip transforms are engineered. Missing limbs, radically different hands, series-elastic actuation, or different contact surfaces may require new mappings. Zero padding can also make “absent joint” indistinguishable from a real zero value unless masks are exposed.

Future work should learn correspondence from geometry while keeping it auditable, attach explicit joint-presence masks, and adapt actuator/contact models separately from morphology. Evaluation should include torque/impact and failure recovery, not only pose reward, and test whether one adapted policy preserves rare source skills. A useful safety layer would reject references outside the target robot's reachable/contact envelope before the transferred tracker tries to execute them.


### Backbone-independent alignment and adaptation failure modes

The paper verifies the same transfer logic on three rather different source policies: a flattened-history MLP, a causal Transformer whose modality tokens share an embedding space, and a SONIC-like encoder with a finite-scalar-quantized motion bottleneck. That is valuable because kinematic alignment occurs before the learned backbone: global observation blocks are permuted, joint blocks are scattered into the source layout, actions are gathered back, and hip axes are transformed in both directions. The adaptation method therefore does not depend on attention or a particular latent representation.

LoRA placement is unusually important here. Updating only internal linear layers assumes the original input and output maps remain meaningful, which is precisely what changes across bodies. Adaptation on proprioceptive projections and action heads, in addition to policy blocks, gives low-rank parameters direct leverage over sensor scaling and actuator semantics. The critic remains privileged during PPO adaptation, using base velocity, contacts, mass distribution, and other simulation variables to improve the value target while the actor stays deployable.

Two transfer errors should be reported separately. **Structural error** comes from an incorrect joint correspondence or discarded DoF and cannot be repaired reliably by more PPO. **Dynamic error** comes from mass, inertia, actuator bandwidth, foot shape, or latency and is the appropriate target for PEFT. A useful extension would estimate both with held-out diagnostic motions, refuse transfer when structural residual is too high, and compare the adapted policy against training a target specialist at equal environment steps—not merely equal wall time.

## Athena-WBC: Capability-Aligned Policy Experts for Long-Tail Humanoid Whole-Body Control

[Athena-WBC](https://arxiv.org/abs/2607.04837) assigns stubborn long-tail clips to capability-aligned teachers: dynamic experts relax conservative effort/temporal penalties while retaining feasibility constraints, and balance experts use a gravity curriculum. Motion-routed DAgger distills them into one deployable policy followed by RL refinement. It recovers difficult transitions efficiently, though expert design still encodes manual capability categories.

### Residual-motion diagnosis and capability experts

Athena begins after a strong general tracker has saturated: remaining failed clips are diagnosed by motion salience and failure regime rather than simply oversampled. Dynamic experts retain tracking and physical-limit penalties but remove effort/action-rate rewards that prevent large necessary accelerations; smoothness is supplied through auxiliary temporal/spatial policy regularization. Balance experts progressively alter gravity/support difficulty so the policy actually visits recoverable states instead of terminating immediately.

Experts use privileged simulation observations and PPO. A router labels their motion domains, then DAgger collects the unified student's own states and queries the appropriate teacher for actions. The deployable student consumes proprioception/reference features and outputs joint targets; final RL fine-tuning repairs compression error. This is an important distinction from a mixture-of-experts at runtime: expertise is an acquisition tool and one controller remains after training.

The highlight is recognizing that some long-tail failures are optimization/capability failures, not data-frequency failures. But hand-defined “dynamic” and “balance” categories may not scale to contact-rich or perceptive skills, and relaxed effort penalties may create damaging solutions. Automated failure clustering, constraint-aware expert synthesis, teacher-disagreement uncertainty, and hardware-aware torque/impact regularization should extend the approach.

### Expert allocation, router supervision, and long-tail diagnostics

Athena's expert design begins with failure attribution. A clip that fails because effort penalties suppress the required acceleration needs a different intervention from a clip that fails because narrow support prevents exploration. Dynamic experts relax torque/action-rate or phase rigidity while keeping joint, contact, and feasibility constraints; balance experts use gravity/support curricula to make precarious states learnable. The expert is trained with privileged PPO, then labels the unified student's visited states through motion-routed DAgger. Final RL refinement closes residual imitation error.

Routing at collection time should use clip and phase, since a nominally dynamic motion can include a static balance phase and vice versa. Teacher disagreement is valuable uncertainty: if experts prescribe incompatible actions on the same state, blindly distilling their average may be worse than either. Reporting router confusion, teacher advantage, student–teacher error by phase, and how many clips each expert recovers would establish that “capability aligned” is more than naming two training recipes.

Long-tail evaluation must preserve the head. Publish success/error before and after expert acquisition for every motion family, action and torque distribution shift, fall/impact rate, and student regression on ordinary walking. Compare against extra generalist PPO updates, failure oversampling, a runtime MoE, and specialist deployment at equal environment steps. Manual expert categories will eventually proliferate for contacts, perception, payload, or recovery. Automatic clustering of failure signatures—reward components, termination state, required acceleration, support geometry—could propose curricula and reward changes. A feasibility critic should also gate hardware attempts when expert success depends on simulator-only margins.

### Control-stack accounting and a stronger evaluation protocol

The decisive chain is failure diagnosis → capability-specific expert training → motion-routed DAgger → generalist RL repair. Evidence should show which long-tail failure each expert fixes and what head-skill performance is preserved.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

## Behavior Foundation Model for Humanoid Robots

Learns a reusable latent behavior prior from a large motion collection, then conditions or fine-tunes it for tracking and downstream control. The foundation representation reduces per-task training and can interpolate among skills. Its value comes from scale and reuse; physical executability, data imbalance, and opaque latent commands can limit zero-shot deployment.

### Proxy behavior data and conditional VAE policy

The paper's abstraction is that different control modes ultimately generate trajectories of proprioceptive states and actions. A privileged PPO proxy tracker converts observation-only human motions into physically executable robot rollouts. Its actor uses robot proprioception and privileged reference goal; successful transitions provide a common dataset independent of whether a later user will request motion tracking, locomotion, or another goal form.

A conditional VAE models the action distribution given deployable proprioception and latent behavior. Observable state includes base angular velocity, projected gravity, joint positions/velocities, and previous action (with stacked history); the decoder outputs target joint angles. During training, an encoder can infer latent behavior from future/goal information, while a learned prior permits deployment and interpolation. The latent becomes a proxy action space for downstream control.

The central idea—standardize around behavior transitions rather than incompatible command schemas—is strong and makes reuse plausible. Yet the BFM can only cover behaviors reached by the proxy and motion corpus; CVAE likelihood may blur rare, multi-modal contact strategies, and latent directions lack guaranteed semantics or safety. Coverage/uncertainty diagnostics, discrete or sequence latents for multimodal contact, and downstream tests outside tracking are needed to establish foundation-model rather than broad-motion-prior status.

### Pretraining objective, latent semantics, and downstream control

The proxy tracker is a crucial but easily overlooked component: it turns heterogeneous motion observations into synchronized robot state–action trajectories. Those rollouts include balance corrections and actuator commands absent from human pose data, letting the conditional VAE model a physical behavior distribution. The encoder sees behavior-defining future or goal information during training; the decoder receives latent plus deployable state history and emits desired joints. A prior predicts useful latent distributions when privileged future information is absent.

Latent capacity determines whether the model memorizes clips, averages incompatible contacts, or learns reusable modes. KL weight, temporal horizon, history stacking, latent dimension, decoder stochasticity, and sampling strategy should be reported. Reconstruction/action likelihood is not enough: decoded sequences need closed-loop rollout success, contact correctness, diversity, and long-horizon stability. Interpolation should be evaluated for physical meaning rather than visually smooth poses; the midpoint between jump and crouch may be neither safe nor useful.

For downstream tasks, a planner or lightweight policy operates in latent space while the BFM supplies motor detail. This can reduce exploration dimensionality, but the latent must retain commands relevant to the task and expose coverage uncertainty. Comparisons should hold environment interactions constant against training from scratch, motion priors, and direct joint-action RL. The “foundation” claim becomes credible when the same frozen decoder improves several distinct interfaces—tracking, velocity, sparse keypoints, contact recovery, or manipulation—not only variants of imitation. Safety requires a latent feasibility envelope and an action-level filter, because a high-likelihood behavior can still be inappropriate near people or obstacles.

### Foundation-model criteria, latent coverage, and deployment accounting

The decisive question is whether the learned latent represents reusable physical behavior rather than clip identity. Cross-interface transfer and closed-loop decoding are stronger tests than reconstruction.

A behavioral foundation model should document the source motion hours and filtering, how kinematic observations become physical robot rollouts, replay or on-policy transition count, network and latent dimensions, temporal horizon/history, privileged training inputs, deployable inputs, action representation, control rate, and total simulation compute. Model size alone is not scale when larger policies receive more environment interaction. Data and compute-normalized curves should vary capacity, behavioral diversity and optimization separately, with several seeds and per-family results.

Latent quality requires closed-loop tests. Reconstruction or action likelihood can be high while decoded contacts drift and the robot falls. Report rollout survival, pose and velocity error, contact timing, diversity, latent utilization or collapse, interpolation behavior, prompt sensitivity, inference latency and hardware reliability. New goals, demonstrations or rewards should be structurally held out, not merely different clips from the same corpus. Compare frozen latent control, weight fine-tuning, task-specific RL, retrieval and random/optimized latent search at equal downstream interactions. A coverage estimator should predict when a requested behavior lies outside the pretrained occupancy.

The practical control stack also needs an authority boundary. A high-level prompt or planner selects latent behavior; the decoder produces joint targets; servo and safety layers constrain execution. Prompt computation, denoising or optimization steps count toward latency, and stochastic samples need temporal consistency. “Zero shot” must state what remains optimized after prompting. My judgment is that foundation status is earned by reusable improvement across different command interfaces and tasks with predictable failure, not by one very broad tracker. Cross-embodiment transfer, uncertainty-based abstention, contact/terrain perception and shielded latent planning are the decisive next tests.

## BFM-Zero: A Promptable Behavioral Foundation Model for Humanoid Control Using Unsupervised Reinforcement Learning

Uses unsupervised RL to discover a broad repertoire and exposes behavior through prompts or goal descriptors rather than fixed task labels. A downstream selector can request skills without task-specific demonstrations. The approach improves exploratory breadth, but discovered behaviors need not align cleanly with human intent and still require safety/feasibility filtering.

### Forward–backward latent prompts and off-policy pretraining

BFM-Zero learns forward–backward representations in reward-free/off-policy simulation. A latent-conditioned policy, critic, forward successor feature, and backward state embedding share a space in which a downstream reward can be projected to a latent vector. Motion-capture states regularize this space, while auxiliary safety/motion rewards, domain randomization, and asymmetric history-dependent training make the resulting joint-target policy deployable.

At inference, several prompt types map into the same latent: demonstration/tracking prompts, goal states, or explicit reward samples. The policy can therefore perform tracking, pose reaching, and reward optimization without gradient retraining. When zero-shot quality is insufficient, optimization searches the low-dimensional latent for a few episodes rather than re-learning millions of policy weights. This is genuine unsupervised RL rather than PPO on a fixed tracking reward.

The attractive feature is objective-centric promptability: the latent is connected to successor occupancy and reward, making it more interpretable than an arbitrary skill code. Off-policy replay also reuses experience. However, projected rewards only work within learned coverage, mocap regularization biases exploration, and few-shot real interaction can be unsafe. Calibrated coverage tests, shielded latent search, semantic prompt grounding, and comparisons at equal environment transitions against on-policy BFMs are important.

### Successor features, prompt construction, and coverage limits

Forward–backward representation learning factorizes discounted state occupancy: the forward embedding depends on state, action, and latent-conditioned policy behavior, while the backward embedding describes reached states. Their inner product approximates successor occupancy, allowing a downstream reward expressed over states to be integrated into a latent prompt. Off-policy replay makes this representation sample-efficient and reward-free relative to any one task. Mocap-state regularization biases the discovered repertoire toward humanoid-looking behavior and auxiliary constraints keep exploration from degenerating into falls or joint-limit exploitation.

Prompt construction deserves separate evaluation. A demonstration prompt estimates the latent whose occupancy resembles demonstrated states; a goal prompt emphasizes a desired terminal region; reward samples project an explicit objective into the learned basis. These are not equivalent information sources. Report prompt computation, number of samples, latent normalization/search, inference rate, and whether prompts remain fixed per episode or adapt online. Few-shot latent optimization should count real/sim interactions and compare with policy fine-tuning at the same budget.

Zero-shot success is limited by span: if a reward or goal requires states outside pretraining occupancy, no linear projection can create the missing behavior. Coverage tests should estimate density or reconstruction for requested states and abstain when low. Motion regularization also means “unsupervised” is not behaviorally unconstrained; its benefit should be ablated. Hardware latent search needs a shield, conservative simulator evaluation, and uncertainty, since reward maximization can exploit model error. The most convincing evidence would include novel task objectives composed after pretraining, systematic prompt sensitivity, and demonstrations that fail predictably when outside learned support.

### Foundation-model criteria, latent coverage, and deployment accounting

The decisive question is whether reward-free successor representations cover downstream state occupancies and whether prompt projection selects them without unsafe latent search.

A behavioral foundation model should document the source motion hours and filtering, how kinematic observations become physical robot rollouts, replay or on-policy transition count, network and latent dimensions, temporal horizon/history, privileged training inputs, deployable inputs, action representation, control rate, and total simulation compute. Model size alone is not scale when larger policies receive more environment interaction. Data and compute-normalized curves should vary capacity, behavioral diversity and optimization separately, with several seeds and per-family results.

Latent quality requires closed-loop tests. Reconstruction or action likelihood can be high while decoded contacts drift and the robot falls. Report rollout survival, pose and velocity error, contact timing, diversity, latent utilization or collapse, interpolation behavior, prompt sensitivity, inference latency and hardware reliability. New goals, demonstrations or rewards should be structurally held out, not merely different clips from the same corpus. Compare frozen latent control, weight fine-tuning, task-specific RL, retrieval and random/optimized latent search at equal downstream interactions. A coverage estimator should predict when a requested behavior lies outside the pretrained occupancy.

The practical control stack also needs an authority boundary. A high-level prompt or planner selects latent behavior; the decoder produces joint targets; servo and safety layers constrain execution. Prompt computation, denoising or optimization steps count toward latency, and stochastic samples need temporal consistency. “Zero shot” must state what remains optimized after prompting. My judgment is that foundation status is earned by reusable improvement across different command interfaces and tasks with predictable failure, not by one very broad tracker. Cross-embodiment transfer, uncertainty-based abstention, contact/terrain perception and shielded latent planning are the decisive next tests.

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

## BFMTrack: Latent Sequence Optimization for Physics-Based Motion Tracking with Behavioral Foundation Models

BFMTrack uses a pretrained behavioral foundation model as the motion/control prior and optimizes a compact latent sequence for a desired reference. Searching in learned behavior space produces dynamically coherent trajectories more readily than frame-wise joint fitting, after which a low-level physics policy executes them. The method is useful for difficult or imperfect references. Its ceiling is the foundation model's behavior manifold, and latent optimization adds offline/test-time computation and may smooth away important contact detail.

### Why a latent *sequence* is needed

Behavioral foundation models (BFMs) are trained without task rewards to represent a family of policies indexed by a reward/goal latent. Standard zero-shot use summarizes a downstream objective into one latent vector. That works for time-invariant goals such as “reach this state” but poorly represents a backflip or dance, where the desired state and contact pattern change continuously. BFMTrack therefore optimizes a time-indexed latent sequence while keeping the pretrained BFM policy and successor-feature/value machinery frozen.

For a candidate latent schedule, the frozen policy rolls the character through physics. The objective compares simulated states with the reference in global and root-relative coordinates and optimizes the latent variables through the learned BFM structure. Temporal regularization discourages violent changes between neighboring latents. Dense reference frames and sparse keyframes use the same formulation; missing times simply contribute no tracking term, so the behavior prior fills the interval.

At execution, the policy input is current physical state plus the active optimized latent, and its output is the BFM's action for the simulated character. Thus BFMTrack does not train a new tracking actor or manually decompose pose, velocity, contact, and balance rewards for each clip. Search occurs in a space already associated with dynamically successful behaviors instead of raw joint trajectories.

### Strength, evidence, and limitations

The paper's useful conceptual result is that temporally varying prompting extends a reward-free foundation policy into precise sequence tracking. It handles highly dynamic/contact-rich motions and under-specified keyframes that a single latent tends to average. Global tracking matters: a solution that matches local pose while drifting across the floor is not accepted.

The method is nevertheless an optimizer around a fixed repertoire. If the pretrained interaction data omitted aerial rotation or a required contact, no latent schedule can manufacture it. Latent smoothness can erase sharp impact timing; weak smoothness can produce switching artifacts. Optimization also introduces per-reference computation, sensitivity to initialization/horizon, and no guarantee of the global optimum. It is closer to trajectory fitting/planning than an instantly generalizing tracker.

A strong next step would amortize sequence optimization with a reference encoder and retain a few test-time refinement iterations; expose uncertainty when no latent fits; and jointly refine contact timing with the latent. Cross-embodiment and real-robot tests would reveal whether the BFM manifold encodes robust physics or simulator-specific shortcuts. Comparisons should equalize test-time compute against diffusion/MPC baselines and report feasibility, not only kinematic error.


### Optimization mechanics and sparse-keyframe prior

For each reference frame, the backward map initializes a mean latent `μ_t = B(g_t)` on the unit hypersphere. LSO defines a Gaussian around every mean, samples 128 full latent sequences in parallel simulation, and scores realized state against the target using cosine similarity between backward embeddings. A leave-one-out return baseline reduces REINFORCE variance; Adam updates every timestep mean, which is re-normalized onto the sphere. The reported default uses 24 optimization iterations, pink temporally correlated exploration noise (`1/f`, rather than independent white noise), discount 0.97, noise standard deviation 0.0125, and a 256-D latent.

Pink noise is a substantive design decision. Independent per-frame samples produce high-frequency latent switching that the physical policy smooths away, leaving similar rollouts and a weak gradient. Correlated samples explore coherent behavioral alternatives such as an earlier crouch or longer step. In sparse tracking, only keyframes receive reward; keyframe latents are initialized from `B(g)` and intermediate means with spherical interpolation, after which the same correlated sequence optimization fills motion between them.

The method trains the underlying BFM for roughly three million gradient steps with 1,024 parallel environments and an off-policy update-to-data ratio of 1/64. A complete trajectory then takes minutes to optimize on a high-end GPU, so current use is offline reference preparation, not reactive teleoperation. Runtime reporting should include trajectory length and parallel rollout count, and future amortization should be judged by how many refinement rollouts it saves without collapsing the multiple valid transitions between sparse keyframes.

### Foundation-model criteria, latent coverage, and deployment accounting

For BFMTrack: Latent Sequence Optimization for Physics-Based Motion Tracking with Behavioral Foundation Models, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

A behavioral foundation model should document the source motion hours and filtering, how kinematic observations become physical robot rollouts, replay or on-policy transition count, network and latent dimensions, temporal horizon/history, privileged training inputs, deployable inputs, action representation, control rate, and total simulation compute. Model size alone is not scale when larger policies receive more environment interaction. Data and compute-normalized curves should vary capacity, behavioral diversity and optimization separately, with several seeds and per-family results.

Latent quality requires closed-loop tests. Reconstruction or action likelihood can be high while decoded contacts drift and the robot falls. Report rollout survival, pose and velocity error, contact timing, diversity, latent utilization or collapse, interpolation behavior, prompt sensitivity, inference latency and hardware reliability. New goals, demonstrations or rewards should be structurally held out, not merely different clips from the same corpus. Compare frozen latent control, weight fine-tuning, task-specific RL, retrieval and random/optimized latent search at equal downstream interactions. A coverage estimator should predict when a requested behavior lies outside the pretrained occupancy.

The practical control stack also needs an authority boundary. A high-level prompt or planner selects latent behavior; the decoder produces joint targets; servo and safety layers constrain execution. Prompt computation, denoising or optimization steps count toward latency, and stochastic samples need temporal consistency. “Zero shot” must state what remains optimized after prompting. My judgment is that foundation status is earned by reusable improvement across different command interfaces and tasks with predictable failure, not by one very broad tracker. Cross-embodiment transfer, uncertainty-based abstention, contact/terrain perception and shielded latent planning are the decisive next tests.

## CHIP: Adaptive Compliance for Humanoid Control through Hindsight Perturbation

Trains adaptive compliance by relabeling or learning from perturbations after they occur, turning disturbance experience into useful supervision. The controller modulates stiffness/response across contacts instead of remaining uniformly rigid. It improves safe interaction and robustness, though compliance estimates are learned and may not generalize to impacts outside the training range.

### Hindsight perturbation as dense compliance supervision

CHIP keeps an agile motion-tracking framework and changes how a perturbed rollout is interpreted. After an external force displaces an end effector, the resulting motion is treated in hindsight as the desired compliant response: goals/observations are relabeled to encode the perturbation while the original clean reference remains available for dense tracking reward. This avoids generating full compliant motion datasets or balancing a new force reward against every reference.

A generalist PPO tracker receives proprioception, reference state, and perturbed end-effector goals and emits joint targets; at deployment the requested goal offset effectively controls adaptive compliance. A three-point head/two-hand interface supports cooperative carrying, wiping, forceful teleoperation, and VLA-driven interaction while preserving unperturbed dynamic skills. The plug-in nature is the highlight: compliance becomes an alternative interpretation of tracking error, not a separately engineered controller.

The learned response is not guaranteed impedance. Relabeled perturbations are only credible inside the dynamically reachable training distribution, and force/energy limits are implicit. Hardware force sensing, passivity/CBF filtering, calibrated force-command response, multi-contact relabeling, and tests under unseen impulse direction/duration would make the “compliance” claim safer and more quantitative.

### Relabeling mechanics, commanded compliance, and required validation

Hindsight perturbation converts a disturbance into a counterfactual goal: after a force displaces a hand or other controlled point, the observed offset is relabeled as if it had been requested. The policy can then receive dense reference-tracking reward for the yielded pose instead of being simultaneously rewarded to return rigidly and penalized for force. Sampling perturbation direction, magnitude, duration, application point, and phase creates a family of reachable compliant responses without solving offline inverse dynamics for each one.

The deployment interface must clarify what the user controls—target offset, compliance scale, force threshold, or a latent—and how this quantity affects realized force–displacement response. A three-point head/hands command leaves legs and torso under the motion prior, which is useful for teleoperation but underdetermined. Observation history lets the actor infer external influence after it changes motion; direct force/tactile input would respond earlier and distinguish contact from reference error. Joint targets remain subject to PD gains, so apparent compliance includes both policy and servo behavior.

Ablations should compare ordinary disturbance randomization, relabeling without clean-reference reward, explicit compliant examples, and fixed low impedance. Report task error, peak/impulse force, effective stiffness by direction/frequency, balance, recovery, and unperturbed skill retention. Hindsight examples can legitimize unsafe large displacement if the rollout survived by chance, so acceptance needs joint, support, energy, and collision checks. Passivity or CBF constraints and multi-contact relabeling would turn this clever training trick into a safer general interaction controller.

### Quantifying compliance or safety beyond nominal task success

For CHIP: Adaptive Compliance for Humanoid Control through Hindsight Perturbation, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

The complete interface should specify sensed contact information, proprioceptive history, reference or safety command, policy architecture, output action, servo gains, and rates from sensing through actuation. If forces are not measured directly, the note must distinguish inferred disturbance response from actual force regulation. If a theorem or barrier is used, its model, disturbance bound, discretization, perception uncertainty, and feasibility assumptions must be connected to the physical implementation. If safety is reward learned, it is an empirical distributional property rather than a hard maximum.

Experiments should sweep contact point, direction, magnitude, duration, speed, material stiffness, support phase, payload and unseen combinations. Report peak force, impulse, pressure where available, realized directional/frequency-dependent impedance, energy exchanged, joint torque/current, pose or task error, balance loss, falls, false interventions, and recovery time. Comparisons should include a stiff robust tracker, low fixed impedance, ordinary push randomization, and any analytic safety or passivity layer. Human-interaction claims additionally need a controlled comfort protocol and hardware force validation rather than videos alone.

A useful architecture separates three responsibilities: a broad motor policy coordinates posture and stepping, a contact-aware adaptation layer selects yielding or protection, and an independent verified envelope limits force, energy, collision or joint state. Uncertainty should cause conservative slowing or takeover, and infeasible constraints need an explicit priority and emergency behavior. Future work should fuse tactile and force/torque sensing, model distributed multi-contact, calibrate sim contact against hardware, and connect the final contact state to recovery instead of terminating the evaluation at first survival.

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

## CLONE: Closed-Loop Whole-Body Humanoid Teleoperation for Long-Horizon Tasks

CLONE targets coordinated, long-horizon teleoperation without the global-position drift of local-frame trackers or the restricted upper/lower-body separation of many stable systems. Using only head and hand poses from an MR headset, it supports locomotion plus manipulation such as walking and picking objects from the floor.

A mixture-of-experts policy represents diverse whole-body motion modes. A privileged teacher learns from the CLONED motion corpus and full-body state; a student is distilled to sparse deployable observations. Real-time global pose feedback closes the loop between operator and robot, correcting accumulated translation error. CLONED augments AMASS with motion editing, new mocap, and synthesized hand orientation.

Teacher/student control uses learned MoE/MLP policies with RL and distillation. Apple Vision Pro supplies 3D head and 6D hand poses and global tracking; the robot supplies proprioception. The interface is sparse and portable but depends on reliable headset localization.

Closed-loop global correction is the central practical contribution. Remaining risks are localization loss, latency, aggressive corrections, and the limited hand information available from head/hand-only tracking.

### Exact controller interface and judgment

The PPO/AMP teacher sees full link pose/velocity, next reference, errors and randomized physics and outputs 29 PD joint targets. DAgger distills a student that receives 25 frames of joints, joint velocities, IMU angular velocity/gravity and previous actions, plus current/target head and wrist position, velocity and wrist orientation. Three layer-wise MoE blocks each use four experts `(2048,512,512,256)` with top-2 routing. LiDAR global feedback is 10 Hz, policy 50 Hz and PD 1 kHz.

CLONED adds retarget editing, wrist sampling and custom squat/reach sequences to AMASS. The 25-frame/MoE ablations show temporal state and conditional capacity both matter. The system closes *intent drift*, but sparse targets still leave feet, elbows, collision and force to the learned prior. Localization confidence, tactile wrists/hands and collision-aware null-space control are the most useful next additions.


### Tracker interface and network

Apple Vision Pro supplies both wrist 6-D poses and head position. LiDAR odometry plus robot forward kinematics estimates those points globally, so the goal includes current-to-target errors rather than only local pose. A privileged PPO/AMP teacher sees all link poses/velocities, next reference state, tracking error, and randomized physics and outputs 29 joint-position targets. DAgger distills it into a deployable student.

The student receives 25 frames of joints, joint velocities, root angular velocity, projected gravity, and previous actions. Its goal block contains positions/velocities of the three tracked points, their errors, and wrist orientations. Three layer-wise MoE blocks each contain four feed-forward experts with widths `(2048, 512, 512, 256)`; top-2 routing blends active experts. The AMP discriminator is a `(256,256,256)` MLP. LiDAR updates at 10 Hz, policy at 50 Hz, and joint PD at 1 kHz, so history bridges stale localization updates.

The ablations support 25 frames and four experts: less history loses motion/error evolution, more enlarges optimization, and additional experts are redundant. The important advance is closed-loop global correction—locally plausible imitation is inadequate if a hand drifts away from the intended table over a long task. Limits are equally concrete: LiDAR delay/outage corrupts error, sparse targets leave elbows/feet/contact unspecified, and MoE routing is not a stability guarantee. Future work should feed localization uncertainty, tactile/force error, and scene collision margins to the controller and evaluate downstream data quality, not only operator-pose error.


### Long-horizon task meaning and safety interpretation

CLONE's “closed loop” closes the geometric loop between operator command and robot's measured global head/hand points; it is not force feedback to the human and not visual object servoing. This distinction matters for tasks such as walking to a table and reaching down: root drift would otherwise move the robot hand away from the intended workspace even if local arm pose remains correct, but a globally accurate wrist still does not prove a stable grasp. The CLONED motion collection deliberately emphasizes squat, reach, and locomotion–upper-body coordination so the experts see configurations absent from generic upright mocap.

MoE routing lets different motion regimes allocate capacity without hard-coded skill switches, while top-2 interpolation reduces discontinuity at expert boundaries. Nevertheless, the gate is another feedback path: a small pose/error change can alter active experts and therefore action. Gate entropy, expert occupancy, and action discontinuity should be reported during localization jumps. Because LiDAR is only 10 Hz, the 25-frame (roughly half-second at 50 Hz) history mixes repeated/stale global estimates with fast proprioception; timestamp/age features would let the network distinguish “unchanged” from “not updated.”

A stronger long-horizon evaluation would report meters/minutes before reset, cumulative root and wrist drift, correction overshoot after deliberate localization bias, and task success under packet loss. Coupling global correction to confidence-weighted MPC or a barrier layer could keep learned motion natural while bounding sudden catch-up commands.


### What CLONE changes relative to sparse pose tracking

The high point is not merely using an MoE. OmniH2O-like sparse tracking already demonstrates that head and hands can command a humanoid. CLONE adds the missing global feedback path and trains the policy to *correct* error smoothly rather than assuming its integrated local motion remains aligned. This is especially important for collecting long-horizon task data: a locally convincing trajectory is unusable for manipulation learning if the hands arrive at the wrong table.

Layer-wise routing is a reasonable match to heterogeneous motion because early and late representations can specialize differently. The ablation showing redundant experts beyond four is more informative than simply reporting a large MoE. However, top-2 routing can still switch discontinuously between modes, and the paper does not establish that experts correspond to semantically stable skills rather than statistical partitions.


### Research opportunities

The next step is to close more than position. Wrist force/tactile feedback could correct contact, hand observations could control fingers, and scene vision could prevent a globally corrected body from colliding with objects. Localization uncertainty should be fed to the policy so corrections are attenuated when LiDAR confidence drops. A learned operator-intent model could also replace some of the 25-frame history with a forecast of upcoming head/hand motion. Finally, evaluating downstream autonomous policies trained on CLONED trajectories would establish whether global correction improves data value, not only teleoperation appearance.


### Architecture interpretation and closed-loop behavior

The 25-frame state window and mixture-of-experts solve different ambiguities. History exposes recent velocity, tracking error, and command evolution; top-2 routing gives conditional capacity for regimes such as standing reach, stepping, crouching, and recovery. Three stacked MoE layers allow specialization at several representation levels rather than selecting one complete controller. Only two of four experts are active in each block, keeping inference cheaper than an equivalently large dense network.

The multi-rate loop is important: LiDAR updates at 10 Hz, the network at 50 Hz, and joint PD at 1 kHz. Geometry is stale between scans, so the policy relies on proprioceptive continuity. That suits static interiors but risks delayed reaction to moving people or objects. LiDAR supplies distance and shape independent of texture, yet does not identify grasp semantics or contact force.

CLONE should not be judged only by robot–operator pose error. Closed-loop remapping may intentionally alter pose to preserve balance or global hand arrival. Future evaluation should separate intent error, global drift, collision margin, and balance margin. Routing plots across tasks would also show whether experts acquire stable roles; without them, MoE specialization remains a plausible explanation rather than a demonstrated one.

## CLOT: Closed-Loop Global Motion Tracking for Whole-Body Humanoid Teleoperation

CLOT provides drift-free, high-dynamic teleoperation on a full-size 31-DoF Adam Pro while preserving global human-to-robot motion over long sequences. It demonstrates contact-rich loco-manipulation and agile motion with high-frequency localization feedback.

“Observation Pre-shift” randomly advances the target shown to the policy while keeping reward evaluation at the current frame. The policy learns smooth interpolation/correction from data instead of responding violently to global error. A transformer captures spatiotemporal motion structure, and AMP regularization suppresses unnatural correction strategies. Training uses 20 hours of curated physically appropriate human motion and more than 1,300 GPU hours.

Optical mocap measures both operator and robot global pose; online Pinocchio IK retargets human motion. The controller also uses robot proprioception. This yields accurate feedback but requires instrumented space.

CLOT turns localization feedback into a trainable tracking problem, but its infrastructure is less portable than VR-only systems and the global correction distribution must match deployment delays/errors.

### Network, rates, and interpretation

The actor receives joint/proprioceptive feedback, motion goal and previous action with ten-step history; an asymmetric critic adds simulator state. Linear token embeddings feed a Transformer over history, current state and goal. Observation Pre-shift sometimes advances the goal to a random future timestamp while reward remains on the current reference, teaching smooth interpolation rather than an impulsive global correction. AMP adds a learned naturalness constraint.

OptiTrack human input runs at 120 Hz (gloves 60 Hz), the policy at 50 Hz and robot control at 400 Hz. Training exceeds 1,300 GPU-hours on 20 hours of captured motion. The method demonstrates drift-free agility but not formal correction stability. Onboard localization, outage tests, overshoot/force metrics, and comparison with explicit error-feedback MPC would clarify whether learned pre-shift generalizes beyond the training error distribution.


### Closed-loop tracking and Observation Pre-shift

Actor input contains joint state, root orientation/angular velocity, goal pose and previous action, with a ten-frame history to handle partial observability. An asymmetric critic adds privileged simulation state. Each component is embedded as a token and a Transformer encoder attends jointly across history, current feedback and motion goals before producing joint targets.

During training, the observed reference is sometimes shifted to a random future time while reward remains aligned to the true current reference. The actor therefore experiences a positional discrepancy but is rewarded for interpolating smoothly rather than instantly jumping to the shifted target. AMP over the mocap distribution suppresses unnatural high-torque correction. Difficulty curriculum and randomized physical/observation conditions support deployment.

OptiTrack provides global pose at high rate; human motion is captured at 120 Hz, policy runs at 50 Hz, and robot PD at 400 Hz. This closes drift that local pose trackers cannot see. The cost is 1,300+ GPU-hours and an instrumented space. Pre-shift teaches an empirical recovery law, not bounded convergence. Future work should use onboard localization with confidence, compare against explicit MPC/error feedback, and measure correction overshoot, latency sensitivity, force and localization outages.


### Why pre-shifting differs from ordinary target noise

If both actor observation and reward are shifted together, the policy merely learns another valid phase of the motion. CLOT instead changes the command visible to the actor while evaluating behavior against the unshifted physical target. This creates a controlled inconsistency resembling accumulated global error: directly chasing the observed future pose would be penalized, so the optimal policy learns gradual correction that preserves balance and AMP naturalness. The technique is data-driven gain shaping without explicitly defining a root-error controller.

The Transformer tokenization can attend across joint/proprioceptive state, previous action, current global error, and the ten-frame command history. That provides more context than feeding one root-error scalar into an MLP, particularly when the same position error should be corrected differently during single support, a jump, or a hand contact. The asymmetric critic can use clean global quantities even when the actor's feedback is randomized, improving value learning without creating a deployment dependency.

The limitation is identifiability: command pre-shift, sensor delay, and true operator acceleration may look similar. A learned correction can lag or overreact when their distribution changes. Evaluation should sweep shift size/duration and localization latency independently, visualize the learned effective gain by gait phase, and compare energy/contact impulse with a conventional global-error controller. Onboard visual–inertial/LiDAR localization with covariance should replace OptiTrack before claiming operation outside an instrumented volume.

### Control-stack accounting and a stronger evaluation protocol

For CLOT: Closed-Loop Global Motion Tracking for Whole-Body Humanoid Teleoperation, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

## Evaluation and comparison of SEA torque controllers in a unified framework

Provides an apples-to-apples analysis of torque-control strategies for series-elastic actuators, comparing bandwidth, disturbance rejection, stability, model sensitivity, and implementation cost under one framework. It guides selection of the actuator layer beneath whole-body control. Conclusions depend on hardware parameters and test conditions rather than defining a universal winner.

### Unified transfer, impedance, and passivity criteria

The framework evaluates torque PID, cascaded motor-velocity/torque loops, full-state feedback, disturbance-observer variants, acceleration feedback, and adaptive/model-reference approaches on the same SEA model and hardware. Each maps desired output torque plus motor/load/spring measurements to motor command. Rather than tuning every method for its favorite metric, the study derives torque transfer, apparent output impedance, noise sensitivity, and passivity boundaries, then uses locked-output tracking, external disturbance, interaction, and impact tests to reveal friction and cogging.

This exposes a real design triangle. High bandwidth and DOB compensation reduce error but can raise noise sensitivity or active impedance; acceleration/model feedback depends on derivative and parameter quality; passive tuning sacrifices rejection but tolerates a broader unknown environment. Adaptive methods help repeatable interactions but may lose their advantage on unpredictable contact. The lasting contribution is therefore an evaluation language for choosing a controller, not a universal winner.

Linear single-actuator analysis still approximates saturation, backlash, sampling, delay, thermal effects, and multi-contact coupling. Future benchmarks should publish reproducible frequency sweeps, energy/passivity observers, current limits, and multi-joint tests. Learned compensation should be judged by the same impedance and interaction-stability metrics so nominal RMSE gains do not hide unsafe energy injection.

### Frequency-domain comparison and controller-selection guide

A unified SEA plant makes controller families comparable by deriving closed-loop transfer from desired to measured spring torque, output impedance seen by the environment, disturbance sensitivity, and noise amplification. PID may be simple and robust; cascaded velocity/torque loops exploit motor control infrastructure; full-state or acceleration feedback shape resonance; disturbance observers cancel friction/model error; adaptive schemes update uncertain dynamics. Each advantage changes phase margin or reliance on derivative/model signals.

Fair tuning is difficult. Matching torque bandwidth may give different noise and passivity; matching peak current may give different tracking; optimizing RMSE can hide active interaction impedance. The test matrix should include swept-sine reference, locked and free output, external torque disturbance, impacts, contact with several stiffnesses, payload/reflected-inertia variation, sensor noise, saturation, and delay. Report Bode plots, passivity/energy, RMS and peak current, temperature, and computation at a common sample rate. Releasing parameters and raw traces is more useful than declaring a universal winner.

For whole-body control, the result should become a joint-specific selection rule. An ankle supporting impacts may prioritize passive impedance and robustness; an arm tracking free-space torque may prioritize bandwidth; a hand contact joint may need low apparent impedance. Higher layers should know each loop's achievable torque envelope and delay. Learned compensation can be added, but it must be benchmarked under the same impedance, passivity, and out-of-distribution load tests. A hybrid controller that schedules conservative verified loops by contact phase may be safer than one aggressively tuned design across the entire humanoid.

### Reproducibility, stability margins, and whole-body integration

For Evaluation and comparison of SEA torque controllers in a unified framework, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

A reproducible actuator result must state motor and load inertia, gear ratio, spring or transmission stiffness, torque-sensor location and resolution, current-loop bandwidth, servo rate, filters, delay, saturation, friction compensation, and controller gains. Reference tracking should include swept-sine bandwidth and phase, steps and reversals; interaction tests should include several reflected inertias and environment stiffnesses, impacts, and sustained load. RMS error alone hides noise amplification, active output impedance, current peaks, heating, backlash, and energy injection. Frequency-domain prediction should be checked against measured Bode plots and time-domain contacts, with raw traces and thermal state disclosed.

Whole-body consequences need a second level of evaluation. The higher controller assumes a torque source, but every joint has different bandwidth and capacity and multibody contacts couple otherwise decentralized loops. Report joint torque residuals during walking, impacts, manipulation and saturation, and show how they affect task-space tracking, balance, contact wrench and passivity. A policy trained with ideal actuation should be tested with the measured closed-loop actuator model, delay and limits. The actuator layer should expose available torque-speed envelope, temperature derating, estimated error and fault status to whole-body control.

My assessment criterion is not which method wins one nominal trace, but which maintains an explicit stability and energy margin over uncertain load, contact and wear. Online identification or learned residuals can improve compensation, yet must be bounded and fall back to a conservative verified loop. Passivity observers, energy tanks, torque-rate limiting, anti-windup and fault detection belong in the evaluation because they determine whether improved bandwidth remains safe around people and hardware.

## ExBody2: Advanced Expressive Humanoid Whole-Body Control

Source: [arXiv 2412.13196](https://arxiv.org/abs/2412.13196).

ExBody2 tracks arbitrary human motion on real humanoids while balancing expressiveness and stability. It decouples keypoint tracking from velocity control and distills a privileged teacher into a deployable student, avoiding dependence on unavailable global/privileged state. Simulation RL and domain randomization transfer running, crouching, and dancing to two platforms. Decoupling lets the legs stabilize while the body expresses the reference, but exact lower-body fidelity may be sacrificed and performance remains limited by the student observation/state estimator.

### Generalist–specialist data and control

An initial policy measures lower-body keypoint/joint error for every retargeted clip. Moderate-threshold filtering removes infeasible lower-body dynamics while retaining diverse upper-body expression; too strict a threshold loses coverage and too loose preserves harmful data. A generalist learns the selected set, then category specialists fine-tune from it for dance or kung fu rather than relearning balance.

The privileged PPO teacher observes true root velocity, link positions, friction/motor properties, proprioception and motion goals. Its 23-D output is a PD joint-position target. DAgger trains a student from observation history and current goal using teacher-action MSE, removing simulation-only state.

Global motion is split into local keypoint imitation plus root velocity/direction and roll–pitch/yaw goals. Periodic recentering tolerates position drift while preserving expressive pose; velocity supplies travel. This avoids aggressive catch-up to unreachable global points but gives up exact world position. Hardware results show lower errors, while specialists improve target categories at some cost elsewhere. Future work should learn continuous specialization/routing, expose feasibility confidence, and close global drift only with reliable localization.


### Decoupling trade-off and student observability

The control goal separates upper/full-body keypoint geometry from locomotion variables because absolute lower-body keypoints couple reference morphology, terrain contact, and accumulated root error. Local keypoints preserve expressive arm/torso/leg shape, while commanded root planar velocity, heading, and attitude let the policy choose dynamically stable steps. Periodic reference recentering prevents a small early foot error from becoming an ever-growing global-position penalty. This is why ExBody2 can remain expressive without the violent catch-up behavior of strict world-frame imitation.

The teacher's true root velocity, link state, dynamics parameters, and contact information make balance easier during PPO, but the student must infer these factors from observation history. DAgger is essential because student deviations create histories not present in teacher-only rollouts; querying the teacher on those states teaches corrective actions. Reporting student history length, teacher–student action error by motion class, and the fraction of failures caused by missing velocity estimation would make this observability claim more reproducible.

Filtering lower-body error is both curriculum and censorship. It can keep rich upper-body motion while dropping impossible leg sequences, but the retained set may bias toward motions already easy for the chosen robot. A better pipeline would store per-clip feasibility scores and use them as conditioning, allowing the same policy to slow, simplify, or reject a reference rather than permanently deleting it. Global navigation feedback can then be layered above the velocity interface without reintroducing strict whole-body world-frame tracking.

### Control-stack accounting and a stronger evaluation protocol

For ExBody2: Advanced Expressive Humanoid Whole-Body Control, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

## Extreme-RGMT: Continual Learning of Highly Dynamic Skills for Robust Generalist Humanoid Control

Source: [arXiv 2607.20110](https://arxiv.org/abs/2607.20110).

Extreme-RGMT first trains a broad tracker, then adds rare dynamic skills without forgetting ordinary motions. Asymmetric skill acquisition constrains drift on mastered data while emphasizing new difficult segments; difficulty-aware sampling and advantage-prioritized trajectory replay focus scarce successful transitions. It executes unseen fixed references and live inertial mocap. Continual consolidation is more data-efficient than rebuilding a generalist, but its stability–plasticity weights can block learning or permit forgetting, and highly dynamic hardware safety still relies on conservative training/deployment boundaries.

### Base tracker and continual expansion

The actor receives ten frames of projected gravity, base angular velocity, joint offsets/velocity and actions, plus a 21-token reference window of base velocities, gravity and joints. Separate MLPs encode interleaved state/action history; a causal encoder queries the reference through cross-attention. FSQ regularizes the command before actor fusion. A privileged critic sees reference height, link pose and base linear velocity. The 29-D residual is added to reference joints before PD.

After base PPO, performance splits mastered and challenging motions. PACE updates aggressively on challenges while reference-policy regularization and a progress-adaptive consolidation weight preserve mastered behavior. Difficulty sampling revisits failed segments. STAR retains high-advantage fragments from scarce successful dynamic rollouts instead of replaying long trajectories dominated by failure.

The key idea is locating continual learning inside one imbalanced tracking distribution. Yet replay advantage depends on a changing critic, consolidation may freeze weak habits, and FSQ can lose contact detail. Future evaluation should publish per-skill forgetting matrices, replay age/bias, repeated acquisition cycles, and impact/torque safety; parameter-efficient routed specialists are a useful baseline.


### PACE/STAR interaction and continual-learning diagnostics

PACE uses different learning pressure for old mastered clips and incoming extreme clips. The new-skill objective remains tracking-driven, whereas a reference-policy constraint limits action-distribution drift on mastered states. Its consolidation weight changes with learning progress: strong enough to prevent catastrophic forgetting when acquisition becomes aggressive, but relaxed when the new skill cannot improve. This is more targeted than mixing all old and new clips uniformly, where numerous easy transitions can drown the rare aerial/contact-critical segments.

STAR further changes the replay unit from episode to informative fragment. In a failed backflip, the approach, takeoff, or partial rotation may have positive advantage even when landing terminates. Retaining high-advantage windows gives later updates samples of the scarce successful substructure rather than replaying mostly failure. The danger is critic nonstationarity: a fragment labeled valuable early may become misleading after the policy changes, so age-based re-evaluation and off-policy correction deserve explicit analysis.

“No forgetting” should be evaluated beyond average old-set success. Per-clip error before and after every acquisition stage, transitions between old and new skills, action/torque distribution shift, and recovery after aborting a dynamic maneuver are more diagnostic. Repeating the process over several sequential skill batches would test whether consolidation capacity eventually saturates. Hardware deployment should also include a feasibility gate that prevents live mocap from requesting an extreme skill when takeoff space, joint temperature, or state-estimation confidence is inadequate.

### Control-stack accounting and a stronger evaluation protocol

For Extreme-RGMT: Continual Learning of Highly Dynamic Skills for Robust Generalist Humanoid Control, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

## FRoM-W1: Towards General Humanoid Whole-Body Control with Language Instructions

[FRoM-W1](https://arxiv.org/abs/2601.12799) uses H-GPT to generate human whole-body motion from language, retargets it, then uses H-ACT—pretraining plus simulation RL fine-tuning—to track it on H1/G1 robots. It links open-ended instructions to stable physical action. Language ambiguity and generated-motion feasibility still require grounding and safeguards.

### Language generation separated from physical execution

H-GPT is a large language-conditioned human-motion generator trained on massive human data; chain-of-thought decomposition improves interpretation of complex instructions before producing a whole-body sequence. Robot-specific retargeting maps that kinematic human output onto H1 or G1 morphology. H-ACT then supplies the physics layer: a pretrained tracker is fine-tuned with simulation RL for stable, accurate execution, and a modular sim-to-real stage handles deployment.

The pipeline's interfaces are important. Text goes to human pose sequence; retargeting yields robot reference joint/body trajectories; H-ACT consumes reference plus proprioception and outputs actuator/joint targets. The language model never directly commands torque, which permits feasibility repair and reuse across embodiments. Reported HumanML3D-X generation results and real H1/G1 motions show that RL fine-tuning improves both tracking and task success.

The separation is sensible but errors cascade: plausible H-GPT motion may violate robot contacts, retargeting can destroy semantics, and a tracker may follow unsafe language literally. Chain-of-thought does not certify grounding. Future work should close the loop with environment perception, reject infeasible/unsafe generations before tracking, use contact-aware retargeting and user clarification, and evaluate compositional instructions beyond choreographed open-loop actions.

### Generation–retargeting–control interfaces and safety checks

FRoM-W1 decomposes a language request into a human motion program before touching robot control. H-GPT produces a time-indexed human pose sequence, potentially using textual decomposition for multi-part instructions. Retargeting maps body proportions, joint conventions, root motion, and contact intent to H1 or G1. H-ACT consumes robot reference and proprioception and produces actuator or joint-position targets; simulation RL fine-tunes tracking under contacts and dynamics randomization. This interface permits the same language generator to serve multiple bodies while each robot retains a specialized physics controller.

Every boundary needs measurable fidelity. Text-to-motion should be scored for semantic correctness and diversity; retargeting for keypoint/contact error, limits, penetration, and smoothness; tracking for survival, pose/velocity error, torque, and hardware transfer. End-to-end task success alone cannot reveal whether a wrong gesture came from language interpretation or physical simplification. Comparisons should include direct retrieval, generator without chain-of-thought, kinematic execution without RL refinement, and robot-specific versus shared generation.

Language is an open command channel and therefore a safety surface. Instructions may be ambiguous, physically impossible, environmentally inappropriate, or adversarial. A feasibility stage should simulate and classify the retargeted sequence, modify speed/amplitude, or ask for clarification before hardware execution. Scene perception and contact affordances are required for phrases involving objects or locations; open-loop generation cannot ground them. The strong contribution is modular breadth, but future work should close the loop by replanning from robot/environment feedback and report latency from instruction to first safe action, not only final motion quality.

### Control-stack accounting and a stronger evaluation protocol

The decisive chain is instruction interpretation → human-motion generation → embodiment retargeting → physics-aware execution. Semantic correctness, geometric feasibility, and closed-loop stability are different errors and should never be collapsed into one success number.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

## GentleHumanoid: Learning Upper-body Compliance for Contact-rich Human and Object Interaction

Learns compliant upper-body response while a stable lower body maintains support. Contact randomization and objectives limiting force/impulse let the humanoid accept pushes, handovers, or close interaction without rigidly fighting them. The division improves gentleness, but sensing/modeling external force and balancing compliance against task accuracy remain difficult.

### Unified multi-joint interaction-force model

GentleHumanoid augments a whole-body motion tracker with spring–damper-style interaction forces applied coherently along shoulder–elbow–wrist chains rather than random independent link pushes. Simulation creates diverse, kinematically consistent human/object interactions and varies desired compliance. PPO learns joint targets that combine reference following with yielding, while task-adjustable force thresholds constrain contact severity. Camera-estimated partner shape can condition hugging posture; other tasks include sit-to-stand assistance and object manipulation.

The key advance is upper-body compliance without sacrificing the lower-body balance prior. Large-perturbation robustness training usually teaches the robot to fight forces; here force direction, chain coupling, and tunable limits teach when to yield. Experiments report lower peak force and more natural adaptation than stiff or perturbation-resistant baselines.

Simulated spring forces are only a proxy for skin, clothing, human motion, and distributed contact, and threshold enforcement through reward is not a hard guarantee. Tactile/force feedback, online partner-intent estimation, passivity constraints, per-link pressure limits, and user studies of comfort are needed. The controller should also distinguish desired assistance load from a destabilizing shove rather than treating force magnitude alone as intent.

### Compliance command, sensory observability, and human-interaction evidence

The simulated interaction-force model creates correlated forces along an arm chain, approximating a partner pushing or pulling through the hand while load transmits through wrist, elbow, and shoulder. A compliance condition or force threshold tells the policy how much to yield. Proprioception and reference goals feed the PPO actor, which outputs desired joint positions; privileged force/contact and dynamics can improve the critic during training. Lower-body balance remains part of the same rollout, so yielding at the arm cannot be learned independently of stepping or torso compensation.

This differs from generic push robustness: a robust policy minimizes deviation and fights the disturbance, whereas a gentle policy accepts controlled deviation to reduce force while preserving task intent. Experiments should sweep force direction, speed, duration, contact point, desired threshold, partner trajectory, and support condition. Report peak and impulse force, realized stiffness, pose/task error, balance/fall, response delay, and recovery after release. Human comfort studies and per-link pressure matter more than visually plausible motion.

Observability is the limiting factor. Without wrist force/torque, tactile skin, or reliable contact estimation, the controller infers interaction only after joint/base motion changes. Camera-estimated partner geometry can shape a hug but not measure distributed pressure. Reward penalties also do not guarantee a maximum force. A stronger architecture would fuse tactile and force sensing, use a passive impedance or barrier layer for certified limits, and let RL coordinate posture within that envelope. Partner intent should be inferred from motion and sustained force so assistance load, cooperative handover, and accidental shove elicit different responses rather than one scalar “softness.”

### Quantifying compliance or safety beyond nominal task success

For GentleHumanoid: Learning Upper-body Compliance for Contact-rich Human and Object Interaction, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

The complete interface should specify sensed contact information, proprioceptive history, reference or safety command, policy architecture, output action, servo gains, and rates from sensing through actuation. If forces are not measured directly, the note must distinguish inferred disturbance response from actual force regulation. If a theorem or barrier is used, its model, disturbance bound, discretization, perception uncertainty, and feasibility assumptions must be connected to the physical implementation. If safety is reward learned, it is an empirical distributional property rather than a hard maximum.

Experiments should sweep contact point, direction, magnitude, duration, speed, material stiffness, support phase, payload and unseen combinations. Report peak force, impulse, pressure where available, realized directional/frequency-dependent impedance, energy exchanged, joint torque/current, pose or task error, balance loss, falls, false interventions, and recovery time. Comparisons should include a stiff robust tracker, low fixed impedance, ordinary push randomization, and any analytic safety or passivity layer. Human-interaction claims additionally need a controlled comfort protocol and hardware force validation rather than videos alone.

A useful architecture separates three responsibilities: a broad motor policy coordinates posture and stepping, a contact-aware adaptation layer selects yielding or protection, and an independent verified envelope limits force, energy, collision or joint state. Uncertainty should cause conservative slowing or takeover, and infeasible constraints need an explicit priority and emergency behavior. Future work should fuse tactile and force/torque sensing, model distributed multi-contact, calibrate sim contact against hardware, and connect the final contact state to recovery instead of terminating the evaluation at first survival.

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

## GMT: General Motion Tracking for Humanoid Whole-Body Control

Source: [arXiv 2506.14770](https://arxiv.org/abs/2506.14770).

GMT trains one real-world policy across a wide motion distribution. Adaptive Sampling balances easy, failed, and hard clips so training effort follows competence, while a Motion Mixture-of-Experts specializes different regions of the motion manifold. The unified policy demonstrates dance, sports, kung fu, locomotion, and unseen references. MoE capacity and sampling solve heterogeneity better than a monolithic MLP, though expert routing can fragment data, difficult motions may remain underrepresented, and physical transfer depends on observation restrictions and randomization.

### Adaptive data, long lookahead, and Motion MoE

AMASS/LAFAN clips over ten seconds are periodically re-clipped with random offsets so difficult moments are not diluted by easy lead-in. Completion and maximum key-body error determine sampling and a tightened termination threshold. Rule filtering plus a preliminary five-billion-sample policy leave 8,925 clips/33.12 hours.

The PPO teacher is a soft MoE: gate and experts receive state plus goal; experts output action distributions; their probability-weighted action avoids hard switches. DAgger yields the student. A goal frame includes 23 joints, six base velocities, roll/pitch, height, and heading-local key bodies. One hundred future frames—about two seconds—are compressed by a convolutional encoder to 128 dimensions and concatenated with the immediate target. Joint commands run under domain randomization and delay.

Sampling, lookahead, and MoE address imbalance, anticipation and capacity separately. Teacher/student errors remain close and outperform an ExBody2 reproduction. However, curation excludes crawling/fallen/extreme motion, and future reference is unavailable for reactive teleoperation unless predicted. Routing need not be semantically stable. Future work should test causal uncertain forecasts, publish expert utilization, and evaluate recovery/interaction beyond the curated manifold.


### Adaptive sampling signals and soft expert composition

GMT avoids assigning one expert to a manually named skill. The gate sees the same state/reference context as the experts and blends their action distributions, allowing a transition to use several specialists and avoiding torque discontinuities from hard switching. This soft composition is particularly relevant when a clip changes from walking to a kick: the useful partition may be by balance/contact regime rather than semantic label. DAgger then distills the privileged MoE teacher into one deployable student, so runtime need not expose simulator-only velocity/contact variables.

Adaptive Sampling uses both completion and maximum key-body error because termination alone is coarse: two surviving clips can differ greatly in fidelity. Periodic re-clipping prevents a short difficult event near the end of a long sequence from receiving little training probability. Tightening termination as competence grows creates a precision curriculum; starting with the final strict threshold would prematurely remove nearly every rollout of dynamic clips.

The 100-frame/two-second goal window provides unusually long anticipation. Its convolutional encoder is cheaper than attending to every frame, but assumes time-aligned, reliable future reference. For live tracking, prediction error grows with horizon and could cause premature motion. A deployable extension should attach confidence and mask uncertain future frames, shorten lookahead after detected command changes, and compare against phase-conditioned recurrent prediction. Expert-gate ablations should include routing entropy and action smoothness, not only mean tracking reward.

### Control-stack accounting and a stronger evaluation protocol

For GMT: General Motion Tracking for Humanoid Whole-Body Control, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

## HALO:Closing Sim-to-Real Gap for Heavy-loaded Humanoid Agile Motion Skills via Differentiable Simulation

Uses differentiable simulation/system identification to fit loaded robot dynamics and refine policies for agile motion under substantial payload. Gradients help calibrate parameters or optimize control beyond coarse domain randomization. It closes a difficult load-dependent transfer gap, though differentiable contact approximations and changing payload configurations can reduce fidelity.

### Real-data identification before RL policy training

HALO first trains a wide-domain-randomized exploration policy and constrains one foot on hardware so full-body motion can be reconstructed from joint encoders without external mocap. It collects unloaded and payload-loaded state/action trajectories, where actions are target joint positions tracked by PD. Differentiable simulation then optimizes structured parameters—payload mass/center/inertia and robot model biases—by backpropagating trajectory mismatch through dynamics.

The identified model becomes the center of a much narrower randomization distribution for subsequent agile tracking RL. This differs from online adaptation: model bias is reduced before the final controller trains, so the policy need not be conservative across implausibly broad dynamics. Reported gains under heavy loads include large improvements in gait precision and motion tracking with zero-shot hardware transfer.

The fixed-foot collection protocol is lightweight and the structured payload estimate is interpretable. But identifiability depends on excitation; differentiable contact/actuator models can assign error to the wrong parameter, and a payload that shifts after calibration invalidates the result. Active excitation selection, posterior uncertainty rather than one parameter vector, online change detection, actuator/thermal identification, and contact-rich validation would improve robustness.

### Identifiability, randomization width, and payload change

Differentiable identification minimizes multi-step mismatch between recorded hardware trajectories and simulated state under the same desired-joint commands. Structured parameters—payload mass, center of mass, inertia and selected robot/actuator biases—are optimized through the simulator. Multi-step loss is necessary because several wrong parameters can match one acceleration instant; diverse excitation across joints and directions improves rank. Fixed-foot collection reduces global-state ambiguity but does not excite walking contacts in the same way as free locomotion.

The learned point estimate should be accompanied by curvature or posterior uncertainty. Strongly correlated payload COM and torso inertia, or motor strength and friction, can produce similar trajectories. Final policy randomization should be narrow only along well-identified directions and remain broad along uncertain ones. Baselines should include nominal model, broad domain randomization, black-box residual dynamics, and identification without payload structure, compared at equal hardware data and RL steps.

Hardware evaluation should sweep payload mass, placement, rigidity, and motion family and report tracking, falls, torque/current, power regeneration, thermal margin, and model prediction on held-out excitation. A load that shifts, sloshes, or is released violates offline calibration; change detection from proprioceptive residuals should select or update a parameter belief online. Differentiable contact bias may otherwise be incorrectly absorbed into body parameters. My judgment is that centering randomization on measured hardware is a strong sample-efficiency principle, but robust deployment needs uncertainty-aware identification rather than treating the fitted simulator as truth.

### Control-stack accounting and a stronger evaluation protocol

The decisive chain is hardware excitation → differentiable parameter identification → uncertainty-centered simulation → agile-policy training under payload. Each stage should be evaluated separately so better tracking is not incorrectly attributed to the final RL policy when it actually comes from a better fitted plant.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

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

## Hold My Beer: Learning Gentle Humanoid

Focuses on robust yet nonviolent response during contact, teaching the humanoid to yield, stabilize, and limit forces rather than maximize rigid tracking. Randomized pushes/interactions and gentle-contact rewards produce safer upper-body behavior. Reduced impedance may sacrifice precision or recovery speed under large disturbances.

### Gentle behavior rather than disturbance rejection

The controller is trained around a whole-body tracking prior but exposes the upper body to randomized external interactions and penalizes peak/accumulated contact, abrupt action, torque, and aggressive recovery. The actor consumes proprioceptive/reference information and produces joint-position targets; simulated perturbations force it to coordinate arm yielding with torso and stepping balance. Unlike a robust tracker whose optimal response is to return immediately to the reference, the learned behavior tolerates temporary deviation to reduce impulse.

This reframing matters for human contact: “robust” and “safe” can conflict when the robot wins a force contest. Demonstrations emphasize maintaining a held object/task and recovering without violent counteraction. The work is strongest as a reward/distribution argument, not a formal impedance controller.

Gentleness learned from contact penalties is empirical and may fail for unseen body regions, hard objects, or fast impacts; lower impedance can also reduce manipulation accuracy and allow cumulative drift. Explicit force/tactile input, task-conditioned compliance, energy/passivity monitoring, hard force limits, and comparisons using pressure/impulse and human comfort—not only success—would make the claim more operational.

### Force objectives, contact distribution, and the meaning of gentleness

The training distribution should distinguish persistent cooperative load, brief accidental collision, a shove threatening balance, and contact caused by the robot itself. Reference tracking, support stability, task completion, contact force or impulse, joint torque, action rate, and recovery compete in the reward. Penalizing force everywhere can teach avoidance or dropping an object, while penalizing pose error too strongly recreates stiffness. Curriculum over force and support condition lets the policy first retain balance and then learn controlled yielding.

The actor's joint-position output is realized by PD servos, so commanded gains and hardware backdrivability set a lower bound on apparent stiffness. Proprioception alone detects interaction after deflection; wrist force/torque or tactile sensing provides earlier local evidence. Experiments should report force–displacement and force–velocity curves, peak pressure and impulse by body part, held-object motion, balance/fall, response time, and behavior after contact release. Human-rated comfort is useful only alongside measured physical quantities and a documented protocol.

“Gentle” is not always “soft.” Carrying a heavy object may require high support stiffness while a forearm contact should yield; preventing a fall may justify a brief strong corrective step. A task- and body-part-conditioned compliance command is therefore preferable to one global penalty. Passivity observers, energy tanks, hard per-link force limits, and tactile feedback could provide a verified envelope within which the learned whole-body policy coordinates posture. Tests with unseen human motion and hard/soft objects should probe whether the policy learned interaction principles or only the simulated perturbation family.

### Quantifying compliance or safety beyond nominal task success

The decisive distinction is yielding safely versus rejecting disturbance aggressively. Force, impulse, effective stiffness, task deviation, and balance recovery must therefore be evaluated together.

The complete interface should specify sensed contact information, proprioceptive history, reference or safety command, policy architecture, output action, servo gains, and rates from sensing through actuation. If forces are not measured directly, the note must distinguish inferred disturbance response from actual force regulation. If a theorem or barrier is used, its model, disturbance bound, discretization, perception uncertainty, and feasibility assumptions must be connected to the physical implementation. If safety is reward learned, it is an empirical distributional property rather than a hard maximum.

Experiments should sweep contact point, direction, magnitude, duration, speed, material stiffness, support phase, payload and unseen combinations. Report peak force, impulse, pressure where available, realized directional/frequency-dependent impedance, energy exchanged, joint torque/current, pose or task error, balance loss, falls, false interventions, and recovery time. Comparisons should include a stiff robust tracker, low fixed impedance, ordinary push randomization, and any analytic safety or passivity layer. Human-interaction claims additionally need a controlled comfort protocol and hardware force validation rather than videos alone.

A useful architecture separates three responsibilities: a broad motor policy coordinates posture and stepping, a contact-aware adaptation layer selects yielding or protection, and an independent verified envelope limits force, energy, collision or joint state. Uncertainty should cause conservative slowing or takeover, and infeasible constraints need an explicit priority and emergency behavior. Future work should fuse tactile and force/torque sensing, model distributed multi-contact, calibrate sim contact against hardware, and connect the final contact state to recovery instead of terminating the evaluation at first survival.

Long-duration interaction tests should additionally measure cumulative drift, actuator temperature, and whether repeated yielding gradually moves the robot into a posture from which it cannot safely continue the task.

## HoloMotion-1 Technical Report

HoloMotion-1 scales a real-time general motion tracker using broad retargeted motion, curriculum/resampling, privileged teacher learning, and deployment-oriented student observations. It targets high-fidelity whole-body execution across locomotion, expressive motion, and teleoperation rather than per-skill policies. A strong tracker also serves as the “cerebellum” beneath motion generators such as OMG. Its breadth depends on motion/data quality; flat-ground tracking does not imply robust object/terrain contact, and challenging references can still require filtering or adaptation.

### Observation, action, and RL objective

The actor sees projected gravity, root angular velocity, relative joint positions, joint velocities, and previous action. Its reference block contains current targets plus ten future control steps of projected gravity, root linear/angular velocity, joint targets, and root height. This lookahead lets the policy prepare for impacts and fast pose changes. During PPO training those deployable inputs receive calibrated noise; an asymmetric critic additionally receives exact anchor/reference differences and robot-link state.

Actions are normalized offsets around default joint angles, converted into PD position targets and then torque. Tracking reward covers root-relative key-body position/orientation/velocities, root velocity ratios, and a heavily weighted local five-point objective over torso, wrists, and ankles. Absolute root pose rewards are disabled in the main configuration, reducing dependence on unavailable global localization. Action-rate, acceleration, joint-limit, and unwanted-contact penalties regularize execution. Episodes terminate for large gravity, key-body-height, or pelvis-position deviation.

Randomization includes zero-to-two-step action delay, rough height fields, friction/restitution, default joint offsets, torso mass/center of mass, PD gains, initial pose/velocity, and periodic linear/angular pushes. These details matter more for transfer than the label “general tracker.”

### Sparse MoE causal Transformer

Each normalized observation/reference vector becomes one token. A decoder-only Transformer operates on a rolling causal history, using RMSNorm, rotary positions, grouped-query attention, query/key normalization, and gated attention. Sparse MoE feed-forward blocks contain one shared expert plus a large specialist pool; top-k routing raises capacity without evaluating every expert.

The router uses **reference motion only**, not proprioception. The paper finds that state-dependent routing amplifies small sim-to-real errors: a noisy state can select another expert, changing action, which further changes state and triggers routing oscillation. Reference-only routing gives the same motion consistent experts in simulation and hardware while the selected experts still receive full state for feedback. A Gaussian head produces joint actions with state-independent learned variance. Auxiliary pre-MoE heads predict base velocity, contacts, and key-body signals during training to make early representations motion aware.

### Scaling evidence and assessment

HoloMotion-1 combines diverse motion sources and tests zero-shot tracking on five held-out datasets. Model/data scaling and sequence-level optimization improve coverage, while sparse activation keeps real-time inference plausible. The architecture is well matched to a heterogeneous corpus: attention handles temporal context, and experts allocate capacity to different motion regimes.

The main scientific risk is that expert routing can partition by dataset artifacts rather than reusable dynamics. Reference-only routing is stable but cannot choose a recovery specialist based directly on an unexpected fall. Ten-step lookahead assumes future references are available and feasible. Absolute-root rewards are omitted, so a visually good policy may accumulate global drift in long teleoperation.

Future evaluation should publish expert-utilization and routing-stability traces, per-motion tail failures, end-to-end latency, torque/contact loads, and global drift. A slow reference router plus a fast state-triggered residual/recovery path could preserve sim-to-real stability while reacting to unexpected dynamics. Scene/contact tokens and explicit feasibility prediction are needed before the same architecture can claim general interaction rather than broad free-space motion tracking.


### Sequence PPO, caching, and real-time scaling

HoloMotion-1 optimizes the autoregressive policy over complete token sequences rather than repeatedly treating a Transformer as a one-step MLP. A rollout stores the causal context, actions, log probabilities, values, rewards, and terminations; PPO forms advantages across time and evaluates chunks in parallel with the same causal mask used at deployment. This lets gradients train temporal credit and routing consistently while reducing redundant forward work. During inference, the key/value cache stores attention projections from past tokens, so each new control step computes attention only for the new token rather than replaying the entire context.

Sparse MoE feed-forward layers combine a shared expert with top-k routed specialists. Only selected experts execute per token, increasing parameter capacity without proportional FLOPs; an auxiliary load-balancing objective is needed to prevent all references from collapsing onto a few experts. Routing from reference features stabilizes expert identity across simulator and hardware, while full noisy proprioception still enters the selected expert and attention feedback path. This is a reasonable compromise between open-loop skill selection and closed-loop control.

Exact termination thresholds reveal what “successful tracking” means during PPO: projected-gravity mismatch above 0.8, vertical error over 0.25 m on pelvis/ankles/wrists, or pelvis drift over 0.25 m ends a rollout. Randomization includes zero–two control-step delay, up to 4-cm rough terrain, broad friction/restitution, torso mass/COM, PD gain, initial-state, and periodic velocity pushes. These ranges should accompany success rates because they define robustness. Future scaling results should hold active expert FLOPs and context latency fixed while varying total parameters, thereby separating the benefit of sparse capacity from simply spending more computation.

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

## HOVER: Versatile Neural Whole-Body Controller for Humanoid Robots

Source: [arXiv 2410.21229](https://arxiv.org/abs/2410.21229).

HOVER distills a full-body motion imitator into one 1.5M-parameter controller supporting more than 15 masked command modes: keypoint positions, selected joint angles, and root velocity/height/orientation. VR, RGB pose, exoskeleton, arms, mocap, or joystick interfaces can activate different subsets. Full-body imitation provides the common motor foundation. Masking makes the interface versatile, but ambiguous sparse modes require the learned prior to choose posture, and unsupported command combinations may be unpredictable.

### Multi-mode distillation

A full-reference PPO teacher learns the complete motion distribution. Student training randomly masks goal components—root velocity/height/orientation, selected joint angles, and body keypoint positions—so the same MLP maps proprioception plus a command mask/value pair to PD targets. Distillation supplies the teacher's physically coherent action when a sparse command admits many poses. Mode curricula prevent the dense full-body condition from dominating easier low-dimensional modes.

The result is a compact motor interface: joystick navigation, upper-body joint control, sparse VR/RGB keypoints, exoskeleton or full mocap can switch modes without changing policy. Its novelty is treating control mode as missing reference fields rather than separate tasks.

Masking does not resolve contradictory commands or show whether a combination is reachable. Unspecified limbs follow dataset priors and may collide with a scene. Future work should add command confidence/priority and feasibility prediction, train continuous sensor dropout and mode switches, and expose environment/contact observations when the null-space behavior matters.


### Command representation, network scale, and practical interpretation

The 19-DoF robot is controlled by a roughly 1.5-million-parameter network, showing that HOVER is a compact motor primitive rather than a large semantic model. Its unified command vector has three blocks: Cartesian body-keypoint positions, local motor angles, and root commands such as linear velocity, height, pitch, and yaw. Binary masks accompany the values and specify which coordinates are authoritative. The policy also receives robot proprioception and produces joint-position targets for PD control. Joystick, MoCap, RGB pose, VR head/hands, exoskeleton, or robot-arm interfaces differ mainly in how they populate this command-and-mask vector.

The oracle imitator is first trained from full kinematic motion, where the desired completion is unambiguous. Multi-mode distillation then samples masks and teaches the student to recover the oracle action from partial goals. This is more than generic sensor dropout: removing a command creates a null space, and the teacher supplies a stable, human-like completion of that null space. Curriculum over modes is essential because dense imitation, root-velocity walking, and a few upper-body angles have very different loss scales and difficulty.

The highlight is the clean abstraction: downstream policies need not relearn balance, and interfaces can switch modes at runtime. The limitation is that HOVER completes missing information from its motion prior, not from scene understanding. A plausible free arm may still hit a table, and two active modalities may conflict. Useful extensions include command confidence and priority, collision/contact inputs, explicit training on rapid mask changes, and a feasibility head that predicts fall risk before accepting a command. Tests should include adversarial command combinations, asynchronous devices, and transient error after mode switches.

### Control-stack accounting and a stronger evaluation protocol

For HOVER: Versatile Neural Whole-Body Controller for Humanoid Robots, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

## Humanoid-GPT: Scaling Data and Structure for Zero-Shot Motion Tracking

Humanoid-GPT frames tracking as structured sequence modeling over large-scale robot motion rather than only reactive MLP control. A transformer-style policy exploits temporal context and motion tokens/structure to generalize zero-shot to unseen references. Scaling data and model capacity improves coverage, while physics/RL training grounds outputs. The language-model analogy is useful for long context and reuse, but autoregressive/transformer latency, rare contact dynamics, and out-of-distribution commands still require safeguards.

### Dataset and expert-to-generalist training

The system assembles and strictly filters public mocap plus in-house recordings, removes scene-dependent motions such as swimming or unsupported sitting, retargets everything to G1, and augments it to roughly two billion motion frames. Rather than ask one untrained Transformer to discover all control modes directly with PPO, motions are clustered and hundreds of PPO experts learn keypoint-level tracking rewards on narrower distributions. Parallel DAgger then distills their state–action behavior into one causal generalist.

The deployable input combines reference joint/keypoint motion with robot joints, root angular velocity, projected gravity, and previous action. The policy predicts per-joint commands that PD control converts to torque. A history of observation/reference tokens enters a GPT-style causal Transformer, so every action can use past motion without leaking future measured robot state. Causal attention also permits efficient batched sequence distillation.

### Scaling evidence and judgment

Experiments vary corpus and capacity up to an approximately 80M-parameter Transformer and compare similarly sized MLP/non-causal alternatives. MLP performance saturates as data grows, whereas the causal model continues improving on unseen and online-retargeted dynamic behavior. Dataset balance remains necessary: indiscriminate scale can overrepresent easy locomotion and erase rare agility.

The useful result is a recipe—physics experts provide stable labels and a sequence model absorbs them at scale—not evidence that autoregression alone learns dynamics. Distillation inherits expert blind spots, filtering removes many interactions, and a large policy is difficult to certify. Future work should publish tail failures, history/latency ablations and expert disagreement; add scene/contact tokens; and use uncertainty to reject infeasible online references.


### Exact representation, supervision, and deployment implications

The corpus merges AMASS, LAFAN1, MotionMillion, and PHUMA, maps the result to the 29-DoF G1, removes explicitly scene-dependent clips, and time-warps each sequence at several speeds to expand the corpus by about fivefold. Harmonic Motion Embedding clusters motions using periodic amplitude/frequency descriptors learned by Periodic Autoencoders; this groups similar dynamics more meaningfully than static pose distance. Experts track weighted body keypoints—arms, hips, feet, pelvis—with exponential position, SO(3) orientation, and velocity rewards plus self-contact and smoothness penalties. Experts that cannot sustain accurate long rollouts are excluded from teacher supervision.

During DAgger, each Transformer token concatenates current proprioception and current reference pose. A causal window of `H` tokens is processed at once, and every token position is supervised with the corresponding routed expert action using Smooth-L1 loss. This provides many action labels per forward pass during training; at runtime only the last output becomes the present control target. It also explains why early-episode behavior can work with an incompletely filled history—the model saw positions with different available causal context during training.

The reported model scale reaches roughly 80 million parameters and the underlying retargeted corpus about two billion frames. Such scale is meaningful only with a control-compute audit: CPU and ONNX inference are reported in the several-millisecond range, but end-to-end latency also includes retargeting, state estimation, PD communication, and history-buffer timing. A useful production extension would distill the Transformer into latency tiers and select the smallest model whose tail-error and recovery statistics meet a hardware-specific safety budget.

### Foundation-model criteria, latent coverage, and deployment accounting

For Humanoid-GPT: Scaling Data and Structure for Zero-Shot Motion Tracking, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

A behavioral foundation model should document the source motion hours and filtering, how kinematic observations become physical robot rollouts, replay or on-policy transition count, network and latent dimensions, temporal horizon/history, privileged training inputs, deployable inputs, action representation, control rate, and total simulation compute. Model size alone is not scale when larger policies receive more environment interaction. Data and compute-normalized curves should vary capacity, behavioral diversity and optimization separately, with several seeds and per-family results.

Latent quality requires closed-loop tests. Reconstruction or action likelihood can be high while decoded contacts drift and the robot falls. Report rollout survival, pose and velocity error, contact timing, diversity, latent utilization or collapse, interpolation behavior, prompt sensitivity, inference latency and hardware reliability. New goals, demonstrations or rewards should be structurally held out, not merely different clips from the same corpus. Compare frozen latent control, weight fine-tuning, task-specific RL, retrieval and random/optimized latent search at equal downstream interactions. A coverage estimator should predict when a requested behavior lies outside the pretrained occupancy.

The practical control stack also needs an authority boundary. A high-level prompt or planner selects latent behavior; the decoder produces joint targets; servo and safety layers constrain execution. Prompt computation, denoising or optimization steps count toward latency, and stochastic samples need temporal consistency. “Zero shot” must state what remains optimized after prompting. My judgment is that foundation status is earned by reusable improvement across different command interfaces and tasks with predictable failure, not by one very broad tracker. Cross-embodiment transfer, uncertainty-based abstention, contact/terrain perception and shielded latent planning are the decisive next tests.

## Humanoid Manipulation Interface: Humanoid Whole-Body Manipulation from Robot-Free Demonstrations

Source: [arXiv 2602.06643](https://arxiv.org/abs/2602.06643).

HuMI gathers portable whole-body demonstrations without a robot and converts them into executable humanoid skills. Across kneeling, squatting, tossing, walking, and bimanual tasks it reports three-times higher collection efficiency than teleoperation and 70% success in unseen environments.

Portable human-worn capture records body motion and manipulation. A hierarchical learning pipeline separates high-level task/motion prediction from low-level feasibility/control, bridging human and humanoid embodiments rather than copying human joints directly.

The collection rig captures human whole-body and manipulation signals; learned execution uses task observations plus robot state. Public summary material does not establish every sensor or layer dimension, so those details should be checked in the full paper before reproduction.

Robot-free collection scales beyond available hardware and avoids robot damage, but embodiment/contact mismatch becomes the core problem. Success depends on accurate spatial calibration and a strong feasibility layer.

### Hierarchical execution and action continuity

Portable capture records full human body/end-effector motion and scene observations without occupying a robot. A low-level PPO tracker learns body and end-effector rewards, with end-effector weight ramped only after stable whole-body motion. Variable-speed reference augmentation repeatedly slows difficult portions so the robot can correct precision error instead of chasing a fixed clock.

The high-level Diffusion Policy consumes task vision and predicts chunks of relative keypoint trajectories, normally two grippers plus base (optionally feet). Two interface choices address hierarchy mismatch. New chunks are anchored to the previous *scheduled target*, not the lagging executed pose, avoiding a reversal at chunk boundaries. For visually ungrounded pelvis/feet, the controller tracks within-chunk relative transforms rather than drifting absolute world poses; wrist cameras can anchor grippers.

Five tasks cover kneeling, squatting, throwing, walking and bimanual coordination; collection is about three times faster than robot teleoperation and unseen-environment success is reported at 70%. The tradeoff is that human contact/force and geometry are not embodiment aligned. Target anchoring preserves momentum but can ignore growing robot error. Future work should blend scheduled and measured state by tracker confidence, add tactile/object force, and retain multiple feasible robot realizations rather than one retarget.


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

## KungfuBot2: Learning Versatile Motion Skills for Humanoid Whole-Body Control

Source: [arXiv 2509.16638](https://arxiv.org/abs/2509.16638).

VMS combines a hybrid objective for local pose fidelity and global trajectory consistency with an Orthogonal Mixture-of-Experts that encourages specialist diversity. A segment-level reward relaxes rigid per-frame matching, improving long-horizon stability under displacement or temporary error. G1 experiments include sports, dance, kung fu, locomotion, minute-long sequences, and unseen motion. Segment tolerance improves recovery but may reduce exact timing; MoE routing/capacity and reference feasibility remain central.

### Hybrid tracking and orthogonal experts

The PPO actor receives robot proprioception plus local/future reference motion and outputs PD joint targets. Local body/joint terms preserve expressive pose while root/world trajectory terms prevent the drift that purely local tracking tolerates. Segment-level matching evaluates progress over a temporal neighborhood rather than insisting on one exact frame, letting the policy resynchronize after pushes or transient phase error.

OMoE increases capacity but adds an orthogonality objective so experts learn different feature/action subspaces instead of collapsing to the same solution. Soft routing blends them inside one policy. Long sequences and held-out motions test whether specialization supports composition rather than memorizing clips.

The segment reward is the important robustness device, but temporal tolerance can select an easier nearby frame and hide timing/contact error. Orthogonal parameters do not guarantee semantically meaningful skills, and router switching can jitter. Future work should visualize expert roles, measure phase/contact deviation separately, condition tolerance on event criticality, and add feasibility/contact sensing for interaction rather than free-space demonstrations.


### Why local/global tracking and temporal segments complement one another

VMS addresses two different failure modes. Root-relative pose rewards can preserve the recognizable style of a kick while the robot drifts away from the intended path; strict world-frame tracking prevents drift but can make every later target inconsistent after a small displacement. The hybrid objective keeps local joint/body geometry for motion identity and global root/body trajectory for spatial intent. Segment-level reward softens the timing correspondence: nearby reference frames can receive credit while the robot catches up after a push, instead of demanding an impossible instantaneous return to frame `t`.

The Orthogonal Mixture-of-Experts tackles interference inside the actor. A router blends expert transformations, while an orthogonality regularizer discourages their features or parameters from becoming identical. One expert can support running and flight phases while another represents low, contact-rich martial-arts poses, yet soft routing permits composition in minute-long sequences. The controller remains model-free PPO conditioned on proprioception and reference motion, producing desired joint positions for PD servos; OMoE changes policy capacity, not the control paradigm.

The strongest result is sustained execution of a broad dynamic repertoire by one real G1 policy, not merely lower mean joint error. Still, parameter orthogonality is only a proxy for behavioral diversity. Router entropy, expert use by skill and phase, switch frequency, and expert-removal tests would show whether specialization is real. Foot strike, landing, and racket contact should not receive the same temporal tolerance as decorative arm motion. A successor could infer phase uncertainty, use contact-conditioned tolerance, penalize router chattering, and add terrain/object observations for genuinely closed-loop interaction.

### Control-stack accounting and a stronger evaluation protocol

For KungfuBot2: Learning Versatile Motion Skills for Humanoid Whole-Body Control, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

## LIMMT: Less is More for Motion Tracking

LIMMT argues that simpler, carefully selected observations/rewards and cleaner references can outperform increasingly complicated trackers. It reduces redundant privileged signals and tracking terms, focusing optimization on deployable proprioception and task-critical kinematics. The streamlined RL pipeline improves robustness, training efficiency, and sim-to-real behavior. The lesson is methodological: complexity can hide reference/data problems. Minimal signals may, however, omit context needed for contact-rich, terrain-aware, or ambiguous sparse-command tasks.

### Global Quality Sampling

The paper's “less” primarily means less but better-selected training data. Motions are screened for floating, penetration, unsupported poses, and severe reconstruction/retargeting artifacts. A Periodic Autoencoder embeds temporal windows as amplitude, frequency, phase, and offset instead of one static code, so distance captures dynamics while tolerating phase shifts.

Global weighted farthest-point sampling then builds a core set covering that space. Candidate weights combine diversity with motion complexity measured from acceleration/abruptness, preventing repeated easy walking from occupying every slot while preserving rare dynamic examples. A 10% AMASS subset outperforms the full corpus; cross-dataset selected training reports 92.8% success, and a highest-quality subset reaches 94.6%, comparable to full data with far less rollout conflict.

### Interpretation and limits

More clips also mean more contradictory gradients and broken supervision. Curation improves signal-to-noise for fixed policy capacity, and plug-in tests across trackers show the gain is not one architecture's trick. But embedding coverage is not universal coverage: complexity can favor jerky artifacts, global diversity can miss contact variants, and selection optimized for one robot/task may discard useful styles. Future work should estimate each clip's downstream marginal value, incorporate contact/torque feasibility, audit removed modes, and recompute the subset when embodiment or target task changes.


### Selection algorithm, baseline controller, and what the result does not imply

LIMMT first learns a Periodic Autoencoder so each clip is represented by joint-wise periodic amplitude, phase/frequency, and offset rather than an arbitrary pooled pose. A complexity score measures movement abruptness/acceleration. Global Quality Sampling begins from a seed and repeatedly adds candidates far from the selected set in the learned space, with quality/complexity weighting; unlike per-cluster quotas, each choice is conditioned on coverage of the complete selected set. The output is a ranked subset, so a user can select 5%, 10%, or another compute budget without retraining the embedding.

The policy comparison is deliberately conventional: PPO with a three-hidden-layer actor/critic, reference tracking terms, physical/action regularization, domain randomization, and PD joint targets. Keeping the tracker ordinary isolates the value of data selection. The striking 10%-versus-100% result therefore says that this corpus contains redundancy and harmful artifacts under a fixed optimization budget; it does **not** prove that more clean, balanced data would hurt, or that a larger policy with more updates could not exploit it.

Quality should also be assessed by the behaviors the subset removes. A high overall success rate can improve while rare kneeling, get-up, asymmetric contacts, or stylistic gestures vanish. A stronger evaluation would publish per-motion-family recall, contact diversity, torque/impact distributions, and scaling curves where environment transitions—not epochs—are equal. Iterative selection using actual policy learning progress could then replace static geometric complexity with measured marginal control value.

### Control-stack accounting and a stronger evaluation protocol

For LIMMT: Less is More for Motion Tracking, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

## M3imic: Learning a Versatile Whole-Body Controller for Multimodal Motion Mimicking

M3imic unifies motion goals from several modalities—full mocap, sparse keypoints, upper-body commands, or other partial references—inside one controller. Modality encoding and masking/fusion map them to a common tracking representation; RL and teacher/student training enforce physical execution. This supports interface switching and missing signals without policy replacement. Shared representation improves reuse, but modality calibration and conflict resolution are difficult, and a universal controller may trail specialists on extreme motions.

### Common latent command and control policy

M3imic constructs synchronized robot-joint, human-pose, and end-effector-transform views. Short temporal horizons from each pass through separate MLP encoders into aligned latent commands. Decoders reconstruct their own and cross-modal targets; reconstruction and latent-consistency losses place different observations of one motion nearby. During RL, one or more latents can be supplied or masked while the same actor receives latent plus deployable proprioception.

The actor uses joints/inertial state and previous action; an asymmetric critic additionally receives root linear velocity and environment/reference information. PPO outputs PD joint-position targets. Rewards track body position/orientation and velocity with action/physical regularization. Noise, pushes, and parameter randomization support transfer. Difficulty-aware segment sampling revisits termination failures but caps extremely hard segments so impossible references do not dominate.

### Evidence and judgment

One hardware policy follows all three command types and reports 98.42% success on unseen OMOMO. This supports modality reuse, but equivalence is underdetermined: sparse end effectors omit posture/contact timing, so a deterministic aligned latent may average valid alternatives. Conflicting modalities also need explicit priority. Future work should predict distributions and confidence, expose masks to the actor, test mid-episode sensor dropout/switching, and verify whether latent semantics transfer across embodiments and contact-rich tasks.


### Dimensions, temporal horizon, and latent-learning losses

The three command spaces are deliberately heterogeneous: robot reference is 29 joint angles; human reference uses 21 SMPL-X joints represented by 6-D rotations; sparse teleoperation uses five SE(3) poses—hands, feet, and chest—encoded by 3-D root-relative position plus 6-D rotation. Each encoder reads ten samples spaced by two control frames (`H=10`, `Δ=2`) and produces a 64-D latent. Separate MLP decoders support three simultaneous objectives: same-modality reconstruction, pairwise latent alignment, and cross-modal consistency measured by decoding different modality latents through a shared robot decoder.

The deployed actor concatenates a command latent with root rotation error, angular velocity, joint positions/velocities, and previous action. It intentionally omits global root position and linear velocity so external localization is unnecessary. The asymmetric critic adds root-position error, 42-D link positions, 84-D link orientations, linear velocity, and other simulator state. Domain randomization is broad—ground friction 0.1–1.6, pushes up to 0.5 m/s, COM/mass/motor variations, and encoder/IMU noise—because missing modalities and hardware uncertainty are separate robustness problems.

Failure-aware segment sampling mixes uniform probability with clipped failure probability. Clipping matters: without it, a few impossible references monopolize training and destabilize the shared latent. For stronger evidence, the paper should report modality-conflict cases and performance as references become progressively sparse, delayed, or asynchronously updated. A probabilistic decoder could represent multiple lower-body completions compatible with the same five end-effectors instead of hiding ambiguity in PPO exploration.

### Control-stack accounting and a stronger evaluation protocol

For M3imic: Learning a Versatile Whole-Body Controller for Multimodal Motion Mimicking, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

## Make Tracking Easy: Neural Motion Retargeting for Humanoid Whole-body Control

The paper jointly addresses retargeting and control by learning a neural mapping from human motion to robot-compatible references rather than relying only on per-frame IK. Temporal/context features capture embodiment constraints and produce smoother, more trackable motion; the downstream RL policy closes dynamics. Learning retargeting can amortize expensive optimization and improve task success. Its training pairs/teacher determine generalization, and opaque mappings may violate contacts or limits on unusual bodies/motions.

### Physics-grounded paired data

The pipeline filters raw SMPL for physical support/reconstruction quality and applies conventional kinematic retargeting. Expert tracking policies execute clustered references in simulation; successful rollouts become about 30,000 physically consistent human–robot sequence pairs. A large noisy kinematic set supplies broad pretraining, while this smaller rollout set shifts output toward motions the robot can execute.

Human input is a 272-D per-frame representation containing root planar motion/orientation and local joint positions/velocities. An autoregressive CNN–Transformer extracts short patterns and long context, then decodes robot root and joints. Regression pretraining uses kinematic retargets; physics-pair fine-tuning suppresses skating, penetration, and dynamically marginal poses without iterative IK for each new sequence.

### Tracker relationship and assessment

The expert PPO sees reference state, body/joint pose and velocity, projected gravity, and previous 29-D action. Tracking rewards dominate; a curriculum progressively tightens error scales. Thus supervision comes from closed-loop trajectories, not IK alone.

Amortization enables online or massive-corpus processing, but the network can silently replace unusual semantics with familiar feasible motion. It outputs a reference, not a feasibility certificate for changed hardware. Future work should predict contact/confidence, reject out-of-distribution inputs, include torque/scene interaction, and retain constraint projection for unsafe frames. Source fidelity and downstream survival must be reported separately so “easy” does not merely mean over-smoothed.


### Network shape, 509-D expert state, and curriculum details

The expert that creates physics pairs is intentionally information-rich. Its 509-D observation includes reference joint position/velocity; 14 reference link positions, orientations, linear velocities, and angular velocities; the corresponding realized body quantities; relative joint state; and previous 29-D action. Tracking rewards cover anchor/root and all link position, rotation, and velocity, while only small action-rate and undesired-contact penalties regularize the policy. An adaptive reward-width schedule starts permissive and progressively tightens exponential tracking tolerances, allowing coarse acquisition before demanding high precision on a large motion cluster.

The retargeter is not a per-frame MLP. A Conv1D/ResNet downsampler extracts local temporal patterns, a bidirectional Transformer provides whole-sequence context, and an upsampling Conv1D decoder returns robot motion. Kinematic pairs pretrain this architecture for broad coverage; about 30,000 SMPL-to-successful-rollout pairs then fine-tune it toward the physical robot distribution. Training uses long temporal windows (the reported setup samples motion at 120 Hz), allowing it to remove brief pose-estimation jitter and infer transitions from context.

Bidirectional context makes offline retargeting strong but complicates truly causal teleoperation: future frames are unavailable unless latency or a prediction buffer is introduced. The work should therefore separate offline and streaming settings and report receptive-field delay. A causal student distilled from the bidirectional model, with an explicit contact head and constraint-projection layer, would preserve much of the smoothing while producing auditable, low-latency references.

### Control-stack accounting and a stronger evaluation protocol

For Make Tracking Easy: Neural Motion Retargeting for Humanoid Whole-body Control, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

## OmniXtreme: Breaking the Generality Barrier in High-Dynamic Humanoid Control

[OmniXtreme](https://arxiv.org/abs/2602.23843) first learns broad motion with a high-capacity flow-matching policy, then performs actuation-aware physical refinement for hardware. Decoupling representation scaling from sim-to-real refinement preserves fidelity on diverse extreme motions. Sampling/control complexity and actuator-specific tuning still affect real-time reuse.

### Specialist distillation with flow matching and residual refinement

Motion-specific expert trackers first solve heterogeneous high-dynamic clips. DAgger rolls out a unified conditional policy and queries the relevant expert at learner-visited states. Instead of regressing the mean expert action, a flow-matching network learns a velocity field that transports Gaussian noise to the expert action conditioned on proprioception/reference observation; a few Euler steps generate the joint target. The generative action representation retains distinct solutions that an MLP would average when the motion set becomes diverse.

After distillation, the base model is frozen and a residual RL policy corrects actions under realistic actuator power, torque, regenerative-energy, and other hardware constraints. Separating representation learning from physical refinement avoids interference-heavy joint multi-motion PPO. One G1 policy reportedly executes flips, acrobatics, martial arts, and breakdance-like motions.

The highlight is identifying both representation-capacity and physical-executability bottlenecks. Still, expert quality bounds the student, iterative flow sampling consumes a tight control budget, and residual correction can hide an infeasible base rather than fix it. Reporting inference jitter, power/thermal margin, falls and hardware wear is crucial. Distilling the flow, uncertainty-aware rejection, and planning transitions among extreme skills are natural next steps.

### Generative action sampling, specialist coverage, and hardware refinement

The flow model is trained on state-conditioned expert actions collected with DAgger, so its target distribution includes corrective states visited by the unified learner rather than only clean specialist rollouts. Starting from Gaussian noise, a learned velocity field integrates toward an action compatible with proprioception and motion reference. Number of integration steps trades latency against action quality; fixing or correlating noise across time may reduce joint-command jitter. The architecture and control loop must disclose solver choice, step count, action variance, and worst-case inference time.

Specialists solve the representation problem only if their union covers transitions and recovery states. A flip expert and dance expert can prescribe distinct actions for superficially similar states; flow matching can retain multimodality where MSE BC averages. But at runtime one sample is executed, and stochastic mode switching can destabilize contacts. Motion conditioning, temporal noise consistency, or a short action sequence can preserve intent. Compare flow, Gaussian policy, deterministic MLP, diffusion, and runtime MoE at matched parameters and expert data.

Residual RL then targets actuator-aware discrepancies with the base frozen. Reward and constraints should include torque-speed envelope, voltage/current, regenerative power, temperature proxy, impact and joint limits, not only pose. Report how much residual authority is used and whether the base alone is already infeasible. Extreme hardware results require repeated trials, aborted attempts, wear, and fall-protection disclosure. Uncertainty from flow likelihood or expert disagreement should gate deployment; an extreme reference outside coverage should be slowed or rejected rather than “refined” after an unsafe base command.

### Control-stack accounting and a stronger evaluation protocol

The decisive chain is specialist skill acquisition → DAgger state coverage → flow-matched action distillation → actuator-aware residual refinement. This paper should be judged on whether multimodal action capacity survives real-time sampling and whether refinement respects hardware energy limits.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

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

OmniH2O uses kinematic pose as a universal interface for VR, verbal, and RGB-camera control of a full-size humanoid with dexterous hands. It demonstrates sports, object handling, human interaction, autonomous policies, and releases OmniH2O-6 with six tasks and paired first-person RGB-D, commands, and whole-body actions.

Large human motion corpora are retargeted/augmented in simulation. A privileged teacher learns robust whole-body tracking; a sparse-observation student is distilled for real deployment. The common kinematic goal allows different front ends—or a frontier model—to command the same controller. Teleoperated trajectories train downstream imitation policies.

Depending on mode, goals come from VR, RGB human pose, language/model output, or stored trajectories. The robot combines sparse proprioception with an egocentric RGB-D stream for autonomous tasks.

A universal pose API is a strong systems abstraction. Its safety and fidelity are ultimately bounded by goal quality, retargeting, hand control, and the low-level tracker's training coverage.

### Sparse command and downstream dataset

The deployable body command is head plus two hand positions/velocities. A student also sees 25 steps of joints, joint velocity, base angular velocity, gravity and prior actions; a privileged PPO teacher sees full body/reference state across 14,000 augmented AMASS clips. DAgger produces PD joint targets. Standing/squatting augmentation and a scheduled foot-height term shape the unconstrained legs; fingers are handled separately with VR-driven IK.

The same motor API accepts VR, camera pose, stored or model-generated goals and supplies the OmniH2O-6 egocentric RGB-D/action dataset. Reported student success (94.1%) nearly matches the teacher (94.77%), but three points cannot specify elbow/foot collision or force. A reachability/uncertainty response, variable-cardinality goals, tactile fingers and scene-aware null-space constraints should accompany any upstream autonomous generator.


### Sparse motor API

The deployable goal is only head position plus two hand positions/velocities. The student also receives 25 steps of joint positions/velocities, root angular velocity, projected gravity, and previous actions; it does not receive global linear velocity. A PPO teacher sees full rigid-body state and reference error over 14,000 augmented AMASS sequences. DAgger transfers teacher actions to the history-based student, which outputs PD joint-position targets. Fingers remain a separate VR-to-IK path rather than learned contact control.

History makes unobserved velocity/contact partially inferable, while the learned prior fills feet, pelvis, and elbows left unspecified by three points. Dataset balance determines that null-space behavior: standing and squatting augmentation prevents a manipulation policy from satisfying hand targets by constantly shuffling. A scheduled foot-height reward distinguishes intended steps from stationary tasks. Reported simulation success of 94.1% is close to the teacher's 94.77%, evidence that deployable history retains most task-relevant information.

The framework should be viewed as a robust command interface, not scene understanding. Any VR, RGB, or language module must still produce reachable safe points; the tracker has no knowledge of an unseen table and sparse commands can conflict. Variable-cardinality goals, reachability/uncertainty prediction, collision perception, tactile fingers, and cross-robot testing are natural extensions. Foot motion during stationary hand tasks and responses to adversarially fast commands should accompany aggregate success.


### Information bottleneck and downstream autonomy

OmniH2O's unusual design choice is to reduce the live whole-body command to three Cartesian points—head and two hands—while giving the student a 25-step proprioceptive history. This makes the policy usable with commodity VR and monocular pose systems, but it also makes lower-body behavior fundamentally prior-driven. The teacher can exploit full rigid-body/reference state during PPO; DAgger then labels states actually visited by the partially observed student, preventing a purely offline distillation set from missing its own balance errors. The history encoder acts as a crude state estimator for base translation, contact, and target velocity that are absent from a single observation.

Kinematic pose is also the boundary between low-level control and autonomy. GPT-4o, verbal instructions, an RGB shadowing module, stored trajectories, or an imitation policy need only propose the same goal representation; none directly commands torque. OmniH2O-6 records paired first-person RGBD, control goals, and whole-body motor actions for six everyday tasks, so autonomous policies can be learned above the stable tracker. Dexterous fingers, however, use a separate mapping/IK path, and neither the sparse tracker nor the high-level model inherently reasons about grasp force.

This modularity is a highlight because failures can be localized to perception/planning versus balance. It can also conceal ambiguity: identical hand/head paths may require stepping, leaning, kneeling, or remaining planted depending on objects and intention. Future versions should condition the completion on scene geometry, explicit contact mode, and uncertainty, and should evaluate multiple valid completions rather than one mocap match. Latency sweeps, history-length ablations, operator-command bandwidth, foot-slip/contact metrics, and intervention rates would better quantify teleoperation quality than whole-body success alone.


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

## OmniTrack: General Motion Tracking via Physics-Consistent Reference

OmniTrack centers reference construction and filtering around what the robot can physically execute. Motion is retargeted, contact/ground consistency is enforced, and infeasible segments are refined or down-weighted before general tracker training. Curriculum/resampling then covers a broad skill distribution. The approach reduces the conflict between fidelity rewards and balance. “Physics-consistent” references still depend on model accuracy and may be feasible in simulation but unsafe or torque-limited on hardware.

### Physical reference generation before robustification

OmniTrack separates two stages. A privileged generation policy first tracks raw human references with full state, no observation noise, and no domain randomization. Its successful physics rollouts remove floating, penetration, and embodiment-induced artifacts while preserving timing/style. Contact rewards and reference-velocity-dependent smoothing retain deliberate contacts without jittering slow sequences.

The final general tracker trains on these realized trajectories using onboard joint pose/velocity, root orientation/angular velocity, and previous action, then outputs joint-position targets. Noise and extensive dynamics randomization are enabled only here for transfer. The main policy uses a single current frame; comparisons include DAgger students with and without a five-frame history and asymmetric actor–critic variants.

### Evidence and critical reading

On raw LAFAN1, increasing data from one-eighth to full scale lowers success from 95.1% because infeasible references create gradient conflict. Physics-consistent data degrades much less as scale grows. Full LAFAN1 plus roughly eight hours of dynamic/contact-rich CMU motion supports G1 execution and noisy teleoperation.

Making feasibility a dataset property is compelling, but the generator can simplify hard motion while survival hides semantic loss. Feasibility is simulator- and embodiment-specific and assumes no unexpected scene. Future work should measure semantic/contact change from source to rollout, attach uncertainty, include thermal/impact constraints, and adapt references online when terrain/object state differs.


### Stage-specific observations, contacts, and quantitative trade-off

Stage I deliberately receives full simulator state—global link poses/orientations/velocities, contacts, and other privileged quantities—and uses a generous early-termination threshold. It applies command noise but no dynamics randomization, because its job is to realize as much of the source as possible under the robot's nominal torque limits, not to learn hardware robustness. The stored reference includes joint state, global link kinematics, and contact state, making later contact supervision internally consistent.

Stage II is restricted to `q`, `qdot`, root orientation, root angular velocity, and previous action. Desired-contact rewards discourage the tracker from matching pose while using the wrong supports. Action smoothness is scaled by reference joint speed—strong in slow poses, relaxed in fast phases—avoiding a single regularization coefficient that either jitters standing or blunts martial arts. Friction, restitution, joint properties, IMU bias, center of mass, observation noise, and external pushes are randomized here. Physics runs at 200 Hz and control at 50 Hz.

The reference filter eliminates reported penetration/floating in both LAFAN1 and AMASS subsets, while introducing approximately 21 mm and 16 mm MPJPE respectively. That is the core trade: modest geometric deviation buys contact feasibility and much better learning scalability. Evaluation should additionally measure which body parts change, whether foot-contact timing shifts, and how often the generation policy fails to produce any rollout. Those statistics would distinguish a faithful physical projection from selective deletion of difficult semantics.

### Control-stack accounting and a stronger evaluation protocol

For OmniTrack: General Motion Tracking via Physics-Consistent Reference, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

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

## Perceptive Behavior Foundation Model: Adapting Human Motion Priors to Robot-Centric Terrain

This truncated entry refers to adapting a behavior foundation model with robot-centric perception so broad human-motion priors respond to terrain and scene geometry. A perception-conditioned adapter/residual preserves general skills while modifying contacts. Its promise is scalable reuse; exact bibliographic title should be corrected if full metadata becomes available.

### Terrain-conformal reference adaptation and identity-gated student

The full paper describes Perceptive BFM, a single policy that accepts diverse raw flat-ground motion references, proprioceptive history, and local visual terrain, then outputs residual joint-position targets adapted to local feasibility. Training has four stages: offline terrain-conformal reference synthesis; a privileged teacher trained on adapted references; student distillation using raw references and terrain; and PPO refinement for deployment.

Target-frame action alignment expresses the teacher's effective target relative to the student's unadapted reference, preventing the student from simply copying privileged terrain-conformal commands unavailable at test time. The identity-gated Transformer separates command/proprioceptive motion tracking from zero-initialized terrain intent/action corrections. Initialization therefore reproduces the original flat-ground BFM; terrain branches learn only necessary contact, posture, and timing residuals instead of overwriting broad behavior.

This is a thoughtful anti-forgetting architecture and scales perception across locomotion, style, and dynamic motions. Its limits follow from local terrain sensing and offline adaptation: unseen deformable/dynamic surfaces, depth artifacts, and motions requiring an entirely new contact strategy may exceed residual capacity. Uncertainty-gated fallback, temporal terrain memory, explicit collision/support constraints, and ablations measuring retained flat-ground skill versus terrain gain should be emphasized.

### Residual perception path, terrain supervision, and retention

Perceptive BFM deliberately initializes the terrain pathway near identity or zero, making the adapted network reproduce the original flat-ground behavior before learning residual corrections. Raw reference motion and proprioceptive history feed the frozen or protected motion backbone; local terrain features enter separate Transformer tokens/branches; gating injects intent and action residuals only when geometry demands them. Target-frame action alignment expresses privileged teacher behavior relative to the unadapted reference, giving the student a learnable correction rather than an unavailable terrain-conformal target.

Offline terrain-conformal synthesis should preserve motion semantics while adjusting foot height, timing, root posture, and clearance. It needs contact labels, kinematic limits, smoothness, collision, and support checks. The privileged teacher tracks these feasible targets with full terrain/contact state. Distillation teaches the visual/onboard student from raw goals, and PPO refinement closes remaining error. Exact terrain sensor, spatial extent/resolution, temporal history, actor output, and control rates are central reproducibility details.

Ablations should measure flat-ground retention and terrain gain independently: original BFM, naive full fine-tuning, residual adapters without identity initialization, oracle terrain, and student perception. Break results down by stairs, slopes, discrete footholds, overhangs, and dynamic motions; report foot collision/slip, body clearance, tracking, falls and latency. Residual capacity may be insufficient when a motion requires a new contact sequence rather than a small deformation. An uncertainty head should slow or reject poor terrain observations, while temporal mapping and online feasibility optimization could handle occlusion and surfaces beyond the local frame.

### Foundation-model criteria, latent coverage, and deployment accounting

The decisive chain is flat-ground motor prior → terrain-conformal supervision → identity-initialized perceptive residual → PPO refinement. Retention of the original repertoire is as important as terrain improvement.

A behavioral foundation model should document the source motion hours and filtering, how kinematic observations become physical robot rollouts, replay or on-policy transition count, network and latent dimensions, temporal horizon/history, privileged training inputs, deployable inputs, action representation, control rate, and total simulation compute. Model size alone is not scale when larger policies receive more environment interaction. Data and compute-normalized curves should vary capacity, behavioral diversity and optimization separately, with several seeds and per-family results.

Latent quality requires closed-loop tests. Reconstruction or action likelihood can be high while decoded contacts drift and the robot falls. Report rollout survival, pose and velocity error, contact timing, diversity, latent utilization or collapse, interpolation behavior, prompt sensitivity, inference latency and hardware reliability. New goals, demonstrations or rewards should be structurally held out, not merely different clips from the same corpus. Compare frozen latent control, weight fine-tuning, task-specific RL, retrieval and random/optimized latent search at equal downstream interactions. A coverage estimator should predict when a requested behavior lies outside the pretrained occupancy.

The practical control stack also needs an authority boundary. A high-level prompt or planner selects latent behavior; the decoder produces joint targets; servo and safety layers constrain execution. Prompt computation, denoising or optimization steps count toward latency, and stochastic samples need temporal consistency. “Zero shot” must state what remains optimized after prompting. My judgment is that foundation status is earned by reusable improvement across different command interfaces and tasks with predictable failure, not by one very broad tracker. Cross-embodiment transfer, uncertainty-based abstention, contact/terrain perception and shielded latent planning are the decisive next tests.

## SafeFall: Learning Protective Control for Humanoid Robots

[SafeFall](https://arxiv.org/abs/2511.18509) runs beside the nominal controller: a lightweight GRU predicts unavoidable falls, then an RL policy takes over to shield vulnerable parts and dissipate energy. Damage-aware rewards substantially reduce forces, torques, and critical collisions on G1. False triggers and truly unmodeled impacts remain concerns.

### Failure predictor, takeover policy, and damage objective

Nominal locomotion is perturbed in simulation through pushes, sensor bias/noise, command delay, and other failures to collect diverse safe/falling trajectories. A compact proprioceptive classifier (the paper describes a lightweight single-layer recurrent/hidden architecture) uses IMU and joint encoders to predict an imminent unrecoverable fall. It runs beside the nominal policy and remains dormant until its threshold triggers.

The fall policy then takes over, using onboard proprioception and a 29-D desired-joint action. Its RL reward limits head/fragile-part collision, joint wrench/torque, and damaging impact while encouraging postures that distribute/dissipate energy. This division avoids compromising normal agility with a permanently cautious objective and supports omnidirectional protection on full-size G1 hardware.

The key contribution is treating impact as an optimizable control phase rather than stopping at fall detection. Yet switching creates its own safety problem: false positives interrupt recoverable motion, false negatives lose preparation time, and training damage proxies may not correlate with gearbox/frame fatigue. Calibrated time-to-impact prediction, uncertainty/hysteresis, blended rather than abrupt takeover, hardware load-cell validation, and recovery from the final grounded pose should extend the system.

### Detector calibration, switching dynamics, and damage measurement

Fall prediction should be framed as time-to-unrecoverability, not merely current tilt. A GRU consumes a causal window of IMU orientation/angular velocity, joint positions/velocities, actions and perhaps contact estimates; training labels come from whether the nominal controller later recovers under simulated failures. Decision threshold trades preparation time against false takeover. Precision–recall alone is incomplete: early-warning distribution, calibration, false triggers per operating hour, and missed falls by direction or failure type determine utility.

The protective policy needs exposure to forward, backward, lateral, twisting and obstacle-mediated falls, different floor stiffness/friction, initial gait phase and actuator faults. Its action should shape angular momentum, tuck fragile limbs, avoid head/knee/wrist impact, and spread energy over robust surfaces without exceeding joint torque. Reward proxies need validation against measured contact impulse, peak force, joint wrench, motor/gear load, and structural acceleration. Simulation contact peaks are especially sensitive to timestep and compliance.

Hard switching can create an action discontinuity exactly when the robot is unstable. Hysteresis, blending, or a short optimal handoff can preserve continuity, but waiting too long reduces protection. Compare always-on cautious training, threshold takeover, blended takeover, and oracle trigger. After landing, the policy should reach a safe static pose and hand off to get-up rather than terminate evaluation at first contact. Hardware trials need load sensing or instrumented mats and disclosure of safety rig influence. The paper's important idea is minimizing damage after prevention fails; its next step is an end-to-end fall–protect–recover state machine with calibrated uncertainty.

### Quantifying compliance or safety beyond nominal task success

For SafeFall: Learning Protective Control for Humanoid Robots, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

The complete interface should specify sensed contact information, proprioceptive history, reference or safety command, policy architecture, output action, servo gains, and rates from sensing through actuation. If forces are not measured directly, the note must distinguish inferred disturbance response from actual force regulation. If a theorem or barrier is used, its model, disturbance bound, discretization, perception uncertainty, and feasibility assumptions must be connected to the physical implementation. If safety is reward learned, it is an empirical distributional property rather than a hard maximum.

Experiments should sweep contact point, direction, magnitude, duration, speed, material stiffness, support phase, payload and unseen combinations. Report peak force, impulse, pressure where available, realized directional/frequency-dependent impedance, energy exchanged, joint torque/current, pose or task error, balance loss, falls, false interventions, and recovery time. Comparisons should include a stiff robust tracker, low fixed impedance, ordinary push randomization, and any analytic safety or passivity layer. Human-interaction claims additionally need a controlled comfort protocol and hardware force validation rather than videos alone.

A useful architecture separates three responsibilities: a broad motor policy coordinates posture and stepping, a contact-aware adaptation layer selects yielding or protection, and an independent verified envelope limits force, energy, collision or joint state. Uncertainty should cause conservative slowing or takeover, and infeasible constraints need an explicit priority and emergency behavior. Future work should fuse tactile and force/torque sensing, model distributed multi-contact, calibrate sim contact against hardware, and connect the final contact state to recovery instead of terminating the evaluation at first survival.

Prediction should also be tested during nominal highly dynamic motions, where large tilt and angular velocity resemble a fall but remain intentionally recoverable.

## Safety-Critical Whole-Body Control for Humanoid Robots via Input-to-State Safe Control Barrier Functions

Formulates whole-body safety constraints with input-to-state safe control barrier functions, accounting for bounded tracking/model disturbances rather than assuming perfect dynamics. A quadratic-program safety filter minimally modifies nominal commands to maintain constraints. It offers formal structure, but guarantees depend on correct bounds, feasible simultaneous constraints, and fast optimization.

### Kinematic filter and full-order dynamic realization

KinWBC takes task commands and current configuration and computes nominal joint velocity. An input-to-state-safe CBF quadratic program minimally modifies that reduced-order velocity to maintain collision-distance, joint-limit, and related barrier sets under bounded tracking disturbance. DynWBC then converts filtered position/velocity/acceleration references into full-order torques while enforcing floating-base dynamics, friction cones, center-of-pressure, and torsional contact constraints.

The key theoretical step transfers safety between models. Barriers are convenient at joint-velocity level, but the physical torque-controlled humanoid cannot track perfectly. The mismatch between commanded reduced-order velocity and full-order execution is modeled as an input disturbance; ISSf analysis guarantees a disturbance-dependent enlarged safe set rather than claiming exact invariance. Inputs are task references, robot/environment state, and contacts; outputs progress from safe joint velocity to dynamically feasible torque.

This layered design can wrap different nominal planners and avoids an enormous barrier solve. Guarantees nevertheless require valid disturbance bounds, accurate distance/state estimates, and feasible simultaneous constraints. Sudden perception error or conflicting barriers can break those premises. Online bound estimation, explicit slack/emergency priorities, sampled-data/delay analysis, and swept-volume barriers should follow.

### Constraint hierarchy, sampled-data effects, and infeasibility handling

At the kinematic layer, signed distance to self/environment collision, joint-limit margins, and task-space bounds define barrier functions. The QP seeks a joint velocity close to the nominal task command while satisfying inequalities that keep barrier values from decreasing too quickly under bounded tracking disturbance. DynWBC then solves floating-base equations with contact wrench variables, torque limits, friction pyramids or cones, center-of-pressure and torsional constraints to realize the safe reference. Contact schedule and state estimation are therefore inputs to the guarantee.

ISSf is valuable because it admits nonzero lower-layer error: instead of exact invariance, safety degrades by a calculable margin related to disturbance bound. That bound must cover discretization, communication delay, torque saturation, unmodeled compliance and controller error. If selected from benign logs, the theorem can be formally correct and physically unsafe. Experiments should sweep injected tracking disturbance and compare measured minimum barrier to predicted enlargement, with solver rate and worst-case latency reported.

Multiple constraints can make the QP infeasible—for example, collision avoidance conflicts with balance or torque limits. Slack must have explicit priority and trigger a defined emergency response; silently relaxing all barriers destroys interpretability. Perception uncertainty should inflate obstacles, and continuous swept-volume or sampled-data corrections should cover motion between QP updates. Comparisons with nominal WBC, non-robust CBF, monolithic dynamic CBF, and MPC at matched compute would clarify the benefit of the two-layer formulation. Online disturbance-bound estimation can reduce conservatism, but only with a conservative fallback when confidence drops.

### Quantifying compliance or safety beyond nominal task success

For Safety-Critical Whole-Body Control for Humanoid Robots via Input-to-State Safe Control Barrier Functions, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

The complete interface should specify sensed contact information, proprioceptive history, reference or safety command, policy architecture, output action, servo gains, and rates from sensing through actuation. If forces are not measured directly, the note must distinguish inferred disturbance response from actual force regulation. If a theorem or barrier is used, its model, disturbance bound, discretization, perception uncertainty, and feasibility assumptions must be connected to the physical implementation. If safety is reward learned, it is an empirical distributional property rather than a hard maximum.

Experiments should sweep contact point, direction, magnitude, duration, speed, material stiffness, support phase, payload and unseen combinations. Report peak force, impulse, pressure where available, realized directional/frequency-dependent impedance, energy exchanged, joint torque/current, pose or task error, balance loss, falls, false interventions, and recovery time. Comparisons should include a stiff robust tracker, low fixed impedance, ordinary push randomization, and any analytic safety or passivity layer. Human-interaction claims additionally need a controlled comfort protocol and hardware force validation rather than videos alone.

A useful architecture separates three responsibilities: a broad motor policy coordinates posture and stepping, a contact-aware adaptation layer selects yielding or protection, and an independent verified envelope limits force, energy, collision or joint state. Uncertainty should cause conservative slowing or takeover, and infeasible constraints need an explicit priority and emergency behavior. Future work should fuse tactile and force/torque sensing, model distributed multi-contact, calibrate sim contact against hardware, and connect the final contact state to recovery instead of terminating the evaluation at first survival.

## Scaling Behavior Foundation Model for Humanoid Robots

Studies how model capacity, motion data, compute, and training mixtures affect a humanoid behavioral foundation model. Larger and more diverse training improves transfer and promptability until optimization/physical-feasibility bottlenecks appear. The work provides scaling evidence, while cost, dataset bias, and diminishing returns matter for practical deployment.

### On-policy scaling and Humanoid Transformer

This work argues that humanoid scaling must coordinate three axes: dynamically executable data, an on-policy learning recipe, and an architecture able to use the diversity. Whole-body motion tracking provides a universal proxy task; PPO rollouts are the effective state–action data, while an expanded global-frame motion corpus supplies target diversity. The actor receives deployable proprioceptive history plus goal/reference tokens and emits desired joint angles under domain randomization.

The Humanoid Transformer tokenizes structured groups—robot state, body/reference information, and temporal context—so attention can form body-part and intention structure that a larger flat MLP may not discover. Scaling experiments vary data, parameters, and compute and study whether latent features organize interpretable behavioral intentions and transfer to alternate interfaces.

The lasting contribution is methodological: raw mocap count is not the true data scale because policy rollouts determine visited dynamics, and capacity without motion diversity or stable on-policy optimization gives limited return. Still, scaling curves on one simulator/embodiment can confound parameter count with training budget, and larger attention models raise control latency. Compute-normalized ablations, held-out contact families, hardware reliability curves, and cross-embodiment scaling laws are needed before extrapolating “foundation” behavior.

### Scaling axes, compute accounting, and evidence for emergence

Scaling laws require separating policy parameters, motion/reference diversity, environment transitions, and optimization compute. A larger Transformer trained for the same number of samples may underfit; a larger dataset with fixed updates reduces exposure per clip; more simulated environments changes both data and PPO batch statistics. The study should present controlled curves for each axis plus joint scaling at approximately compute-optimal allocation. Effective data is closed-loop rollout state–action experience, not the nominal hours of mocap alone.

Structured tokens offer a hypothesis for improved scaling: body parts, proprioceptive history, and future/reference elements can attend selectively rather than being flattened. Diagnostics should show attention or probing for phase, support/contact, velocity, style and recovery, while matched-width MLP and recurrent baselines test whether gains are simply capacity. Actor latency, memory, control jitter, and throughput must be measured on onboard hardware because a policy too slow for its servo interface loses practical value.

Held-out evaluation needs motion families, subjects/datasets, contacts, perturbations and hardware, not random frames from the same clips. Per-family success/error, falls, torque/impact, prompt/interface transfer, and robustness scaling reveal whether rare behaviors improve or averages dominate. “Emergence” should mean a capability absent or sharply weaker below a scale threshold under controlled training, not a selected demo. Dataset quality may dominate size when references are infeasible. Scaling should therefore include curation and physics-consistency axes, uncertainty or rejection for out-of-support prompts, and cross-embodiment tests before extrapolating one simulator's trend.

### Foundation-model criteria, latent coverage, and deployment accounting

The decisive variables are executable motion diversity, on-policy environment transitions, architecture capacity, and training compute. Scaling conclusions require controlled curves rather than comparing differently trained checkpoints.

A behavioral foundation model should document the source motion hours and filtering, how kinematic observations become physical robot rollouts, replay or on-policy transition count, network and latent dimensions, temporal horizon/history, privileged training inputs, deployable inputs, action representation, control rate, and total simulation compute. Model size alone is not scale when larger policies receive more environment interaction. Data and compute-normalized curves should vary capacity, behavioral diversity and optimization separately, with several seeds and per-family results.

Latent quality requires closed-loop tests. Reconstruction or action likelihood can be high while decoded contacts drift and the robot falls. Report rollout survival, pose and velocity error, contact timing, diversity, latent utilization or collapse, interpolation behavior, prompt sensitivity, inference latency and hardware reliability. New goals, demonstrations or rewards should be structurally held out, not merely different clips from the same corpus. Compare frozen latent control, weight fine-tuning, task-specific RL, retrieval and random/optimized latent search at equal downstream interactions. A coverage estimator should predict when a requested behavior lies outside the pretrained occupancy.

The practical control stack also needs an authority boundary. A high-level prompt or planner selects latent behavior; the decoder produces joint targets; servo and safety layers constrain execution. Prompt computation, denoising or optimization steps count toward latency, and stochastic samples need temporal consistency. “Zero shot” must state what remains optimized after prompting. My judgment is that foundation status is earned by reusable improvement across different command interfaces and tasks with predictable failure, not by one very broad tracker. Cross-embodiment transfer, uncertainty-based abstention, contact/terrain perception and shielded latent planning are the decisive next tests.

## Robust and Generalized Humanoid Motion Tracking

This work trains a single policy across diverse retargeted references with domain randomization, difficult-motion curricula/resampling, and deployable observations. Robustness is evaluated under perturbations and unseen motions, while teacher/student structure bridges privileged simulation state. It establishes general motion tracking as a reusable low-level interface. Broad averages can hide rare-skill failure; robustness to pushes does not automatically cover terrain, object contact, or impossible commands.

### Dynamics-conditioned reference attention

The actor has a ten-step proprioceptive history—29 joint positions/velocities, root angular velocity, projected gravity, and previous action—encoded causally into a dynamics embedding. That embedding queries a Transformer cross-attention block over a reference command window around the current time. Unlike uniform concatenation, the current inferred dynamics decide which past/future reference tokens matter. The actor fuses the attended command with current state; an asymmetric critic receives clean privileged information.

Actions are corrective offsets around reference joints, converted to PD targets. Dense tracking covers key bodies and joint motion; safety/regularization penalizes rapid commands, limits, contacts, and noise. The compact roughly 3.5-hour dataset mixes mocap, video-derived sequences, and ground interaction. Special recovery environments use assisted initialization and then remove help, adding fall recovery without a separate deployed controller.

Cross-attention most improves contact-rich breakdance/ground motion where timing depends on current dynamics, while causal history is more robust than a CNN replacement under noise. The approach is efficient, but its centered command window presumes indexed future reference and a 10-step history cannot identify every payload/contact state. Future work should expose attention under disturbances, predict feasibility, and test terrain/object deviations rather than only reference and pushes.


### Exact causal encoding and recovery integration

Each of the latest ten proprioceptive samples is mapped by a two-layer MLP to a 128-D token and passed through a causal self-attention encoder. The final history embedding becomes the query for cross-attention; a window of reference poses is separately tokenized as keys/values. This asymmetric query construction is useful: reference frames are not treated equally merely because they are near the current timestamp—the realized dynamics determine which phase cue deserves weight. The resulting command vector, dynamics vector, and current observation feed the PPO actor.

The action is an offset from the current reference joint configuration, not an unconstrained absolute target. This reduces exploration dimension and makes the zero residual meaningful, while PD control provides the actuator interface. During training the critic sees clean global errors/link state while the actor receives noised deployable signals. Fall-recovery environments inject fallen or unstable configurations and initially provide upward assistance; assistance is removed as recovery improves. A state machine is not added at deployment, so recovery and tracking share parameters and command semantics.

The compact 3.5-hour mixture is important evidence against assuming billion-frame scale is always necessary, but its selection may be unusually high-value. Generalization should be reported against source-held-out subjects and motion categories, not random clips. The reference window also makes online teleoperation dependent on prediction or buffering. Causal/no-future ablations, phase-error robustness, recovery under continuing commands, and per-contact-family results would reveal whether attention learns real dynamics alignment or merely corrects small timing jitter.

### Control-stack accounting and a stronger evaluation protocol

For Robust and Generalized Humanoid Motion Tracking, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

## SONIC: Supersizing Motion Tracking for Natural Humanoid Whole-Body Control

SONIC scales motion quantity, diversity, policy capacity, and training infrastructure to create a natural general tracker. A large multi-source motion bank, adaptive sampling/curriculum, and deployable student policy let one controller execute expressive long-tail references in real time. Scaling improves coverage and naturalness, but training cost and data curation grow substantially; empty flat-ground motion does not teach environmental interaction, and the policy can fail on physically incompatible commands.

### Scaled data, representation, and control

SONIC retargets about 700 hours/100M+ frames at 50 Hz spanning 33 categories and scales policies from 1.2M to 42M parameters. Ten steps of joints, joint velocity, root angular velocity, gravity, and previous actions provide proprioceptive context. Specialized MLP encoders accept robot references, human body/keypoint references, or hybrid commands at several future frame intervals and map them into a shared FSQ-quantized latent. The actor outputs PD joint-position targets.

Tracking rewards compare robot and target pose/velocity plus end-effectors; penalties regularize action and contacts. The discrete universal token can be generated by teleoperation, an autoregressive kinematic planner, or a VLA, making the tracker a motor API. Scaling curves on held-out internal sets and PHUMA show that data and capacity together improve zero-shot motion; either alone eventually saturates.

FSQ constrains upstream commands to a learned motion vocabulary but can discard fine contact detail and cannot certify token transitions. The corpus remains mostly empty-scene motion, and the paper notes no formal safety/energy treatment. Future work should publish rare-category failures and code utilization, add interaction/terrain tokens, and pair the interface with reachability and torque/impact guards.


### Multi-format command tokens and downstream interfaces

SONIC's “universal token” is more than a compressed pose. Separate encoders accept native robot joint sequences, human body/keypoint motion, and hybrid sparse/dense specifications, then finite-scalar quantization maps them into a shared discrete code. Proprioceptive history covers ten frames; reference commands include several future offsets so the actor can anticipate foot strike and fast upper-body motion. The policy is trained with PPO and per-frame tracking rather than an adversarial discriminator, avoiding the mode-collapse pressure that grows when one discriminator must explain hundreds of hours.

The shared token provides several control paths. A kinematic autoregressive planner produces tokens for arbitrary velocity/direction and style-conditioned navigation. Live monocular video, text-to-motion, and music-to-motion models can generate human motion that is encoded into the same space. A lightweight VR setup with head and two wrists plus finger/waist information supplies sparse teleoperation. For VLA tasks, the high-level action is reported as 78 dimensions: a 64-D universal motion token plus 14 task-specific dimensions, leaving SONIC to realize coordinated whole-body motion.

Scaling ranges from about 1.2M to 42M policy parameters and uses more than 100M retargeted frames at 50 Hz from an approximately 700-hour source across 33 categories. BONES-SEED exposes a substantial public portion. A proper reading of the result is that model, data, and infrastructure co-scale; comparing the largest model to older small-data baselines cannot isolate any single factor. Important missing diagnostics include FSQ codebook occupancy/perplexity, transition validity, end-to-end latency for every modality, and the proportion of source hours rejected during retargeting. Those would show whether the learned motor vocabulary truly covers the long tail or compresses it into common styles.

### Control-stack accounting and a stronger evaluation protocol

For SONIC: Supersizing Motion Tracking for Natural Humanoid Whole-Body Control, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

## SoftMimic: Learning Compliant Whole-body Control from Examples

SoftMimic learns reference motion together with compliant response instead of maximizing rigid tracking accuracy. Disturbance/contact randomization and impedance-like objectives let the controller yield under external forces while retaining whole-body intent. The result suits human/object interaction better than stiff imitation. Compliance trades pose accuracy for force accommodation; without explicit force sensing/limits, learned softness is distribution-dependent rather than guaranteed.

### Compliance is supervised, not inferred from pushes alone

For an original pose sequence, an offline IK solver applies sampled end-effector wrench and a desired task-space stiffness law to generate feasible displaced configurations. These augmented targets preserve style/balance while encoding how far the robot should yield. During PPO, the actor observes proprioception, the original reference, and a user-selected stiffness; crucially, reward compares the rollout with the *augmented* compliant target. Ordinary disturbance randomization would instead still reward snapping back to the rigid reference.

The policy outputs desired joints for a backdrivable QDD humanoid and learns whole-body balance together with force–displacement response. At deployment, stiffness continuously modulates yielding without retraining. Simulation and hardware pushes show reduced interaction force at low settings and closer pose adherence at high settings.

The paper's key contribution is resolving reward conflict through authored response data. But IK compliance is kinematic and assumes a chosen wrench/application point; learned force behavior is not passive or stable by construction and sensor/model error can change effective stiffness. Future work should add force/tactile feedback, passivity or control-barrier guarantees, multi-contact augmentation, and frequency-dependent tests rather than only quasi-static force–displacement curves.


### What the policy learns, and what “compliance” guarantees

Compliant Motion Augmentation begins from one nominal reference and samples a link, force direction/magnitude, and requested stiffness. A task-space spring law determines the desired displacement; whole-body IK realizes that displaced link while staying close to the original pose and respecting kinematics. The RL goal therefore includes both nominal motion and a stiffness-conditioned compliant target. The actor receives proprioception, reference features, and the stiffness request, then outputs desired joints for the G1's backdrivable quasi-direct-drive actuators. Tracking, balance, and smoothness rewards teach it to realize this response without falling.

This formulation is novel because low force is not obtained by merely reducing PD gains. The learned controller can reorganize the whole body—moving hips, knees, and support—while preserving the example's intent, and a single clip can generalize to variations such as different box sizes or unexpected collisions. Hardware experiments showing reduced contact force while retaining nominal tracking support the central claim that compliance can be a controllable skill dimension.

The word “stiffness” should still be interpreted empirically. The policy is trained to imitate sampled force–displacement examples; it is not an analytically passive impedance controller, does not necessarily reproduce the requested stiffness at every frequency/direction, and can only respond after an unobserved force changes proprioception. Important follow-ups are system-identification plots of the realized 6-D impedance, tests under oscillatory and multi-point contact, energy/passivity bounds, and explicit force/torque or tactile input. A hybrid design could keep a certified low-level impedance envelope while RL selects whole-body posture and distributes compliance within that envelope.

### Quantifying compliance or safety beyond nominal task success

For SoftMimic: Learning Compliant Whole-body Control from Examples, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

The complete interface should specify sensed contact information, proprioceptive history, reference or safety command, policy architecture, output action, servo gains, and rates from sensing through actuation. If forces are not measured directly, the note must distinguish inferred disturbance response from actual force regulation. If a theorem or barrier is used, its model, disturbance bound, discretization, perception uncertainty, and feasibility assumptions must be connected to the physical implementation. If safety is reward learned, it is an empirical distributional property rather than a hard maximum.

Experiments should sweep contact point, direction, magnitude, duration, speed, material stiffness, support phase, payload and unseen combinations. Report peak force, impulse, pressure where available, realized directional/frequency-dependent impedance, energy exchanged, joint torque/current, pose or task error, balance loss, falls, false interventions, and recovery time. Comparisons should include a stiff robust tracker, low fixed impedance, ordinary push randomization, and any analytic safety or passivity layer. Human-interaction claims additionally need a controlled comfort protocol and hardware force validation rather than videos alone.

A useful architecture separates three responsibilities: a broad motor policy coordinates posture and stepping, a contact-aware adaptation layer selects yielding or protection, and an independent verified envelope limits force, energy, collision or joint state. Uncertainty should cause conservative slowing or takeover, and infeasible constraints need an explicit priority and emergency behavior. Future work should fuse tactile and force/torque sensing, model distributed multi-contact, calibrate sim contact against hardware, and connect the final contact state to recovery instead of terminating the evaluation at first survival.

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

## Stubborn: A Streamlined and Unified Reinforcement Learning Framework for Robust Motion Tracking and Fall Recovery for Humanoids

### Recovery emerges from not ending every failure

Stubborn trains one proprioceptive humanoid policy to track a large motion set, withstand pushes, continue from severe tracking error, and stand after falls. Conventional trackers terminate as soon as error crosses a threshold, so they never learn what to do in precisely the states where recovery is needed. Stubborn replaces this with probabilistic soft termination and deliberately resamples difficult reference frames.

### Representation and actor–critic

Reference root pose/velocity, joint/body targets, current joints/inertial state, and phase are expressed in a yaw-aligned frame. Removing absolute world yaw makes tracking insensitive to heading drift while retaining gravity roll/pitch essential to balance. The actor uses deployable proprioception/reference signals; the asymmetric critic also sees contacts, terrain, root state, and domain parameters. It outputs joint targets at 50 Hz.

### Bernoulli termination and sampling

When tracking error is large, the episode terminates by a Bernoulli draw rather than deterministically. With a physical recovery horizon of 4 s at 50 Hz, expected extra exploration is 200 steps. Some failures reset for efficiency, but many continue long enough to discover bracing, rolling, and get-up actions. This avoids a separate recovery-policy switch.

Reference-frame weights rise after poor tracking and fall after success, bounded from 0.05 to 1.0. Simulation focuses on hard transitions without forgetting easy motion. Fallen/perturbed initial states, pushes, terrain, contacts, latency, motors, and body parameters are randomized around PPO tracking/style/regularization rewards.

### Assessment

The core contribution is a training-distribution intervention rather than a giant architecture: experience recoverable failure and revisit it. Recovery is not the same as safe falling, however. Simulation continuation ignores impact limits, self-collision damage, and clutter; a motion reference may be inappropriate during get-up.

Future work should include impact-aware fall objectives and “safe to recover here” perception. Hardware reports need peak force and damage/intervention counts. Conditioning recovery on free space would make the unified policy practical beyond open floors.


### Soft termination changes the experience distribution

State/reference features are yaw aligned, removing world heading drift while retaining gravity roll/pitch. The actor uses deployable proprioception/reference signals and outputs joint targets at 50 Hz; the asymmetric critic additionally sees contacts, terrain, root state, and domain parameters.

Large tracking error triggers a Bernoulli termination rather than an immediate reset. With a four-second physical recovery horizon at 50 Hz, failed states can remain for roughly 200 useful steps, allowing bracing, rolling, standing, and reference reacquisition. A bounded frame weight rises after poor tracking and falls after success, repeatedly sampling hard transitions without abandoning easy coverage. Pushes, terrain, latency, motor/body parameters, and fallen initial states are randomized.

The contribution is deliberately a training rule rather than a recovery network: policies learn failure only if rollouts include recoverable failure. Yet continuation ignores impact injury and clutter, and a reference can be inappropriate while rising. Next work should condition on free space, optimize safe-fall impact, and report peak forces, interventions, recovery time, and post-recovery tracking—not just eventual success.


### Probabilistic termination and bin-based reward mechanics

Ordinary early termination creates an absorbing blind spot: as soon as error exceeds a threshold, the policy receives no state–action experience showing how to reduce it. Stubborn converts the error signal into a termination probability. Mild failures usually continue; severe or persistent divergence remains likely to reset, preserving training throughput. Surviving failure for a four-second/200-step horizon supplies gradients through the entire sequence from loss of balance to ground contact, reorientation, standing, and reference reacquisition.

Its tracking reward is also discretized into error bins rather than relying only on sharply tuned exponentials. Bins maintain useful discrimination when error is already large, where an exponential reward may be essentially zero, while still distinguishing accurate tracking near the reference. Poorly tracked frames receive increased sampling weights and successful frames decay, coupling curriculum to frame-level failure rather than only whole-clip labels. Yaw-aligned reference/body quantities remove arbitrary global heading while preserving roll, pitch, relative link motion, and contact meaning.

The unified design is operationally simpler than switching among tracking, falling, and get-up policies, but it may learn expedient high-impact trajectories because eventual tracking dominates. Bernoulli termination also adds variance and its probability schedule is a safety-relevant hyperparameter. Evaluation should disclose success conditional on initial fall orientation, surface friction, self-collision, and reference phase, alongside maximum joint torque/contact impulse and time to stable support. Combining Stubborn's experience-distribution idea with SafeFall-style damage objectives would directly address its most consequential blind spot.

### Control-stack accounting and a stronger evaluation protocol

For Stubborn: A Streamlined and Unified Reinforcement Learning Framework for Robust Motion Tracking and Fall Recovery for Humanoids, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

## TANGO: Humanoid Navigation in Cluttered Environments with a Whole-Body Vision-Language-Action Model

[TANGO](https://arxiv.org/abs/2609.09158) maps a natural-language instruction plus egocentric RGB directly to 29-DoF humanoid action targets, treating arms, torso, and gait as part of navigation. Its simulation data pipeline combines global planning, kinematic whole-body motion generation, obstacle-aware editing, and RL tracking to create feasible supervision. This is novel because it learns language-conditioned whole-body traversal rather than issuing planar waypoints to a fixed locomotion policy. It transfers zero-shot to a Unitree G1 in reported tests. The direct policy may be difficult to verify, and RGB-only partial observability or instructions outside the synthetic distribution remain important risks.

### VLA interface and synthetic whole-body supervision

TANGO receives an egocentric RGB observation plus a natural-language route/traversal instruction and directly predicts 29-DoF joint-space action targets for downstream G1 whole-body control. Arms, torso, and gait are therefore action dimensions rather than fixed posture around a planar base. The vision-language-action model must learn that wording and scene geometry imply coordinated behaviors such as narrowing shoulders, moving an arm past clutter, rotating the torso, or changing gait—not just choosing forward/turn waypoints.

Because large human demonstrations of collision-free humanoid navigation do not exist, supervision is synthesized in four stages. A global planner selects a collision-free route; kinematic whole-body generation produces body motion along it; obstacle-aware editing adjusts links to actual 3-D clearance; and an RL tracker converts those references into dynamically executable trajectories. Rendered egocentric images, instructions, and tracked 29-joint targets then train the VLA entirely in simulation. This pipeline is as important as the model: it transforms geometric paths into action labels that respect the morphology and dynamics.

The highlight is collapsing the traditional gap between semantic navigation and whole-body clearance. Modular baselines can choose the right corridor but still strike it with an elbow; TANGO learns visual-language-conditioned posture continuously and reports strong simulation results plus zero-shot transfer to a physical G1. The same directness makes assurance difficult. Monocular RGB hides metric depth and occluded geometry, and synthetic language/motion correlations may shortcut true grounding. Joint prediction also provides no explicit collision certificate. Recurrent multi-view memory, depth/uncertainty fusion, an independent swept-volume safety filter, intervention/clearance metrics, and tests on compositional instructions and moving obstacles are necessary next steps.


### Action semantics, temporal context, and independent safety

Although described as a whole-body VLA, the model predicts desired 29-joint targets rather than raw torque. A downstream controller and servo loop realize those targets, giving stabilization and limiting authority. The model must infer metric clearance and motion phase from egocentric RGB plus language; with only a short context, an obstacle leaving view or a passage behind the camera becomes unobservable. Training data generated by planning, motion editing, and RL tracking implicitly teaches the action distribution of the specific robot and controller.

The highlight is embodiment-level supervision: language about passing sideways or keeping an arm clear maps to coordinated posture, not a planar symbol. But imitation can exploit synthetic correlations among wording, appearance, and standard trajectories. Tests should paraphrase instructions, alter textures and obstacle shapes, reverse layouts, and measure link clearance, collision impulse, joint saturation, and intervention. Depth or multi-view memory would reduce monocular ambiguity. An independent swept-volume checker should filter proposed joint chunks, while uncertainty can slow the robot when instruction or scene lies outside training support.


### Dataset synthesis, VLA architecture questions, and evidence standards

The synthetic pipeline first creates a collision-free base route, then expands it into articulated kinematic motion whose shoulders, elbows, torso, and legs respect scene geometry. Obstacle-aware editing corrects link clearance, and an RL tracker executes or rejects the reference under dynamics. Only successful tracked trajectories should become paired egocentric images, instruction text, and 29-joint targets. This acceptance step prevents the VLA from imitating geometrically attractive but dynamically impossible poses, although it can also censor difficult behaviors and narrow diversity.

Architectural details are essential for interpretation: visual backbone and image resolution, language encoder, number and duration of frames, whether action targets are single-step or chunks, causal memory, fusion mechanism, action normalization, and control rate. A Transformer with temporal tokens can remember occluded obstacles and infer motion phase; a single-image policy cannot. The downstream joint controller's gains and limits determine how much “control” belongs to the VLA versus stabilization. These should be documented with end-to-end latency.

Evaluation should compare language-conditioned whole-body action with waypoint-plus-tracker, RGB-only ablations, oracle depth/scene geometry, and motion generation without RL feasibility filtering. Measure instruction completion, path efficiency, link-wise clearance, collision, fall, action smoothness, and generalization across paraphrases, obstacle geometry, textures, and layouts. Zero-shot real G1 demonstrations are promising but require repeated trials and interventions. The key risk is shortcut learning from synthetic correlations. Counterfactual scenes where the same instruction demands a different posture, plus an external collision shield, would provide much stronger evidence of genuine language-grounded spatial control.

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

## Track Any Motions under Any Disturbances

This paper broadens both sides of the tracker problem: diverse reference motions and diverse external perturbations/dynamics. Large-scale randomization, adversarial disturbance training, hard-motion resampling, and teacher/student learning produce a deployable policy that continues tracking or recovers after pushes. The ambitious “any” claim should be read within training distributions; unseen contact geometry, hardware faults, and infeasible reference accelerations remain failure modes.

### AnyTracker plus AnyAdapter

Stage one deliberately learns expressive motion without broad dynamics randomization. Current state includes angular velocity, projected gravity, joints and previous action; the goal contains next-frame target joint position/velocity. Specialist policies handle clusters with similar action distributions, then a specialist-to-generalist procedure distills their coverage while retaining dynamic/contact-rich AMASS and LAFAN1 clips. The base policy outputs per-joint PD targets.

Stage two freezes/preserves that motor competence and adds a history-informed adapter. A world-model-style encoder processes recent state–action transitions and is distilled from privileged dynamics information; its embedding captures payload, terrain, external force, and actuator mismatch. The adapter predicts bounded, joint-scaled residual corrections to the base action. Separating adaptation avoids the common loss of motion fidelity when one PPO objective simultaneously learns every skill and every randomization.

Hardware tests cover pushes, slopes/terrain and payload/model changes with zero-shot G1 transfer. Still, a history encoder only identifies a disturbance after it affects motion, and multiple causes can be observationally identical. Residual authority must be bounded for safety and may be insufficient for structural failures. Future work should quantify identification time, uncertainty and worst-case residuals, include explicit exteroception for upcoming terrain, and test compound out-of-distribution disturbances rather than interpreting “any” literally.


### Canonical action space and world-model adaptation objective

AnyTracker observes angular velocity, projected gravity, joint positions/velocities, previous action, and the current tracking goal containing target joint position/velocity and related reference state. Before training, actions are canonicalized with bounded nonlinear scaling (including `tanh`) so joints with very different ranges and motion amplitudes contribute comparable optimization difficulty. Motions are clustered by action distribution, specialist trackers solve narrower clusters, and DAgger transfers their actions to the generalist. This addresses motion diversity before exposing the policy to the much larger disturbance distribution.

AnyAdapter uses a history window of 79 state–action pairs. The history encoder does not learn dynamics only from the final tracking reward: a learned world model predicts future/next behavior from its embedding, supplying a dense proxy objective that forces the code to retain dynamic information. A privileged encoder available in simulation further supervises dynamics identification. Small adapter modules are zero-initialized or constrained so the frozen base action is unchanged initially; alternating history-encoder and adapter updates then learn residual corrections under randomized friction, armature, terrain, external velocity impulses, payloads, and sensing effects.

This decomposition provides a useful diagnostic: if nominal tracking deteriorates, the adapter can be disabled and the base policy remains. But one-step/world-model predictability is not identical to control-relevant identification, and a 79-step window introduces a response delay whose physical duration depends on controller rate. The paper would be stronger with causal plots showing embedding convergence after a payload change, ablations over history length, and a bound on residual joint displacement/torque. A mixture or posterior over dynamics could also represent ambiguity instead of forcing every history into one point estimate.

### Control-stack accounting and a stronger evaluation protocol

For Track Any Motions under Any Disturbances, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

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

## UniTracker: Learning Universal Whole-Body Motion Tracker for Humanoid Robots

Source: [arXiv 2507.07356](https://arxiv.org/abs/2507.07356).

UniTracker trains a privileged teacher, then a CVAE student whose partial-observation prior is aligned with a full-state encoder. The latent captures global intent and reduces orientation/root drift that plain MLP students exhibit. A third fast-adaptation stage tunes single or batches of difficult sequences. It reports robust real G1 tracking over thousands of motions. CVAE stochasticity/latent collapse and per-hard-sequence adaptation remain tradeoffs.

### Three-stage universal tracker

Stage one uses PPO and privileged root/body/dynamics state to produce strong joint-position targets. In stage two, a CVAE posterior sees full teacher information while a deployable prior sees proprioceptive history and reference. KL alignment makes the partial prior predict a global motion-intent latent; a decoder combines latent and current state to imitate teacher actions. This explicitly carries information that a flat MLP otherwise loses, particularly heading/root evolution.

Stage three identifies persistent failures and fine-tunes either one sequence or a batch of related hard motions from the universal initialization. This preserves a common base while allowing expensive edge cases to receive targeted capacity. Simulation and real G1 results cover thousands of motions and unseen references.

The CVAE represents ambiguity but may collapse or inject frame-to-frame variation; fixing noise per episode and reporting latent utilization are important. Fast adaptation also weakens “universal” deployment if many clips require separate weights. Future work should use parameter-efficient adapters or routed experts for hard sets, quantify forgetting and adaptation cost, and predict failure/uncertainty before attempting an unsafe reference.


### Latent input/output path and meaning of the adaptation stage

The teacher's privileged observation includes global/root and detailed rigid-body tracking information unavailable from onboard sensing; PPO converts it to joint-position actions. During student learning, the CVAE posterior encoder sees that full information and teacher action, while a deployable prior sees only partial robot observations, their temporal context, and the reference. Both parameterize a latent distribution. A decoder combines a sampled intent code with current deployable state to reconstruct the teacher action, and KL regularization aligns the prior with the privileged posterior. At test time only the prior and decoder remain.

This structure targets a specific distillation failure. A deterministic MLP trained under partial observation averages teacher actions corresponding to different global headings or motion phases, leading to gradual orientation/root drift and loss of expressive diversity. A sequence-level/global latent can preserve which completion is being executed. The paper reports real G1 deployment over more than 8,100 motions, making the scale and teacher-to-student retention more significant than success on a handful of selected demonstrations.

Stage three is best viewed as residual capacity allocation, not evidence that every reference is solved zero-shot. Failed sequences are identified and fine-tuned singly or in batches from the universal initialization, improving extreme cases cheaply compared with retraining the corpus. The operational questions are how adaptations are stored/routed, whether they forget the base repertoire, and how a live system recognizes that an unseen reference needs one. Per-sequence success before and after adaptation, update time and data, base-regression tests, KL/latent-variance statistics, and fixed-versus-resampled latent ablations would answer these questions. A routed low-rank adapter bank plus an uncertainty/failure predictor could retain the benefits while keeping one deployable controller interface.

### Control-stack accounting and a stronger evaluation protocol

For UniTracker: Learning Universal Whole-Body Motion Tracker for Humanoid Robots, the decisive evidence is whether its named mechanism improves the claimed capability while preserving nominal balance, actuator feasibility, and behavior outside the targeted training subset.

For reproducibility, the paper should expose the full causal interface: robot DoFs, proprioceptive variables and normalization, reference representation and future look-ahead, observation-history duration, perception inputs, actor/critic architecture, privileged variables, action parameterization, policy/physics/servo rates, PD gains, termination, rewards, curriculum, motion hours, environment transitions, randomization ranges and hardware limits. These details determine whether performance comes from anticipation, state estimation, reference curation, policy capacity or low-level stabilization. A joint-position target at 50 Hz with 1-kHz PD is a different controller from direct torque even if both are called whole-body policies.

Evaluation should be decomposed by motion or task family and by failure source. Report survival/success, root and body pose/velocity error, contact timing and slip, global drift where relevant, joint-limit/torque/current and impact, action smoothness, fall and intervention, recovery, inference latency and sustained hardware reliability. Oracle-reference, oracle-state or privileged-teacher tests isolate observability; raw versus physics-cleaned data isolates supervision; MLP/history/Transformer or specialist/generalist comparisons isolate architecture. Equal environment steps and compute matter, as do repeated real trials rather than selected demonstrations.

The most useful judgment asks what problem this design solves that a strong conventional tracker does not, and what it sacrifices. Better average tracking may hide lost rare skills, softened timing, higher power or unsafe contacts. The controller should estimate feasibility and uncertainty, reject or simplify references outside support, and retain an independent safety envelope for collision, torque and balance. Future work should test compound distribution shift—new motion plus payload, terrain, latency or disturbance—because isolated robustness factors are easier than the real whole-body deployment problem.

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

## ViBe: Visual Behavior Adaptation for Perceptive Humanoid Whole-Body Control

[ViBe](https://arxiv.org/abs/2609.09918) post-trains an existing motion tracker for visual tasks instead of rebuilding control from scratch. Frozen pretrained visual features pass through a multi-query extractor, and low-rank adapters graft task-relevant feedback into the tracker. Policy optimization then adapts only a compact set of parameters using task reward and reference motion. The method transfers across curb walking, parkour, object interaction, and dodgeball, including difficult lighting and distractors. Its modular efficiency is compelling, though visual backbone bias and reward design still bound what the adapted behavior perceives.

### Post-training visual graft onto a motion tracker

ViBe starts with an already competent reference-motion tracker, whose proprioceptive actor can reproduce broad humanoid behaviors but has no exteroception. A pretrained visual foundation encoder processes RGB; a multi-query extractor learns several task-relevant visual summaries rather than compressing the image with one generic pooled token. These visual features are grafted into selected tracker layers through low-rank adapters. Most tracker and visual-backbone weights can remain fixed, so adaptation changes a small parameter set and retains the motion prior instead of retraining a new geometry policy or distilling a privileged teacher.

Training uses direct policy optimization with the target task reward and a reference-motion dataset. Inputs are RGB, tracker proprioception/state history, and motion/task command; outputs remain the original whole-body joint action/target interface. The task reward teaches visual queries and adapters which features matter for interaction, while motion tracking prevents the controller from forgetting plausible, stable humanoid movement. This is RL fine-tuning of a modular perception-conditioned policy, not behavioral cloning from a separate perceptive teacher.

The breadth of reported zero-shot sim-to-real transfer is the highlight: curb and parkour walking, Repose Cube, omni-object loco-manipulation, and dodgeball, including outdoor, low-light, and RGB-distractor conditions. A deliberately simple high-level planner solving goal-directed Repose Cube suggests the adapted controller contains meaningful closed-loop visual competence rather than requiring a sophisticated planner.

The design makes a strong efficiency claim—reuse large visual and motor priors, train only task-facing connections—but also couples safety to opaque foundation features. RGB lacks explicit metric geometry, several tasks may demand different queries, and low-rank updates can still create rare catastrophic behavior outside the reward distribution. Precise ablations should compare frozen/random/pretrained encoders, adapter rank/query count, and full fine-tuning at equal samples. Future work should add temporal visual memory, uncertainty and out-of-distribution detection, depth/tactile fusion where metric contact matters, independent collision constraints, and continual adapters that compose behaviors without interference.

### Visual token pathway, temporal observability, and task-level evidence

The frozen visual encoder provides generic spatial/semantic features, while multiple learned queries extract different task-relevant summaries—object location, free space, motion, or interaction state—without updating the expensive backbone. Low-rank adapters inject these summaries into selected layers of the motion tracker. The actor still receives proprioceptive history and motion/task command and outputs the established whole-body joint target, so visual learning modifies a stable motor interface rather than replacing it.

Direct RL fine-tuning assigns credit through sparse task outcomes and dense motion/reference terms. Training should state camera placement, image rate/resolution, backbone, query count, adapter rank/location, frame history, action rate, frozen parameters, and simulator visual randomization. A single image can estimate appearance but not object velocity or occluded geometry; temporal tokens or recurrent memory are necessary for dodgeball and moving interaction. Frozen foundation features also need calibration under robot-specific blur, exposure and viewpoint.

The breadth of tasks is promising, but evidence should include controlled ablations: random versus pretrained encoder, one versus multiple queries, adapter versus full fine-tuning, proprioception-only, oracle state/depth, and equal-sample task specialists. Report task success with confidence intervals, collision, tracking degradation, inference latency, outdoor/low-light/distractor performance, and catastrophic visual failures. Low-rank adaptation can still alter actions unpredictably under out-of-distribution images. Uncertainty gating, depth/tactile fusion, an independent collision/contact filter, and composable task adapters would let visual feedback expand behavior while preserving the safety and skill coverage of the base tracker.

### Foundation-model criteria, latent coverage, and deployment accounting

The decisive question is whether low-rank visual grafting adds closed-loop scene response without erasing the pretrained tracker's motor competence or creating rare image-driven failures.

A behavioral foundation model should document the source motion hours and filtering, how kinematic observations become physical robot rollouts, replay or on-policy transition count, network and latent dimensions, temporal horizon/history, privileged training inputs, deployable inputs, action representation, control rate, and total simulation compute. Model size alone is not scale when larger policies receive more environment interaction. Data and compute-normalized curves should vary capacity, behavioral diversity and optimization separately, with several seeds and per-family results.

Latent quality requires closed-loop tests. Reconstruction or action likelihood can be high while decoded contacts drift and the robot falls. Report rollout survival, pose and velocity error, contact timing, diversity, latent utilization or collapse, interpolation behavior, prompt sensitivity, inference latency and hardware reliability. New goals, demonstrations or rewards should be structurally held out, not merely different clips from the same corpus. Compare frozen latent control, weight fine-tuning, task-specific RL, retrieval and random/optimized latent search at equal downstream interactions. A coverage estimator should predict when a requested behavior lies outside the pretrained occupancy.

The practical control stack also needs an authority boundary. A high-level prompt or planner selects latent behavior; the decoder produces joint targets; servo and safety layers constrain execution. Prompt computation, denoising or optimization steps count toward latency, and stochastic samples need temporal consistency. “Zero shot” must state what remains optimized after prompting. My judgment is that foundation status is earned by reusable improvement across different command interfaces and tasks with predictable failure, not by one very broad tracker. Cross-embodiment transfer, uncertainty-based abstention, contact/terrain perception and shielded latent planning are the decisive next tests.

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

## Zero-Shot Whole-Body Humanoid Control via Behavioral Foundation Models

Uses a pretrained behavioral foundation model as a motor prior that can interpret new goals/prompts and produce whole-body behavior without task-specific policy retraining. Latent planning or conditioning selects behavior while a robust controller enforces physics. Zero-shot breadth is valuable, but prompt ambiguity and out-of-distribution contacts need safety checks.

### FB-CPR and META MOTIVO prompting

META MOTIVO is trained in a reward-free MDP with Forward–Backward representations and Conditional Policy Regularization (FB-CPR). Forward features predict latent-conditioned successor occupancy; backward features embed states. Observation-only mocap trajectories are placed in the same latent space, and a latent-conditioned discriminator encourages policies to cover those unlabeled behaviors without needing recorded robot actions. The policy maps proprioceptive state plus latent `z` to continuous joint actions.

A new task is specified without weight updates. Reward samples produce `z` by averaging backward embeddings weighted by reward; a goal state supplies its embedding; demonstration sequences yield an imitation latent. The same policy then performs reward optimization, goal reaching, or motion tracking. This factorization is more than nearest-neighbor retrieval: successor features make the latent an approximate policy/objective coordinate system.

Zero-shot is bounded by occupancy coverage and low-rank approximation. If no pretrained behavior visits the necessary contact state, projecting a reward cannot invent it; observation-only mocap also omits force/action intent. The method's highlight is nevertheless a principled bridge from unsupervised RL to multiple control interfaces. Future work should estimate coverage and prompt uncertainty, combine latent planning with safety constraints, support online memory, and test contacts/objects whose physics cannot be inferred from pose alone.

### Prompt estimation, policy execution, and fair zero-shot claims

FB-CPR trains one latent-conditioned policy and representation, but each prompt modality estimates `z` differently. Reward prompts need state/reward samples representative of the desired objective; demonstration prompts average or infer backward features along a trajectory; goal prompts embed one target state. The latent is then fixed or replanned while the actor maps current proprioception and `z` to continuous joint action. Reporting latent dimension, history, network sizes, replay scale, pretraining interactions, action parameterization and control rate is necessary to compare with ordinary tracking priors.

Zero-shot means no policy-weight update after task specification, not no computation or data. Reward sampling, demonstration encoding, latent search, or environment interaction should be counted. Baselines should include nearest motion retrieval, goal-conditioned RL, imitation prior, random latent search and task-specific training at equal pretraining and prompt budgets. Test reward optimization, goal reaching and imitation on objectives withheld structurally—not merely new clips of familiar behaviors—and report coverage/failure, return, pose/contact quality and hardware robustness.

Successor-feature factorization assumes new rewards are representable over learned occupancy. Contact forces, object dynamics or history-dependent objectives may be invisible in proprioceptive state embeddings, and observation-only mocap supplies no action/force semantics. A coverage estimator should reject low-support prompts and identify which states are missing. Latent planning can sequence several supported behaviors rather than expecting one vector to solve a long task. Combining this with environment perception and a safety shield would make promptable control useful outside an empty floor; otherwise “zero-shot whole-body” remains impressive motor recombination within the pretraining envelope.

### Foundation-model criteria, latent coverage, and deployment accounting

The decisive question is whether a new reward, goal, or demonstration lies inside the successor-feature occupancy learned during pretraining. Promptability should include calibrated failure outside that span.

A behavioral foundation model should document the source motion hours and filtering, how kinematic observations become physical robot rollouts, replay or on-policy transition count, network and latent dimensions, temporal horizon/history, privileged training inputs, deployable inputs, action representation, control rate, and total simulation compute. Model size alone is not scale when larger policies receive more environment interaction. Data and compute-normalized curves should vary capacity, behavioral diversity and optimization separately, with several seeds and per-family results.

Latent quality requires closed-loop tests. Reconstruction or action likelihood can be high while decoded contacts drift and the robot falls. Report rollout survival, pose and velocity error, contact timing, diversity, latent utilization or collapse, interpolation behavior, prompt sensitivity, inference latency and hardware reliability. New goals, demonstrations or rewards should be structurally held out, not merely different clips from the same corpus. Compare frozen latent control, weight fine-tuning, task-specific RL, retrieval and random/optimized latent search at equal downstream interactions. A coverage estimator should predict when a requested behavior lies outside the pretrained occupancy.

The practical control stack also needs an authority boundary. A high-level prompt or planner selects latent behavior; the decoder produces joint targets; servo and safety layers constrain execution. Prompt computation, denoising or optimization steps count toward latency, and stochastic samples need temporal consistency. “Zero shot” must state what remains optimized after prompting. My judgment is that foundation status is earned by reusable improvement across different command interfaces and tasks with predictable failure, not by one very broad tracker. Cross-embodiment transfer, uncertainty-based abstention, contact/terrain perception and shielded latent planning are the decisive next tests.

## 机器人关节伺服系统力矩控制技术综述

This Chinese-language survey reviews torque-control technology for robot joint servo systems, including current/torque loops, disturbance observers, friction and dynamics compensation, series-elastic sensing, and force-control interfaces. It is useful background for the actuator layer beneath whole-body policies. As a survey, it compares approaches rather than contributing a new controller; applicability varies by transmission and sensor design.

### Taxonomy from torque estimation to interaction control

The survey organizes the torque loop as electromagnetic torque/current control, transmission dynamics, output-torque sensing or estimation, compensation, and outer interaction command. Direct-drive and rigid joints can infer torque from motor current but suffer friction, cogging, gear efficiency, and temperature. Strain/load sensing moves measurement toward the output but adds drift, noise, compliance, and cost. Series-elastic joints infer torque from spring deformation, gaining impact safety while introducing resonance and bandwidth limits.

Controller families include PI/PID and inverse-model feedforward, full-state feedback, DOB/reaction-force observation, time-delay, adaptive/robust/sliding-mode compensation, and passivity/impedance loops. High-gain/DOB designs improve transparent tracking on known loads but amplify noise and can inject energy in uncertain contact; passive/compliant designs are safer but less precise; friction identification changes with wear, temperature, direction, and load. Saturation and torque-rate limits must be inside the design.

There is no single network or history window. The highlight is connecting device physics to controller claims: accurate current control does not imply output torque after a gearbox, and low static RMSE does not establish collision safety. Standard bandwidth, apparent-impedance, passivity, impact, thermal, and lifetime benchmarks—plus uncertainty-aware learned residuals validated in multi-joint contact—are natural next steps.

### Practical selection matrix and open benchmarking needs

Controller choice begins with transmission and sensing. Direct drive offers low friction/backlash and high torque observability from current but may be heavy; high-ratio gearing increases torque density while making current a poor proxy for output torque; strain sensing measures nearer the load but drifts; series elasticity adds a calibrated force element and impact tolerance while limiting bandwidth. Sampling rate, current-loop bandwidth, encoder resolution, torque sensor range, compliance, backlash and thermal envelope should be specified before comparing algorithms.

Compensation methods occupy a robustness–performance spectrum. Inverse dynamics and friction feedforward work when identification is accurate; DOB and sliding/robust control tolerate bounded uncertainty but can amplify noise or chatter; adaptive control handles slow parameter change but needs excitation; impedance/admittance and passivity methods shape interaction but depend on outer-loop environment assumptions. Learned residuals can model hysteresis and temperature, yet require out-of-distribution safeguards and should never be judged only by nominal torque RMSE.

A useful public benchmark would use common reference trajectories, locked/free output, several inertias and environment stiffnesses, impacts, reversals through backlash, temperature/wear cycles, saturation and delay. Metrics should include bandwidth and phase, disturbance/noise sensitivity, apparent impedance, injected energy/passivity, peak/RMS current, torque-rate error, computation and failure recovery. Multi-joint tests are essential because whole-body contacts couple local servos through structure and power supply. The survey ultimately reminds policy researchers that a “desired joint target” is not an abstract action: actuator control changes tracking, compliance, safety and sim-to-real behavior.

### Digital implementation, commissioning, and connection to learned policies

Real servo performance depends on the complete discrete implementation. Current control usually runs fastest, followed by torque feedback and then whole-body control; sensor filtering, numerical differentiation, communication buses and zero-order hold add phase that a continuous-time design can miss. Commissioning should proceed from current-loop verification to torque-sensor calibration, friction/backlash characterization, locked-output frequency response, free-motion response, interaction with known compliance, and finally multi-joint contact. Anti-windup, command and torque-rate limits, watchdogs, encoder plausibility, overcurrent/temperature trips and safe torque-off are part of the controller rather than peripheral engineering.

Model compensation also changes over robot life. Gear friction and backlash evolve with wear, strain gauges drift, spring stiffness varies, lubrication and motor resistance depend on temperature, and cables create posture-dependent loads. Online estimators can update slow parameters, but aggressive adaptation during contact may confuse environmental load with actuator change. Parameter confidence and change-rate bounds should determine when adaptation is accepted. Fault isolation should distinguish sensor failure, saturation, unexpected contact and model mismatch because their safe responses differ.

Learned whole-body policies make standardized actuator reporting more urgent. A simulation action expressed as desired joint position is converted through gains and motor limits into torque; a direct-torque policy assumes a different bandwidth and noise model. Papers should publish this action-to-hardware chain and train with measured torque-speed, delay, friction, compliance and saturation. Learned low-level residuals may compensate systematic error, but an analytic/passive baseline should remain available and residual authority should be bounded. Cross-platform transfer should compare not only joint layout and mass but servo dynamics. This actuator-level perspective explains why identical high-level networks can behave very differently on two nominally similar humanoids.


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
