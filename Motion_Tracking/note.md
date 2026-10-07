# Motion Tracking — Paper Notes

Motion tracking work is compared here by the bottleneck each paper attacks: retargeting, long-tail motions, robustness, morphology transfer, reference representation, or scaling. The notes follow those paper-specific arguments and comment on their evidence and remaining failure modes; architectural details are included where they explain why tracking improves. Cross-tagged systems are condensed around their motion-tracking contribution.

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

## CLONE: Closed-Loop Whole-Body Humanoid Teleoperation for Long-Horizon Tasks

CLONE combines a MoE tracking policy, privileged teacher/student distillation, sparse MR head/hand targets, and global pose feedback. The loop corrects root translation drift that accumulates when a tracker only matches local body pose. Its curated CLONED data improves squat/reach and manipulation coverage. This makes tracking useful for long-horizon loco-manipulation rather than short demonstrations. Localization outages and delay can still destabilize correction, and sparse targets underconstrain detailed posture/contact.

### Tracker interface and network

Apple Vision Pro supplies both wrist 6-D poses and head position. LiDAR odometry plus robot forward kinematics estimates those points globally, so the goal includes current-to-target errors rather than only local pose. A privileged PPO/AMP teacher sees all link poses/velocities, next reference state, tracking error, and randomized physics and outputs 29 joint-position targets. DAgger distills it into a deployable student.

The student receives 25 frames of joints, joint velocities, root angular velocity, projected gravity, and previous actions. Its goal block contains positions/velocities of the three tracked points, their errors, and wrist orientations. Three layer-wise MoE blocks each contain four feed-forward experts with widths `(2048, 512, 512, 256)`; top-2 routing blends active experts. The AMP discriminator is a `(256,256,256)` MLP. LiDAR updates at 10 Hz, policy at 50 Hz, and joint PD at 1 kHz, so history bridges stale localization updates.

The ablations support 25 frames and four experts: less history loses motion/error evolution, more enlarges optimization, and additional experts are redundant. The important advance is closed-loop global correction—locally plausible imitation is inadequate if a hand drifts away from the intended table over a long task. Limits are equally concrete: LiDAR delay/outage corrupts error, sparse targets leave elbows/feet/contact unspecified, and MoE routing is not a stability guarantee. Future work should feed localization uncertainty, tactile/force error, and scene collision margins to the controller and evaluate downstream data quality, not only operator-pose error.

### Long-horizon task meaning and safety interpretation

CLONE's “closed loop” closes the geometric loop between operator command and robot's measured global head/hand points; it is not force feedback to the human and not visual object servoing. This distinction matters for tasks such as walking to a table and reaching down: root drift would otherwise move the robot hand away from the intended workspace even if local arm pose remains correct, but a globally accurate wrist still does not prove a stable grasp. The CLONED motion collection deliberately emphasizes squat, reach, and locomotion–upper-body coordination so the experts see configurations absent from generic upright mocap.

MoE routing lets different motion regimes allocate capacity without hard-coded skill switches, while top-2 interpolation reduces discontinuity at expert boundaries. Nevertheless, the gate is another feedback path: a small pose/error change can alter active experts and therefore action. Gate entropy, expert occupancy, and action discontinuity should be reported during localization jumps. Because LiDAR is only 10 Hz, the 25-frame (roughly half-second at 50 Hz) history mixes repeated/stale global estimates with fast proprioception; timestamp/age features would let the network distinguish “unchanged” from “not updated.”

A stronger long-horizon evaluation would report meters/minutes before reset, cumulative root and wrist drift, correction overshoot after deliberate localization bias, and task success under packet loss. Coupling global correction to confidence-weighted MPC or a barrier layer could keep learned motion natural while bounding sudden catch-up commands.

## CLOT: Closed-Loop Global Motion Tracking for Whole-Body Humanoid Teleoperation

CLOT trains a transformer tracker on 20 hours of curated motion and uses high-frequency mocap localization for global pose closure. Observation Pre-shift exposes the policy to future-shifted targets while retaining current-time rewards, teaching smooth catch-up rather than aggressive error cancellation. AMP regularization keeps corrections natural. The method achieves agile, drift-free behavior on a 31-DoF full-size humanoid, but depends on instrumented global tracking and expensive training; its correction behavior is learned, not a formal stability guarantee.

### Closed-loop tracking and Observation Pre-shift

Actor input contains joint state, root orientation/angular velocity, goal pose and previous action, with a ten-frame history to handle partial observability. An asymmetric critic adds privileged simulation state. Each component is embedded as a token and a Transformer encoder attends jointly across history, current feedback and motion goals before producing joint targets.

During training, the observed reference is sometimes shifted to a random future time while reward remains aligned to the true current reference. The actor therefore experiences a positional discrepancy but is rewarded for interpolating smoothly rather than instantly jumping to the shifted target. AMP over the mocap distribution suppresses unnatural high-torque correction. Difficulty curriculum and randomized physical/observation conditions support deployment.

OptiTrack provides global pose at high rate; human motion is captured at 120 Hz, policy runs at 50 Hz, and robot PD at 400 Hz. This closes drift that local pose trackers cannot see. The cost is 1,300+ GPU-hours and an instrumented space. Pre-shift teaches an empirical recovery law, not bounded convergence. Future work should use onboard localization with confidence, compare against explicit MPC/error feedback, and measure correction overshoot, latency sensitivity, force and localization outages.

### Why pre-shifting differs from ordinary target noise

If both actor observation and reward are shifted together, the policy merely learns another valid phase of the motion. CLOT instead changes the command visible to the actor while evaluating behavior against the unshifted physical target. This creates a controlled inconsistency resembling accumulated global error: directly chasing the observed future pose would be penalized, so the optimal policy learns gradual correction that preserves balance and AMP naturalness. The technique is data-driven gain shaping without explicitly defining a root-error controller.

The Transformer tokenization can attend across joint/proprioceptive state, previous action, current global error, and the ten-frame command history. That provides more context than feeding one root-error scalar into an MLP, particularly when the same position error should be corrected differently during single support, a jump, or a hand contact. The asymmetric critic can use clean global quantities even when the actor's feedback is randomized, improving value learning without creating a deployment dependency.

The limitation is identifiability: command pre-shift, sensor delay, and true operator acceleration may look similar. A learned correction can lag or overreact when their distribution changes. Evaluation should sweep shift size/duration and localization latency independently, visualize the learned effective gain by gait phase, and compare energy/contact impulse with a conventional global-error controller. Onboard visual–inertial/LiDAR localization with covariance should replace OptiTrack before claiming operation outside an instrumented volume.

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

## From Generated Human Videos to Physically Plausible Robot Trajectories

This pipeline uses generative human video as a scalable semantic motion source, reconstructs 3D human motion, retargets it to robot morphology, and applies physics-aware optimization/control to remove floating, penetration, joint-limit, and contact violations. It expands motion coverage beyond mocap libraries while producing tracker-ready trajectories. The design is modular—video generation, human recovery, retargeting, physical refinement—but error compounds across stages. Visual plausibility is not dynamic feasibility, so contact reconstruction and validation are essential before hardware use.

### GenMimic pipeline

GenMimic starts from a text prompt and uses a video generator such as Wan2.1 or Cosmos-Predict2 to synthesize a human performing the requested action. A 4-D human reconstruction stage recovers temporally consistent 3-D body motion and camera/world trajectory. The recovered skeleton is mapped into humanoid keypoints and used as the reference for a physics-based policy rather than copied directly into motors. This lets the final rollout repair some visual artifacts through balance and contact dynamics.

The policy is pretrained on AMASS and then expected to track novel generated sequences zero shot. Its reward reweights 3-D keypoints rather than treating every body point equally: reliable/task-defining regions can dominate noisy or occluded ones. Adversarial motion regularization keeps solutions near a broad human-motion distribution when the generated/reconstructed reference jitters. The resulting actor consumes robot state and reference keypoint features and outputs joint targets through the usual PD hierarchy; raw RGB is processed offline, not observed by the deployed tracker.

GenMimicBench contains 428 generated videos spanning multiple subjects, indoor contexts, simple actions, walking-plus-upper-body combinations, and object-interaction-like compositions. Evaluation traces the pipeline at both motion reconstruction and physical execution stages, which is necessary because a video can score well perceptually while inducing an impossible robot reference.

### What the work demonstrates

The key contribution is using video generation as an open-ended *motion specification layer*. It can create combinations and viewpoints absent from mocap, and the physics policy converts a noisy visual proposal into a robot trajectory. This is a plausible path from foundation video models to robot behavior without collecting a robot demonstration for every prompt.

It is not yet a general text-to-robot planner. The video model may violate object permanence, contact, timing, or anatomy; monocular reconstruction can compound those errors; retargeting removes morphology detail; and the final controller sees no scene/object state to verify that an apparent interaction works. Keypoint weighting can ignore unreliable limbs but cannot infer the missing physical cause. Dataset/video-model bias also makes “zero shot” conditional on actions recognizable to every upstream stage.

Future work should generate multiple candidates, score them with contact/dynamics feasibility, and retain uncertainty through reconstruction rather than committing to one pose. Scene geometry and object trajectories should accompany body motion for interaction. A closed loop could render the simulated robot, compare it with intended video semantics, and revise the prompt/trajectory. Reporting failure attribution by generation, reconstruction, retargeting, and tracking would show where additional scale actually helps.

### Teacher–student signals and why keypoints are used

The 4-D reconstruction produces body pose, global translation, and camera-consistent 3-D keypoint trajectories. GenMimic uses the keypoints—not recovered joint angles—as its goal representation because cross-morphology body landmarks remain more stable when the source skeleton has anatomical noise or different joint conventions. A privileged PPO teacher receives clean simulator state plus desired keypoints and emits joint targets. DAgger then supervises a student on onboard proprioception and the same goal form, closing the distribution gap created when a student deviates from teacher rollouts.

End-effector/key-body rewards receive larger weights than less reliable interior landmarks, and symmetry augmentation mirrors robot state, action, and reference while adding a PPO symmetry loss. This both doubles directional examples and reduces systematic left/right bias in synthetic videos. Training begins from 11,451 AMASS motions across 346 subjects before generated-video references are used zero shot. Noise on both robot observation and goal keypoints prepares the student for reconstruction jitter.

The benchmark's 428 clips should be read as a pipeline stress set rather than independent proof of video-model planning ability. Generated video can depict object contact, but the controller has neither object mesh nor force target; it can only reproduce the visible body pattern in empty physics. Results should label such cases “interaction-shaped motion” unless physical interaction is actually simulated. Adding a semantic success evaluator—did the robot point, turn, and fold arms in the right order?—alongside MPJPE would better measure whether physics correction preserves the requested action sequence.

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

## GMR Retargeting Matters General Motion Retargeting for Humanoid Motion Tracking

GMR emphasizes that tracking quality is often bottlenecked before RL: poor human-to-robot references encode impossible joint poses, scale, contacts, or root motion. It formulates general kinematic optimization over selected body correspondences, joint limits, root alignment, and contact/ground constraints, producing robot references from standard human motion formats. The tool is broadly useful for training and online teleoperation. It remains kinematic rather than dynamics-aware; weights/correspondences need embodiment tuning and object interaction is not automatically preserved.

### Five-stage retargeting pipeline

GMR accepts common BVH or SMPL-family human motion plus a robot URDF/XML model. The user declares semantically corresponding key bodies, after which the method (1) matches human and robot bodies, (2) aligns Cartesian coordinate conventions, (3) applies non-uniform local scaling, (4) solves robot inverse kinematics with rotation constraints, and (5) jointly refines root translation/orientation and joints. Joint bounds remain explicit constraints.

Non-uniform local scaling is the main departure from simpler global-height scaling. Human and robot torso, upper/lower leg, and arm proportions differ independently; one scalar can align total height while moving knees, wrists, or feet away from their intended task-space locations. Segment-aware scaling preserves motion semantics before IK, reducing the temptation for the optimizer to compensate with distorted root motion. The staged solve also avoids forcing root translation to absorb pose mismatch too early.

### Controlled downstream evaluation

The study retargets the same diverse LAFAN1 subset to Unitree G1 with GMR, PHC, ProtoMotions, and a proprietary Unitree pipeline. It excludes most non-foot environmental interactions so that body retargeting, not missing scene geometry, is evaluated. Identical independently developed BeyondMimic policies are trained per resulting motion without method-specific reward tuning. Tests include nominal simulation, thousands of domain-randomized trials, and ROS/MuJoCo sim-to-sim with realistic state estimation and network timing. Metrics include survival/success plus global and root-relative body-position error; a blinded user study compares perceived fidelity.

This design makes a strong causal point: retargets that look acceptable kinematically can train policies that are fragile to observation noise, latency, and model mismatch. Better reference geometry improves robustness before adding reward terms or larger networks. The user study is useful because low numerical error can still produce visibly wrong style.

### Judgment and next work

GMR is best understood as a reproducible data compiler. It supports several source formats and robot descriptions, but human–robot correspondences, scale groups, and objective weights remain engineered. The solve is kinematic: it does not know actuator torque, momentum, friction cones, or impact. The evaluation covers one robot and intentionally avoids crawling, sitting, and object contact, so interaction preservation is unproven.

A stronger successor should infer correspondences from body geometry while permitting manual correction; optimize contact timing, torque and support feasibility; and attach a confidence score to every frame. Testing one shared generalist policy, not only per-motion policies, would reveal whether GMR reduces cross-clip conflicts. Cross-robot experiments and failure attribution should distinguish source-conversion, scaling, IK, ground-contact, and controller errors.

### Why retargeting errors amplify under closed-loop noise

Retargeted reference trajectories determine not only pose targets but the error landscape seen by PPO. If a stance foot slides in the target, the controller must choose between matching the moving foot and maintaining frictional support; if the root height is incompatible with leg length, every state contains a persistent balance error. A sufficiently engineered reward can teach the policy to ignore those contradictions, but then the learned behavior no longer has a clear relationship to the reference. GMR's controlled evaluation intentionally suppresses that compensation to expose reference quality.

Local segment scaling reduces these contradictions before IK: torso, arms, thighs, and shanks can be scaled independently in task space, followed by joint/root refinement under explicit limits. Global and root-relative error are both necessary evaluation views. A method can align limbs relative to the pelvis while translating the full body incorrectly, or preserve global feet while distorting local style. Robustness trials with observation noise, mass/friction variation, state-estimation error, and network delay test whether nominal reference error has consumed the controller's stability margin.

The study still does not isolate velocity construction. Differentiating noisy retargeted pose can introduce sharp desired velocities even when positions appear smooth; this can dominate tracking reward and actuator demand. Future retargeters should optimize pose, velocity, acceleration, and contact phase jointly, publish per-joint limit proximity and support-polygon margins, and run forward-dynamics feasibility checks. These diagnostics would let dataset builders reject or repair individual segments instead of learning around them.

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

## HDMI: Learning Interactive Humanoid Whole-Body Control from Human Videos

HDMI reconstructs both human and object trajectories from monocular video and trains an RL policy to co-track them with a unified object representation, residual action space, and general interaction reward. It supports diverse contact modes and reports long repeated hardware interactions such as door traversal. Tracking object state as part of the reference is the key extension beyond free-space motion imitation. Runtime object estimation, reconstruction error, and exact phase dependence remain limitations.

### Reference, policy, and contact objective

GVHMR reconstructs the human, LocoMujoco retargets the body, and object pose/contact are cleaned into a synchronized reference. Every frame contains root/joints, object pose, binary contact, and desired contact points in the object's local frame. The PPO policy observes proprioception, normalized phase, root-relative object pose, and root-relative contact targets. It outputs a residual around the reference joint pose, which makes exploration around kneeling or other far-from-default configurations tractable.

Rewards track local body pose, global root, body velocity, joints, and object pose; regularizers cover action rate, torque, limits, foot impact/slip, and air time. During annotated contact, one generic interaction reward combines end-effector proximity with sufficient but bounded force. Expressing the target in object coordinates lets the same formulation cover a door, box, or ball. Randomizing robot/object inertia and friction teaches some tolerance to physical mismatch.

The 67 consecutive door passages establish repeatability, but phase supplies progress and external state supplies object feedback. Raw RGB/depth is not an actor input. The controller therefore answers how to physically realize a known contact schedule, not when to select or revise it. Event/contact-conditioned phase, onboard pose/tactile estimation, and timing perturbations are the obvious next tests. Success, peak force, object error, and imitation error should be reported separately because visual fidelity and physical task success can disagree.

### Object-frame representation and residual exploration

HDMI's unified object representation expresses desired contact locations and object pose relative to the robot/root and, for contact targets, in the object's local coordinates. This removes arbitrary world placement from the learning problem: opening a door at a new global position should present nearly the same local geometry as the training reference. Binary contact annotations gate the interaction term so free-space approach is not rewarded for generating unnecessary force.

The residual action is added around the retargeted reference joint pose. This makes a human kneel, lean, or reach the starting point of exploration rather than asking PPO to discover it from the neutral configuration. It also creates a risk: if video retargeting places a joint near its limit or encodes an impossible object trajectory, a bounded residual may lack authority to recover. Reference feasibility filtering and adaptive residual scale would separate “repairable mismatch” from “reject this demonstration.”

Co-tracking the object supplies a task-relevant outcome signal unavailable to pose-only imitation. Yet object pose from an external tracker can make hardware tests easier than autonomous perception, and desired force is represented only indirectly. Future versions should estimate object state onboard, use tactile/force feedback to switch interaction phases, and randomize phase/contact timing. For doors in particular, reporting handle force, hinge-angle error, foot slip, and recovery from a missed grasp would reveal more than consecutive successful traversals.

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

## HumanPlus: Humanoid Shadowing and Imitation from Humans

Source: [arXiv 2406.10454](https://arxiv.org/abs/2406.10454).

HumanPlus trains a low-level tracker in simulation on roughly 40 hours of human motion, then drives it from monocular RGB body/hand estimates for real-time shadowing. Shadowing collects robot-aligned whole-body data; behavior cloning on egocentric vision produces autonomous task policies. The 33-DoF robot demonstrates fast and dexterous tasks. It is a complete tracking-to-learning loop, but monocular occlusion/depth error and human–robot contact mismatch limit fidelity.

### Shadowing as a data engine

A Transformer-based low-level controller is trained with RL on AMASS-scale retargeted motion, taking robot proprioception plus target body/hand pose and outputting joint targets. At runtime an external monocular RGB estimator reconstructs the operator; retargeting converts human joints/hands to the custom 33-DoF, 180-cm humanoid. The tracker consumes kinematics, not pixels, and stabilizes morphology/actuation mismatch in feedback.

Shadowing records egocentric RGB, robot state and executed whole-body action in the robot's own distribution. Supervised behavior cloning then trains autonomous visual policies for shoe wearing, warehouse unloading, garment folding, rearrangement and typing with up to 40 demonstrations, reporting 60–100% success. This closes a valuable loop: human video drives a physics tracker, and tracker rollouts become embodiment-aligned imitation data.

The main bottleneck is information quality. Monocular depth/occlusion and hand estimation can issue impossible targets; contacts seen in a human body do not transfer automatically; behavior cloning compounds error. Future systems should fuse depth/multi-view or inertial cues, close object/tactile feedback, attach uncertainty to pose goals, and compare autonomous data value against direct robot teleoperation.

### Separation between shadowing control and autonomous skill learning

HumanPlus contains two distinct policies that should not be conflated. The simulation-trained whole-body tracker maps retargeted human body/hand goals and proprioception to motor/joint targets; it enables real-time shadowing and remains responsible for balance. The downstream autonomous policy is trained by supervised behavior cloning on egocentric RGB plus robot state/action collected during those shadowing rollouts. It learns task-specific visual decisions but delegates low-level feasibility to the same embodiment-aligned action interface.

This data path has an important advantage over simply training on human video: the observations are captured from the robot camera, actions are the commands actually executed by robot actuators, and failures caused by morphology are visible in the demonstration distribution. Up to forty demonstrations per task can therefore be useful despite the modest count. The 33-DoF, 180-cm custom humanoid and roughly 40-hour motion prior also provide hand/body coordination beyond lower-body-only teleoperation.

However, shadowing quality sets a ceiling on behavior-cloning data. A human may compensate visually for a robot error during demonstration, creating correlated corrections that an autonomous policy cannot interpret without the operator. Dataset documentation should record operator view, latency, retargeting confidence, and intervention. Closed-loop imitation methods, recovery demonstrations, and action-chunk or diffusion heads could reduce compounding error, while independent low-level safety limits should remain active when the visual policy encounters a novel object layout.

## HumanX: Toward Agile and Generalizable Humanoid Interaction Skills from Human Videos

HumanX's XGen reconstructs and augments human/object motion; XMimic learns interaction tracking without hand-crafted task rewards. Zero-shot G1 skills cover sports, cargo, and reactive fighting, including repeated exchanges. Its tracking contribution is joint robot–object reference learning and task-agnostic augmentation rather than isolated body imitation. The policy needs reliable object/human state and can only generalize within the physical/contact diversity synthesized from source videos.

### XGen and XMimic data flow

XGen decomposes a source video into pre-contact, contact, and post-contact phases, reconstructs/retargets body and object trajectories, and varies object mesh/size, initialization, and motion. Physics can synthesize non-contact flight; interpolation makes separately augmented phases continuous. The output is therefore a distribution of related interactions rather than one literal monocular reconstruction.

XMimic uses asymmetric actor–critic PPO. A privileged teacher observes proprioception, complete reference/body state, object state, and external-force information and produces a Gaussian joint-action distribution. The student combines PPO with teacher distillation while removing unavailable dynamics/body variables. Policy and mocap run at 100 Hz; joint PD runs at 1 kHz.

In No External Perception mode, recent joint-velocity history helps infer an impact after contact because force changes acceleration. It cannot reveal an unseen object's approach before impact. In MoCap mode, a 14-camera system supplies object feedback. These modes test prior-driven execution versus measured correction, not autonomous visual perception. Future work should fuse vision before contact with proprioception/tactile sensing after contact, infer phase online, and abstain when hidden object state makes aggressive motion ambiguous. Rare impact misses, torque, and timing error are more informative than average pose tracking.

### What is actually generalized, and where the evidence is strongest

HumanX should be read as a **data-and-control co-design** rather than as a policy architecture alone. XGen's phase decomposition makes augmentation physically meaningful: approach and recovery can be varied without blindly warping the instant of contact, while ball flight or other free motion can be recomputed by physics. XMimic then avoids ten separate hand-shaped task rewards and instead learns to reproduce the resulting coupled robot/object trajectories. This separates it from ordinary motion tracking: object state and exchanged momentum are part of the reference, not merely disturbances acting on a body-pose tracker.

The deployment modes reveal the observability boundary. Without external perception, the policy can execute a learned interaction prior and react after impact through joint-state history, but it cannot know the pre-contact trajectory of an unexpectedly moved ball. The 14-camera mode closes that loop by supplying object state. Thus a nominal shot without perception demonstrates a strong motor prior, whereas repeated passing is stronger evidence of feedback interaction. The ten skills across five domains and reported eightfold generalization gain are impressive, but results should be separated by object size, launch velocity, opponent behavior, and whether external tracking was active.

The most valuable next step is autonomous object perception with calibrated uncertainty. A vision estimator should predict object pose, velocity, contact phase, and confidence; the controller should slow, reposition, or refuse an aggressive strike when that posterior is broad. Training should also perturb restitution, aerodynamic drag, latency, and human response rather than only initial geometry. Contact impulse, miss distance, recovery after a mistimed hit, and success over long exchanges would characterize interaction quality better than average pose error.

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

## MOSAIC: Bridging the Sim-to-Real Gap in Generalist Humanoid Motion Tracking and Teleoperation with Rapid Residual Adaptation

Source: [arXiv 2602.08594](https://arxiv.org/abs/2602.08594).

MOSAIC learns a world-frame-consistent general tracker from multiple motion sources with adaptive resampling. A small interface-specific policy learns corrections from limited VR/IMU data, then multi-teacher distillation adds it as a residual to the general tracker. This preserves general competence better than full fine-tuning while handling latency/noise. Residual adaptation is efficient, though major interface/embodiment changes may exceed additive correction capacity.

### General base plus interface residual

The base PPO tracker uses deployable proprioception and motion goals and emphasizes world-frame root/trajectory consistency, because a locally accurate teleoperator can still drift away from the intended workspace. Adaptive resampling revisits difficult clips across the multi-source bank. Domain randomization builds a broad dynamics prior before any particular headset or inertial suit is introduced.

For each interface, a small policy trains on limited streams containing its characteristic delay, noise, coordinate bias and dropout. Multi-teacher DAgger then supervises an additive residual module on top of the frozen/general action. Only that module specializes; the base retains offline replay and unrelated motions. Hardware long-horizon teleoperation and OOD tests show less forgetting than full fine-tuning or ordinary continual learning.

Additive adaptation is transparent and cheap, but cannot repair a command representation outside the base manifold or large phase/global-estimation errors. Several residual teachers may conflict when interfaces are combined. Future work should gate residual authority by uncertainty, compose adapters with conflict tests, bound corrections for safety, and evaluate adaptation data efficiency versus calibration and estimator improvements.

### Exact policy interface, data scale, and residual training

Both offline replay and live teleoperation use only the **next** robot-space reference frame, which avoids depending on future human intent. The actor stacks five steps of reference joint position/velocity, anchor orientation, base angular velocity, measured joint position/velocity, and previous action. The asymmetric critic additionally receives anchor/body positions and orientations plus base and reference linear velocity. A `[1024, 1024, 512, 256]` ELU MLP outputs 29 desired joint positions at 50 Hz; PD control and torque, velocity, and joint-limit saturation form the final safety layer. This one-step causal interface is a major practical contrast with trackers that perform well partly because they see a long future window.

Training combines about 64 hours from optical and inertial MoCap, AMASS/OMOMO, generated motion, and roughly one hour of adaptation data. Sampling operates at two levels: failure-aware bins revisit hard portions within a clip, while motion-level assignment balances difficulty, novelty/under-use, and a uniform floor. World-frame anchor, body, foot, velocity, and VR terms are added because root-relative accuracy can look excellent while a teleoperator drifts across the room. The OOD benchmark also emphasizes sequences longer than ten seconds, making accumulated instability visible.

For a new interface, a specialized policy learns its gait bias, delay, and noise from roughly a short calibration session (the paper discusses about 30 minutes). The frozen general policy is then combined with a near-zero-initialized residual. Dual-teacher behavior cloning uses the specialist on adaptation samples and the general tracker on broad data, so the residual adds corrections without overwriting the base repertoire. This is the paper's clearest novelty and explains why it beats naive fine-tuning.

Residual success should nevertheless be interpreted locally: it works when the new stream remains inside the base policy's motion/action manifold. A large coordinate error, missing contact phase, or new embodiment cannot necessarily be repaired additively. Stronger diagnostics would plot correction magnitude and failure probability over time, impose joint-wise residual limits, test abrupt interface switching, and compare correction learning with fixing the upstream calibration itself. RobotBridge and the open training/deployment stack are also meaningful contributions because they make these distinctions reproducible across simulator and hardware backends.

## OmniH2O: Universal and Dexterous Human-to-Humanoid Whole-Body Teleoperation and Learning

OmniH2O uses kinematic pose as a common goal representation across VR, RGB shadowing, language/frontier models, and stored data. A privileged RL teacher is distilled to a sparse-sensor real policy over large retargeted/augmented human motion. The same tracker supports sports, manipulation, teleoperation, and data collection. Versatility comes from the interface; execution remains bounded by reference quality, morphology, hand hardware, and training coverage.

### Sparse motor API

The deployable goal is only head position plus two hand positions/velocities. The student also receives 25 steps of joint positions/velocities, root angular velocity, projected gravity, and previous actions; it does not receive global linear velocity. A PPO teacher sees full rigid-body state and reference error over 14,000 augmented AMASS sequences. DAgger transfers teacher actions to the history-based student, which outputs PD joint-position targets. Fingers remain a separate VR-to-IK path rather than learned contact control.

History makes unobserved velocity/contact partially inferable, while the learned prior fills feet, pelvis, and elbows left unspecified by three points. Dataset balance determines that null-space behavior: standing and squatting augmentation prevents a manipulation policy from satisfying hand targets by constantly shuffling. A scheduled foot-height reward distinguishes intended steps from stationary tasks. Reported simulation success of 94.1% is close to the teacher's 94.77%, evidence that deployable history retains most task-relevant information.

The framework should be viewed as a robust command interface, not scene understanding. Any VR, RGB, or language module must still produce reachable safe points; the tracker has no knowledge of an unseen table and sparse commands can conflict. Variable-cardinality goals, reachability/uncertainty prediction, collision perception, tactile fingers, and cross-robot testing are natural extensions. Foot motion during stationary hand tasks and responses to adversarially fast commands should accompany aggregate success.

### Information bottleneck and downstream autonomy

OmniH2O's unusual design choice is to reduce the live whole-body command to three Cartesian points—head and two hands—while giving the student a 25-step proprioceptive history. This makes the policy usable with commodity VR and monocular pose systems, but it also makes lower-body behavior fundamentally prior-driven. The teacher can exploit full rigid-body/reference state during PPO; DAgger then labels states actually visited by the partially observed student, preventing a purely offline distillation set from missing its own balance errors. The history encoder acts as a crude state estimator for base translation, contact, and target velocity that are absent from a single observation.

Kinematic pose is also the boundary between low-level control and autonomy. GPT-4o, verbal instructions, an RGB shadowing module, stored trajectories, or an imitation policy need only propose the same goal representation; none directly commands torque. OmniH2O-6 records paired first-person RGBD, control goals, and whole-body motor actions for six everyday tasks, so autonomous policies can be learned above the stable tracker. Dexterous fingers, however, use a separate mapping/IK path, and neither the sparse tracker nor the high-level model inherently reasons about grasp force.

This modularity is a highlight because failures can be localized to perception/planning versus balance. It can also conceal ambiguity: identical hand/head paths may require stepping, leaning, kneeling, or remaining planted depending on objects and intention. Future versions should condition the completion on scene geometry, explicit contact mode, and uncertainty, and should evaluate multiple valid completions rather than one mocap match. Latency sweeps, history-length ablations, operator-command bandwidth, foot-slip/contact metrics, and intervention rates would better quantify teleoperation quality than whole-body success alone.

## OmniRetarget: Interaction-Preserving Data Generation for Humanoid Whole-Body Loco-Manipulation and Scene Interaction

An interaction mesh preserves agent–object–terrain spatial/contact relationships while Laplacian deformation and kinematic constraints adapt human motion across robot embodiments and scenes. The generated trajectories train simple shared-reward trackers for long interactions. This improves motion tracking because contacts are designed into the reference rather than left for RL to repair. Optimization is still kinematic and requires accurate interaction geometry/contact information.

### Tracking implications of interaction-preserving data

Mesh vertices join body, object, and support keypoints; Laplacian coordinates preserve local relations such as hand-to-handle and foot-to-obstacle as robot proportions or scene geometry change. Joint limits, collisions, and contact constraints then refine the deformation. Object pose/scale and terrain can be augmented while attached relations move the relevant body parts consistently, yielding more than eight hours of pickup, carrying, pushing, parkour, and scene references.

The downstream PPO actor is intentionally simple and proprioceptive: pelvis linear/angular velocity, joints, projected gravity, and previous action enter; joint-position targets come out. A privileged critic may see reference/environment state, but deployed policies use no camera or object pose. Only a small common reward/randomization set is used. This is a diagnostic result: if a simple controller works, the interaction reference—not task-specific reward engineering—contains much of the solution.

Geometry is still not dynamics. A mesh-consistent lift may violate torque or friction, and a proprioceptive skill cannot correct an object that departs from the scripted path. Future versions should add wrench, support, momentum, and learned dynamic-feasibility costs; attach confidence/difficulty to generated clips; and close execution with object perception. Evaluation should report contact error, penetration, required torque, and success versus augmentation distance, not just resemblance to the human seed.

### Why the simple downstream controller is an important result

The interaction mesh changes the unit of retargeting from isolated body joints to a graph spanning robot/human surface samples, manipulated objects, and terrain. Preserving Laplacian coordinates keeps local shape relationships—hand around a handle, pelvis above a seat, foot on an edge—while optimization enforces joint limits, smoothness, nonpenetration, and designated contacts. Because the graph can be deformed with a new object pose, terrain shape, or embodiment, one captured interaction produces a family of references whose participants still meet at the right places and times.

The resulting policies share only five reward terms, four robot-domain randomizations, and proprioceptive observations across tasks, without a task curriculum. That minimalism is strong evidence for the paper's thesis: 30-second chair-moving/stepping/vaulting/rolling sequences become learnable because the reference already encodes the interaction, rather than because a reward engineer separately describes each event. More than eight hours generated from OMOMO, LAFAN1, and in-house capture also support pickup, carrying, slopes, platforms, and multiple robot morphologies.

Yet the real demonstrations use a known, arranged scene. A purely proprioceptive actor follows the memorized object/environment trajectory and cannot detect that a chair is displaced or a foothold is missing. Laplacian preservation is geometric rather than force-aware, so it may keep a hand attached while demanding impossible torque or friction. A natural extension is a two-stage acceptance test: retarget geometrically, then simulate/contact-optimize and label required wrench, stability margin, and uncertainty. Online depth/object pose and tactile correction could then turn these impressive scripted interaction tracks into robust skills under scene variation.

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

## Stubborn: A Streamlined and Unified Reinforcement Learning Framework for Robust Motion Tracking and Fall Recovery for Humanoids

Stubborn trains tracking and recovery in one RL framework rather than switching to a get-up policy. Fallen/perturbed initialization, unified rewards, and a streamlined observation/action design teach the robot to resume its reference after disturbance. This reduces mode-transition engineering and supports locomotion continuity. Recovery coverage depends on sampled fall states and surface assumptions; aggressive self-righting may be unsafe around people or objects.

### Soft termination changes the experience distribution

State/reference features are yaw aligned, removing world heading drift while retaining gravity roll/pitch. The actor uses deployable proprioception/reference signals and outputs joint targets at 50 Hz; the asymmetric critic additionally sees contacts, terrain, root state, and domain parameters.

Large tracking error triggers a Bernoulli termination rather than an immediate reset. With a four-second physical recovery horizon at 50 Hz, failed states can remain for roughly 200 useful steps, allowing bracing, rolling, standing, and reference reacquisition. A bounded frame weight rises after poor tracking and falls after success, repeatedly sampling hard transitions without abandoning easy coverage. Pushes, terrain, latency, motor/body parameters, and fallen initial states are randomized.

The contribution is deliberately a training rule rather than a recovery network: policies learn failure only if rollouts include recoverable failure. Yet continuation ignores impact injury and clutter, and a reference can be inappropriate while rising. Next work should condition on free space, optimize safe-fall impact, and report peak forces, interventions, recovery time, and post-recovery tracking—not just eventual success.

### Probabilistic termination and bin-based reward mechanics

Ordinary early termination creates an absorbing blind spot: as soon as error exceeds a threshold, the policy receives no state–action experience showing how to reduce it. Stubborn converts the error signal into a termination probability. Mild failures usually continue; severe or persistent divergence remains likely to reset, preserving training throughput. Surviving failure for a four-second/200-step horizon supplies gradients through the entire sequence from loss of balance to ground contact, reorientation, standing, and reference reacquisition.

Its tracking reward is also discretized into error bins rather than relying only on sharply tuned exponentials. Bins maintain useful discrimination when error is already large, where an exponential reward may be essentially zero, while still distinguishing accurate tracking near the reference. Poorly tracked frames receive increased sampling weights and successful frames decay, coupling curriculum to frame-level failure rather than only whole-clip labels. Yaw-aligned reference/body quantities remove arbitrary global heading while preserving roll, pitch, relative link motion, and contact meaning.

The unified design is operationally simpler than switching among tracking, falling, and get-up policies, but it may learn expedient high-impact trajectories because eventual tracking dominates. Bernoulli termination also adds variance and its probability schedule is a safety-relevant hyperparameter. Evaluation should disclose success conditional on initial fall orientation, surface friction, self-collision, and reference phase, alongside maximum joint torque/contact impulse and time to stable support. Combining Stubborn's experience-distribution idea with SafeFall-style damage objectives would directly address its most consequential blind spot.

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

## TWIST2: Scalable, Portable, and Holistic Humanoid Data Collection System

TWIST2 uses PICO 4U full-body tracking and a low-cost active neck to command a full-body policy without mocap. The tracker must tolerate noisier portable signals while preserving locomotion/manipulation coordination. It collects 100 demonstrations in roughly 15–20 minutes and feeds hierarchical visual policy learning. Portability improves scale, but VR calibration, occlusion, and drift reduce reference quality relative to optical capture.

### Portable reference stream and low-level tracker

PICO supplies head, wrists and body estimates at up to 100 Hz; a retargeter converts them to robot motion. The learned low-level controller runs at 50 Hz and sends desired joints to PD. It is trained on about 20,000 clips—roughly 7,000 GMR-retargeted motions plus TWIST data—with tracking and small action regularizers. A convolutional history encoder compresses recent proprioception and reference motion before an MLP, helping smooth portable-sensor noise and delay. The active two-DoF neck moves the egocentric camera independently enough to inspect hands/workspace while maintaining whole-body control.

The same system collects RGB/proprioception/action demonstrations for a high-level visuomotor Diffusion Policy. That policy uses image plus normalized proprioception, predicts 64-command chunks with temporal convolution, runs around 20 Hz, and supplies motion references at 30 Hz; the tracker closes the faster physical loop. Hierarchy prevents visual inference jitter from directly becoming torque.

TWIST2's importance is throughput and one-operator portability, not better optical accuracy. VR drift, body self-occlusion, network latency, hand mapping, and accumulated global error can contaminate demonstrations. Future work should log calibration confidence, close global/object pose, integrate tactile hands, and measure downstream policy success per hour of collection rather than only teleoperation completion.

### Hierarchy, action bandwidth, and data-quality judgment

The autonomous stack deliberately predicts a motion-level command rather than raw motor torques. Egocentric stereo images and normalized proprioception enter a Diffusion Policy that emits chunks of base velocity/orientation, full-body pose, neck, and hand commands; the general tracker converts those commands to stable motor targets at the faster control rate. Chunking (the reported policy predicts 64-command sequences) reduces visual-action jitter and supplies short-horizon intent, while the low-level history encoder absorbs sensor noise and closes balance. This division is especially suitable for mobile manipulation because visual reasoning need not run at the servo frequency.

The active two-DoF neck is more than a camera mount. It decouples gaze from torso pose, allowing the operator and autonomous policy to inspect hands or the next foothold without forcing whole-body retargeting to turn the robot. The released state/action representation includes planar base velocity, base height, roll/pitch and yaw rate, body joints, hands, and neck; synchronized stereo images and last action make the dataset useful for closed-loop imitation. Reported collection rates—about 100 successful pick-and-place demonstrations in 15–20 minutes and roughly 50 mobile demonstrations in 20 minutes—show the throughput advantage over studio MoCap.

High success during collection does not automatically mean high information diversity. Action chunks can blur corrective timing, one environment can produce correlated backgrounds, and VR pose estimation may introduce systematic morphology bias that the tracker quietly repairs. The right comparison is autonomous success per operator-hour against MoCap, bilateral teleoperation, and scripted collection, with calibration/setup time included. Dataset audits should publish latency distributions, tracker residual error, camera coverage, intervention/collision labels, operator diversity, and train/test scene separation. Those measurements would establish whether TWIST2 scales not only the count of demonstrations but also the effective diversity needed for general visuomotor learning.

## TWIST: Teleoperated Whole-Body Imitation System

TWIST retargets optical human mocap into a unified whole-body RL tracker and records synchronized egocentric vision, states, and actions for autonomous imitation. It established that locomotion and manipulation demonstrations should be collected through one coordinated controller rather than decoupled base/arms. High-quality mocap yields clean tracking, but the studio setup is costly and less scalable than later VR systems.

### Teacher, student, and live interface

Public AMASS/OMOMO and 150 in-house mocap clips are retargeted, with IK cleanup for foot placement and body orientation. A privileged PPO teacher sees two seconds of future reference, allowing anticipation of fast turns, squats, and contacts. The deployable student sees proprioception and only the immediate target; it is optimized jointly with PPO tracking reward and behavior-cloning/distillation from the teacher. This avoids the hesitation produced when a live controller is trained to expect future frames that streaming mocap cannot provide.

Mocap streams at 120 Hz, teleoperation policy at 50 Hz, and joint PD at 1 kHz. One network outputs whole-body joint targets, coordinating stepping, torso and arms while synchronized egocentric RGB/state/action logs become imitation data. Domain randomization and action/joint penalties support transfer.

TWIST's scientific contribution is aligning training information with live availability while retaining an anticipatory teacher. The student cannot reconstruct genuinely unknowable operator intent, so distillation learns the average best response and may smooth sudden changes. Optical infrastructure gives excellent pose but constrains workspace and scale. Portable sensing, explicit intent prediction, global drift correction, and uncertainty-aware slowing are natural successors.

### RL+BC architecture and why real-time data matters

TWIST represents each target by retargeted humanoid joint positions and root velocity rather than raw human skeletal coordinates. A privileged teacher trained with PPO receives robot state plus roughly two seconds of future reference, which lets it prepare support changes before a squat, kick, or lateral step. The causal student receives the present streaming target and proprioception and is optimized with both the physical tracking reward and behavior cloning from the teacher. BC transfers anticipatory structure where it is predictable from the current pose, while continued RL prevents the student from merely copying actions that are unsafe in states it reaches under its restricted observation.

Adding live in-house MoCap is important even when public AMASS/OMOMO motion exists. Real teleoperation contains pauses, reversals, calibration error, abrupt operator decisions, and transitions between reaching and walking that curated clips underrepresent. Training on those signals narrows the interface distribution gap. A single whole-body action vector then coordinates feet, torso, and arms at 50 Hz, with faster joint PD underneath, instead of switching between locomotion and manipulation controllers.

The main achievement is coordinated embodiment: carrying a box while walking, crouching to reach the floor, kicking, sideways locomotion, and expressive dance all use one learned balance mechanism. It is not yet autonomous adaptation—the operator sees the scene and supplies intent, and the controller has no explicit object geometry or force objective. Future work should add time-stamped latency compensation, predict a distribution over short future human motion, fuse force/tactile feedback, and train recovery demonstrations. Evaluation should separate operator error, retargeting error, and low-level tracking error and report contact/torque peaks, task completion, and correction latency in addition to pose fidelity.

## Unified Motion Retargeting for Humanoids with Learned Point Cloud Correspondence

Source: [arXiv 2609.02134](https://arxiv.org/abs/2609.02134).

UMR treats exterior surface point clouds—not skeleton joints—as the common interface between humans and robots. Learned dense correspondence in canonical poses supplies fine geometric anchors; constrained matching then retargets pose and interaction contacts across motion sources and embodiments. This removes hand-designed sparse body mappings and improves surface/contact fidelity. A new source template or robot still needs correspondence learning, and point geometry alone does not ensure dynamic feasibility or force consistency.

### Dense correspondence as the retargeting interface

Human sources and robot meshes are sampled into exterior point clouds. A correspondence network learns canonical dense features so a source surface point can select an analogous target location without matching skeleton names or joint topology. For each motion frame, constrained optimization moves robot root/joints so corresponding surface points align while respecting kinematics, smoothness, limits and contact anchors. The same representation transfers a hand/object or body/support contact directly rather than approximating it with a few manually chosen key bodies.

Experiments span heterogeneous source formats, robot embodiments, locomotion and interaction, measuring fidelity/plausibility and downstream reference usefulness. Dense anchors capture torso/limb shape and contact area that sparse joints miss, making UMR particularly attractive for cross-morphology data compilation.

Surface similarity is not semantic identity or physics: nearest-looking patches may have different load capability, friction or actuation, and correspondence errors can create plausible but wrong contacts. Optimization remains per-sequence and a new mesh/domain needs learned canonical features. Future work should include contact wrench/torque feasibility, confidence and human-correctable correspondences, temporal consistency, and closed-loop simulation acceptance before references reach hardware.

### Learned correspondence, reuse, and failure analysis

UMR first aligns source and robot in canonical poses and learns an ordered dense surface correspondence. This is a reusable preprocessing step for a given source template–robot pair, not a network that must infer mappings independently on every motion frame. During retargeting, corresponding positions and orientations become optimization objectives alongside robot joint limits, temporal smoothness, contacts, and kinematic constraints. Because the interface is an exterior point cloud, the same machinery can consume different skeletal conventions and can transfer a palm, forearm, knee, or torso contact without authors choosing a short list of semantic joint pairs.

This is especially valuable for morphology gaps. Sparse end-effectors can match both hands and feet while producing an implausible spine, elbow orientation, or surface collision; dense anchors constrain the complete silhouette and distribute error over the body. Directly reusing contact-associated surface points also preserves interaction geometry better than asking a downstream PPO reward to rediscover it. Experiments across heterogeneous sources, robots, locomotion, and interaction test this claimed unification rather than only one human/G1 mapping.

Dense correspondence also expands the ways a system can be wrong. Symmetric limbs or geometrically similar patches can be swapped; clothing/body-shape surfaces do not reveal which robot link safely bears load; and a visually close surface match can demand excessive velocity or torque. Correspondence confidence, cycle consistency, temporal tracking of each match, and a small human-editable anchor set would make failures auditable. A strong production pipeline would cache the learned map, optimize a sequence, validate it in dynamics, and either repair or reject references using contact impulse, torque margin, penetration, and closed-loop survival. That final dynamics gate is needed before “robot-ready” can mean more than geometrically plausible.

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

## X-OP: Cross-Morphology Whole-Body Teleoperation via MPC Retargeting

X-OP performs receding-horizon optimization from captured human pose to robot whole-body targets, jointly considering morphology, smoothness, contact, balance, and feasibility. MPC anticipation avoids independent-frame IK artifacts and feeds a stabilizing low-level controller. It is interpretable and constraint-aware, but compute, model/contact mismatch, and tuning constrain agility; learned trackers are often faster but less explicit.

### MPC around the closed-loop robot system

Apple Vision Pro provides head and wrist targets. Rather than map these directly into joints, sampling-based MPC optimizes a horizon of the *same high-level commands accepted by an existing low-level policy*. Its rollout model includes simulator plus policy; costs measure operator target alignment, smoothness, energy and stance/contact constraints. Sequential quadratic programming synchronizes simulated root/joint state with SLAM, IMU and encoder measurements at each MPC step, preventing model rollouts from drifting away from hardware.

On a humanoid using FALCON, the high-level action is a compact seven-dimensional locomotion/body command and the low-level policy contains state–action history; MPC runs at 25 Hz while standing and 10 Hz while walking, with policy at 50 Hz. The same formulation controls a wheeled manipulator, demonstrating that only action interface and model/costs change rather than retraining a cross-morphology neural retargeter.

Explicit constraints and prediction improve obstacle clearance and completion time, but the optimization is only as correct as contact/model/state estimates. Low MPC rate limits rapid human motion, and operator intent is still sparse. Future work should incorporate uncertainty/chance constraints, learned residual dynamics, collision perception and warm-start failure handling, while reporting worst-case solve time and safe fallback behavior.

### Optimization interface and cross-morphology significance

X-OP does not optimize every motor torque. The XR headset supplies head and wrist goals, and MPC searches over the compact command space already understood by a robot's low-level controller. For the humanoid/FALCON example this is a seven-dimensional locomotion/body command; the learned 50 Hz policy, including its own state–action history, realizes balance and contact. The predictive outer loop runs around 25 Hz in standing and 10 Hz in walking. A wheeled manipulator exposes a different action interface, yet the same goal/cost/rollout formulation applies without training a new human-to-robot neural policy.

At every MPC update, measured encoders and IMU plus SLAM global pose synchronize the simulated rollout state to hardware. This reset is crucial: contact simulation is sensitive, and allowing the optimizer's internal trajectory to drift would make an apparently optimal horizon irrelevant to the real robot. Costs combine head/hand intent, locomotion or posture preference, smoothness, power, and collision/contact constraints. Receding-horizon prediction can trade a temporarily imperfect hand match for a feasible step, something per-frame IK cannot do.

The headline cross-morphology result comes from separating **intent optimization** from **motor execution**, not from one universal low-level policy. Reported simulation gains include over 30% lower task completion time and 20% lower power for the humanoid and zero collisions for the mobile platform relative to baselines. The price is compute and model dependence: 10 Hz walking updates may miss rapid intent, a bad contact model can confidently choose the wrong step, and optimizer failure requires a defined fallback. Future work should publish solve-time tails and infeasibility rates, use uncertainty-aware or learned residual dynamics, warm-start from operator motion history, and enforce a certified safe command set. Comparisons should hold the same low-level controller constant so gains are attributable to MPC retargeting.


## BFM-Zero: A Promptable Behavioral Foundation Model for Humanoid Control Using Unsupervised Reinforcement Learning

BFM-Zero replaces a library of task-specific whole-body policies with one latent-conditioned controller whose command can be derived from a motion, goal state, or reward. The key achievement is not a new motion tracker alone: the same frozen policy performs zero-shot motion tracking, single-state goal reaching, and reward optimization, while low-dimensional latent search provides few-shot adaptation when zero-shot behavior is insufficient. Hardware demonstrations use a Unitree G1, making this an explicit sim-to-real behavioral-foundation-model result rather than only character animation.

### Unsupervised RL and forward–backward representation

The method uses off-policy unsupervised reinforcement learning based on a forward–backward (FB) factorization of discounted successor occupancy. A forward representation depends on state, action, and behavior latent, while a backward encoder represents reached states. Their inner product approximates how a latent-conditioned policy occupies future states. A downstream reward can therefore be projected into the learned basis to obtain a command (z) without retraining policy weights; a desired goal uses its backward embedding, and a reference sequence produces a time-varying tracking prompt.

This is genuine RL, but not PPO trained separately on each downstream objective. Reward-free/off-policy pretraining learns the shared representation and policy from replay. Motion-capture states regularize exploration toward useful humanoid behavior; auxiliary safety and motion terms prevent reward-free discovery from collapsing into falls or joint-limit exploitation. Asymmetric learning lets the training critic use privileged simulation state, while the deployed actor uses a history of robot observations. Domain randomization, reward shaping, and history dependence address latency, disturbances, and unobserved physical parameters. The output is joint-level target action consumed by the robot's low-level servo/PD stack; there is no camera, LiDAR, or language model in the core policy.

The prompt interfaces contain different amounts of information and should not be conflated. A motion prompt asks the learned occupancy to resemble demonstrated states over time. A goal prompt emphasizes one desired state region. Reward inference estimates a latent from sampled states weighted by a user-specified reward. Few-shot adaptation searches the compact latent rather than updating millions of network weights. This makes the representation objective-centric and more interpretable than an arbitrary skill code, but performance is limited to behaviors inside the pretraining occupancy span.

### Contribution, limitations, and next tests

The important difference from conventional universal motion tracking is that reference following is only one inference mode. The learned successor structure exposes a reusable control interface for goals and rewards, and natural recovery can emerge because the policy optimizes occupancy rather than replaying a brittle trajectory. Its real-world value is a common motor layer that a planner can prompt without training a fresh controller for every task.

“Zero shot” still needs careful accounting: prompt construction may require reward samples, a reference window, or optimization, and few-shot latent search consumes interaction. Mocap regularization also means the behavioral prior is not fully unsupervised. A requested reward outside the learned state distribution cannot be created through linear projection, and unsafe reward maximization may exploit simulator or representation error. Results should therefore report observation-history length, control rate, network and latent dimensions, replay transitions, prompt-computation time, privileged critic inputs, action-to-torque chain, and coverage diagnostics.

The next step is a calibrated coverage estimator that abstains when a prompt lies outside learned occupancy, combined with safety-shielded latent search. Structurally held-out motions, rewards, and goals should test compositional generalization rather than clip-level interpolation. Cross-embodiment transfer, vision/contact conditioning, online prompt updates after environment changes, and comparisons against task-specific RL at equal environment interactions would show whether the “foundation” representation scales beyond a broad proprioceptive controller.


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
