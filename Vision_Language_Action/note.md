# Vision-Language-Action — Paper Notes

This folder contains policies that use both visual observations and language or semantic task instructions to generate robot actions. It includes foundation VLAs, humanoid-specific VLA adaptations, and world-action models with an explicit language-conditioned action interface. Vision-only controllers, execution middleware, and data systems without a deployed language-to-action policy are excluded. Each note emphasizes observation and language inputs, token or latent fusion, action representation, control hierarchy, training data, inference rate, embodiment interface, and safety limitations.

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

## π0.7: a Steerable Generalist Robotic Foundation Model with Emergent Capabilities

Source: [arXiv 2604.15483](https://arxiv.org/abs/2604.15483).

[π0.7](https://arxiv.org/abs/2604.15483) extends the π-family from task-conditioned imitation toward a context-steerable generalist. Context may include language strategy/manner, visual subgoals, embodiment/task metadata, episode quality, and autonomous success/failure data. Conditioning on metadata allows one model to distill RL-trained specialists and also learn recovery states from suboptimal rollouts.

The architecture accepts up to four 448×448 camera views—front, two wrists, optional rear—with as many as six historical frames per view sampled one second apart, plus up to three visual subgoal images. A MEM history encoder compresses each multi-frame stream to the token budget of one image. History and rear view are independently dropped with probability 0.3 for robustness. Proprioceptive current/history states are linearly projected as individual tokens rather than serialized as text.

Block-causal attention gives bidirectional processing within observation and subgoal blocks, then causal processing for text/context. The continuous flow/action head remains related to earlier π models, while the richer context can request not merely “make espresso” but a particular stage, strategy, or quality regime.

Reported emergent behavior includes unseen scenes, multi-stage appliances, cross-embodiment laundry folding, and performance approaching specialized policies. For humanoid work it is a promising semantic planner/data prior, but not evidence of whole-body balance. Its cameras and state histories assume embodiment-specific calibration; visual subgoals can be unreachable, and context labels may correlate with dataset artifacts rather than causal strategy. A humanoid still needs a feasible command abstraction and stabilizing controller.

### Why steerable context is more than a larger prompt

The notable shift from π0 to π0.7 is that context describes not only the task but also **how the task should be attempted**. Strategy, subgoal images, embodiment identity, and rollout quality can distinguish trajectories that share the same short language instruction. This provides a common mechanism for distilling specialist policies, incorporating autonomous trial data, and selecting among multiple valid approaches without training a separate model for each regime.

The history design is also well matched to partial observability. Six frames sampled across several seconds can reveal whether a drawer is moving, whether an earlier grasp failed, or which stage has already completed. MEM compression prevents that benefit from multiplying the transformer token count by six. Multiple camera views reduce occlusion, while random view/history dropout prevents the model from treating every stream as mandatory.

There is, however, a causal ambiguity. A “successful” metadata tag may correlate with a particular lab, camera, operator, or object appearance. The model can appear steerable while exploiting these incidental cues. Visual subgoals similarly express desired appearance but not whether that appearance is reachable under current contacts and dynamics. Large context capacity improves behavioral selection only when the dataset varies strategy independently of nuisance factors.

### Evaluation and next steps for humanoids

The strongest future experiment would hold task and scene fixed while intervening on strategy labels, then measure whether the requested physical strategy—not merely success—changes. Counterfactual metadata and deliberately balanced successes/failures would test this. For humanoids, π0.7 should generate feasibility-aware task-space subgoals rather than raw whole-body motor commands. Support-state tokens, contact history, and fall-risk estimates could be added alongside cameras. Calibrated uncertainty should trigger a safe lower-body behavior or human query when context is contradictory. Distilling the large model into a faster rolling-horizon planner would make those corrections available at contact-relevant rates.

### Why the richer observation still leaves a control gap

Four views, long image history, proprioceptive history, and visual subgoals give π0.7 far more context than a single-frame VLA. MEM compression is the enabling mechanism: it turns temporal evidence into a bounded token set rather than making attention cost grow linearly with every image. Independent history/view dropout is also a practical robustness measure because real robot streams fail asynchronously.

Yet perception breadth does not imply physical feedback bandwidth. Widely spaced historical frames capture task stage and slow change, not millisecond contact transients. A visual subgoal can say what a successful future should look like but not what wrench or support transition safely reaches it. On a humanoid, those variables must enter through a lower-level controller or a richer action interface.

Evaluation of emergent behavior should distinguish recombination from retrieval. Nearest-neighbor analysis over scenes, language, and action chunks can test whether “unseen” tasks are genuinely novel compositions. Cross-embodiment results should report how much adaptation data each projector/action head receives. Strategy steering is most convincing when the same observation yields measurably different, requested behaviors while controlling for dataset source and operator.

### Comparative position

π0.7 extends the generalist-policy axis rather than the humanoid-control axis. Its context can choose a strategy, while CEER, HANDOFF, OmniH2O, or a MotionWAM-like decoder realizes that strategy safely. Compared with language alone, visual subgoals describe desired state more densely; compared with symbolic plans, they remain ambiguous about contact and feasibility. The systems question is not whether π0.7 replaces a controller, but which physical interface best exposes its contextual competence.

## π0: A Vision-Language-Action Flow Model for General Robot Control

Source: [arXiv 2410.24164](https://arxiv.org/abs/2410.24164).

[π0](https://arxiv.org/abs/2410.24164) joins a 3B-parameter PaliGemma vision-language backbone with a roughly 300M-parameter action expert. Observation contains two or three current RGB views, a language command, and joint-angle proprioception. Images/language use the pretrained expert; state and noisy action tokens use the smaller robotics expert, with both interacting through shared transformer attention.

It predicts a 50-step continuous action chunk. Conditional flow matching interpolates Gaussian noise toward demonstrated actions; action tokens use bidirectional attention and an MLP embeds action plus sinusoidal flow time. Ten Euler integration steps produce the chunk. The block-causal mask separates image/language, state, and action blocks so observation KV caches are reused during denoising. The PaliGemma side has width 2048/depth 18; the action expert uses width 1024 and MLP dimension 4096.

Pretraining mixes about 10,000 hours over seven robot configurations and 68 dexterous tasks with OXE/DROID/Bridge data, followed by curated post-training. At 50 Hz the system replans after executing 25 actions (0.5 s); chunks are open-loop between calls because temporal ensembling hurt performance. Reported RTX 4090 latency is 73 ms onboard or 86 ms offboard.

π0 is relevant here as a semantic/high-level action generator, not a native humanoid balance controller. Its training robots are manipulators/mobile platforms and its action semantics vary by embodiment. HAF and HOIST therefore wrap or adapt this class of model rather than directly sending its outputs to humanoid motors. Whole-body stability, contacts, falls, and high-rate disturbance rejection need an additional interface/controller.

### What is genuinely important about π0

π0's main contribution is the combination of broad vision-language pretraining with a continuous, high-dimensional action generator. Tokenizing robot actions as ordinary language symbols is awkward for smooth control; conditional flow matching instead models a distribution over entire continuous chunks. The smaller action expert preserves a robotics-specific computation path while still attending to semantic features from the large pretrained model. Block-causal caching is not merely an implementation detail: without reusing observation computation across flow steps, a model of this scale would be much harder to run interactively.

Cross-embodiment pretraining is also meaningful, but it should not be confused with a single universal physical action space. Image and language features can transfer, while output dimensions, scaling, camera calibration, and action semantics remain robot-specific. The 50-step chunk improves temporal coherence and amortizes inference, yet executing half the chunk open-loop creates a 0.5-second interval in which unexpected contact cannot alter the high-level decision. This is acceptable for many tabletop motions and risky for balance-critical humanoid behavior.

### Critical assessment for humanoid use

π0 is a powerful prior for answering *what interaction should occur next*, but a poor direct answer to *which torques keep a humanoid safe now*. A useful humanoid adaptation should output a constrained task-space interface—hand/object goals, contact schedules, or motion latents—consumed by a high-rate whole-body controller. A learned feasibility critic could reject commands that violate reach, support, or collision constraints. RTC-style asynchronous replanning, force/tactile conditioning, and shorter adaptive chunks would improve contact responsiveness. Most importantly, the model needs substantial humanoid fall, recovery, locomotion, and whole-body-contact data; parameter scale cannot substitute for missing dynamics coverage.

### Data, action normalization, and evaluation cautions

Combining seven robot configurations requires embodiment-specific action dimensions and normalization. The common transformer can share semantic/visual structure, but each dataset's cameras, control frequency, gripper convention, and action scale remain potential shortcuts. Balanced sampling is essential: a large easy dataset can dominate gradients without improving the rare dexterous behaviors that motivate the model.

Conditional flow matching represents a distribution rather than regressing the conditional mean, which is useful when a command admits multiple grasps. Ten Euler steps approximate the continuous flow; fewer steps reduce latency and may lower fidelity. The 50-step horizon supplies coherence, while replanning after 25 actions is the main feedback point. Evaluations should vary that execution length and include unexpected object motion to show when chunking becomes too open-loop.

For humanoid comparison, success alone is insufficient. One should report command feasibility rejection, balance interventions, falls, peak contact force, and how often the low-level controller modifies the VLA output. Those measurements reveal whether π0 contributes valid semantic plans or merely supplies rough proposals repaired by a powerful downstream controller.

### Comparative position

π0 is a general manipulation foundation model included here because newer humanoid systems reuse its design principles or backbone class. Unlike MotionWAM, it does not claim a unified humanoid motion representation; unlike HAF, it does not explicitly stage body-part generation; unlike RTC, it does not solve asynchronous execution semantics. Its value is broad semantic and visuomotor pretraining. For a humanoid stack, the safest role is high-level proposal generation with a documented task-space contract, not direct 29-joint control.


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
