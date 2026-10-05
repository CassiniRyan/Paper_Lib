# Teleoperation and Data — Paper Notes

This folder mixes teleoperation systems, interfaces, and datasets, so the commentary follows the real contribution of each work. For a teleoperation paper, the important questions are usually what the human commands, how the robot closes the loop, and where drift or embodiment mismatch enters. For a dataset paper, collection design, coverage, annotation quality, and downstream evidence matter more than inventing a nonexistent controller architecture. Local full texts were used when present; linked primary paper/project sources were used otherwise.

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

### Teleoperation-specific latency and data-quality interpretation

For demonstration collection, CLOT's global feedback is valuable only if timestamps across human mocap, gloves, robot localization, proprioception, camera, action, and control are retained. A visually successful episode with an unknown alignment offset can train an autonomous policy to act late. Dataset releases should therefore include raw and resampled clocks, dropped-frame indicators, calibration, tracker confidence, and both requested and executed robot actions. Recording the pre-shifted command is also necessary to distinguish operator intent from controller correction.

## HiPHI: A Large-Scale Benchmark for High-Precision Human Motion and Object-Interaction

Source: [arXiv 2608.16222](https://arxiv.org/abs/2608.16222), [dataset page](https://noitom-robotics.github.io/hiphi/).

HiPHI addresses the tradeoff between broad but physically weak internet video and precise but narrow laboratory mocap. It releases 617.5 hours/200.1 million frames at 90 Hz from 132 performers, including synchronized human motion, mesh-level object trajectories, object meshes, language descriptions, and metadata.

FrameNet linguistic frames and lexical units define systematic motion seeds; variations in direction, speed, amplitude, posture, involved body parts, objects, and contacts expand coverage rather than relying on ad-hoc scripts. Benchmarks measure motion diversity, interaction grounding, object consistency, tracking usefulness, scaling, and real G1 deployment.

A sub-millimeter optical marker pipeline records full-body and object motion. This is a dataset rather than a runtime policy/network.

HiPHI is valuable as a controlled motion/data foundation, especially for contact-aware imitation. Optical accuracy does not automatically make retargeted robot motion dynamically feasible, and the non-commercial research license affects downstream use.

### Dataset design, signals, and critical reading

HiPHI contains 617.5 hours and 200.1 million frames at 90 Hz from 132 performers. Sub-millimeter optical markers recover full-body motion; rigid/marker tracking yields mesh-level object trajectories, while calibrated object meshes, contact/interaction metadata, language and performer/session metadata remain synchronized. It is an offline source, not a runtime perception or control network.

Coverage is organized through FrameNet: semantic frames and lexical units seed actions, then direction, speed, amplitude, posture, body part, object and contact are varied systematically. This is more auditable than collecting whatever scripts operators invent. Benchmarks separately test motion-space diversity, contact grounding, object consistency, motion-model scaling and robot-policy usefulness, including G1 execution.

Optical precision reduces measurement noise but cannot resolve all contact force, friction, occluded marker or retargeting ambiguity. Hours are not independent samples, and scripted laboratory balance can underrepresent recovery and clutter. Dataset users should split by performer/session/action family, retain object/contact uncertainty, validate dynamics after retargeting and respect the research-only license. Future versions should add force/tactile data, scene geometry, failed attempts and embodiment-specific feasibility labels.

### Released schema, coverage statistics, and downstream policy interpretation

The release combines 308.7 hours of original capture with augmentation or mirroring to reach 617.5 hours, including 245.7 hours of human–object interaction. Forty physical objects across 12 categories span roughly 0.45–6.25 kg. Each sequence links BVH body motion to object trajectory and mesh, FrameNet frame and lexical unit, natural-language description, performer/session identifiers, frame rate, and object metadata. These fields permit splits by actor, semantic primitive, object, and capture session instead of leaking near-duplicate variants across train and test.

Coverage is quantified through a shared motion atlas rather than hours alone: occupied cells, effective occupancy, and rare-cell share expose whether a corpus repeatedly records common walking while missing unusual posture/contact. Reported statistics include 1,620 occupied cells, 1,443 effective occupancy, and a 14.1% rare-cell share. Interaction checks include near-surface grounding and conflict metrics; the project reports 98.1% non-conflict and 95.7% near-surface grounding. These remain geometric proxies, not measured contact force or dynamic feasibility.

The humanoid benchmark connects data scale to execution. Matched-budget tracker training, convergence, cross-dataset MPJPE, and real G1 behaviors test whether diversity survives retargeting and dynamics. Scaling from 3 to 300 hours should keep model, updates, and sampling controlled. Per-family survival, contact error, torque/impact, and rejected retargets are more informative than one mean.

HiPHI's highlight is treating mocap as designed coverage rather than incidental collection. FrameNet supplies a reproducible semantic scaffold, while controlled variation creates explicit axes. Remaining gaps are physical: no full force/tactile field, laboratory contexts, limited failed/recovery attempts, and human motions infeasible for robots. Future releases should add object inertial/friction data, force plates or tactile contact, scene geometry, uncertainty and feasibility labels, and licenses compatible with broader model development.

### Leakage-resistant splits, licensing, and use as teleoperation pretraining

Mirroring and controlled variants make leakage especially easy: sequences can differ in direction or amplitude while sharing performer, script, timing, and object. Random clip splits therefore overestimate generalization. Benchmark splits should hold out performers, sessions, Frame–LU combinations, object instances/categories, and entire variation axes, with near-duplicate detection in a shared motion embedding. Interaction evaluation should distinguish category transfer, a new mesh/mass, and genuinely new contact semantics.

As teleoperation pretraining, HiPHI can teach a tracker broad pose/contact priors before noisy live VR or vision. The decisive experiment would initialize identical teleoperation controllers with HiPHI, matched-hour alternative mocap, or no prior, then measure live tracking, falls, interventions, operator effort, and demonstrations collected per hour. Offline capture does not replace streaming data because it lacks latency, abrupt correction, dropout, operator viewpoint, and robot execution feedback.

The research license is non-commercial, relevant to foundation-model and deployment claims. Documentation should include skeleton definitions, object coordinates, calibration, processing scripts, versioned checksums, and known-error flags. Consent, intended use, performer coverage, and motion-risk protocols belong beside technical metadata. These details determine whether 200.1 million frames become dependable supervision or merely a large number.

## HOMIE: Humanoid Loco-Manipulation with Isomorphic Exoskeleton Cockpit

Source: [arXiv 2502.13013](https://arxiv.org/abs/2502.13013).

HOMIE combines a low-cost operator cockpit with a whole-body policy so one person can walk, squat to commanded height, and manipulate simultaneously. The isomorphic arm interface removes much of the ambiguity and retargeting error of generic IK.

Hardware consists of joint-matched exoskeleton arms, motion-sensing gloves, and a locomotion pedal. An RL policy tracks arbitrary upper-body poses while controlling walking and body height; upper-body pose curriculum, a height reward, and symmetry augmentation produce robustness without motion priors. Collected demonstrations also train autonomous imitation policies.

Joint-aligned exoskeleton sensing and gloves provide commands; robot proprioception closes low-level control. Visual feedback supports the operator but is not the core learned input.

Isomorphic hardware improves precision and intuitiveness, but ties the cockpit to a morphology and adds wearable/setup burden. Pedal-driven locomotion also transfers less of the operator's natural lower-body intent.

### Cockpit-to-policy data flow

Each 3-D-printed exoskeleton arm has seven joint-matched DoFs—three shoulder, elbow, and three wrist—with absolute Dynamixel sensing around 0.09-degree resolution. Fixed coordinate/offset mapping sends angles directly to matching G1 or GR-1 joints, avoiding iterative IK. Gloves provide up to 15 finger DoFs. A pedal supplies locomotion/squat commands, so one seated operator controls arms, hands, travel and height.

The PPO locomotion policy observes robot proprioception, commanded velocity/yaw, desired body height and arbitrary upper-body joint targets, and outputs lower/whole-body PD targets. A curriculum gradually increases upper-body pose range; targets change each second with interpolation so balance is learned under continuous arm motion. One third of environments train squat while the rest walk/stand, with a knee-aware height reward. Mirrored transitions and actor/critic symmetry losses double useful experience and reduce directional bias. No mocap prior is required.

Direct isomorphism improves precision and task time, and demonstrations train autonomous imitation policies. But it shifts retargeting cost into custom hardware per arm morphology; pedal commands omit natural stepping and force feedback. Adjustable linkage calibration, bilateral haptics, foot/CoM intent sensing, collision feedback and cross-robot adapter evaluation would test whether the cockpit scales beyond its matched embodiments.


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

## HumanPlus: Humanoid Shadowing and Imitation from Humans

Source: [arXiv 2406.10454](https://arxiv.org/abs/2406.10454).

HumanPlus is a full pipeline from human movement to real humanoid teleoperation data and autonomous skills. A custom 33-DoF, 1.8 m robot shadows body and hand motion from a single RGB camera; behavior-cloned egocentric policies then perform tasks such as clothing handling, warehouse unloading, typing, and interaction.

A low-level RL tracker is trained in simulation on roughly 40 hours of human motion, retargeted across morphology, and transferred to hardware. RGB-based human pose drives that universal tracker in real time. Shadowing becomes a data-collection interface; task policies learn from up to 40 demonstrations using robot egocentric observations.

Monocular RGB human pose/hand estimation drives the tracking policy. Autonomous policies use egocentric vision and supervised behavior cloning, while low-level stabilization is proprioceptive.

HumanPlus clearly connects motion tracking to autonomous learning. Monocular occlusion/depth ambiguity and behavior cloning's distribution shift remain bottlenecks, and successful shadowing does not guarantee safe object contact.

### From shadowing to autonomous data

The low-level Transformer tracker is trained through simulation RL on roughly 40 hours of retargeted motion. It receives robot proprioception plus desired body/hand pose and outputs joint targets for the custom 33-DoF, 1.8-m platform. At runtime a monocular RGB estimator reconstructs the human and drives that kinematic interface; raw pixels are not processed by the tracker.

Shadowing records egocentric RGB, robot state and executed robot actions, eliminating a major embodiment mismatch in the later behavior-cloning data. With up to 40 demonstrations, autonomous policies achieve reported 60–100% success on shoe wearing, unloading, folding, typing and interaction tasks. Remaining errors originate in monocular depth/occlusion, retargeted contacts and BC compounding. Depth/multiview fusion, pose uncertainty, tactile/object feedback and DAgger-style autonomous correction are natural extensions.


### Shadowing as a data engine

A Transformer-based low-level controller is trained with RL on AMASS-scale retargeted motion, taking robot proprioception plus target body/hand pose and outputting joint targets. At runtime an external monocular RGB estimator reconstructs the operator; retargeting converts human joints/hands to the custom 33-DoF, 180-cm humanoid. The tracker consumes kinematics, not pixels, and stabilizes morphology/actuation mismatch in feedback.

Shadowing records egocentric RGB, robot state and executed whole-body action in the robot's own distribution. Supervised behavior cloning then trains autonomous visual policies for shoe wearing, warehouse unloading, garment folding, rearrangement and typing with up to 40 demonstrations, reporting 60–100% success. This closes a valuable loop: human video drives a physics tracker, and tracker rollouts become embodiment-aligned imitation data.

The main bottleneck is information quality. Monocular depth/occlusion and hand estimation can issue impossible targets; contacts seen in a human body do not transfer automatically; behavior cloning compounds error. Future systems should fuse depth/multi-view or inertial cues, close object/tactile feedback, attach uncertainty to pose goals, and compare autonomous data value against direct robot teleoperation.


### Separation between shadowing control and autonomous skill learning

HumanPlus contains two distinct policies that should not be conflated. The simulation-trained whole-body tracker maps retargeted human body/hand goals and proprioception to motor/joint targets; it enables real-time shadowing and remains responsible for balance. The downstream autonomous policy is trained by supervised behavior cloning on egocentric RGB plus robot state/action collected during those shadowing rollouts. It learns task-specific visual decisions but delegates low-level feasibility to the same embodiment-aligned action interface.

This data path has an important advantage over simply training on human video: the observations are captured from the robot camera, actions are the commands actually executed by robot actuators, and failures caused by morphology are visible in the demonstration distribution. Up to forty demonstrations per task can therefore be useful despite the modest count. The 33-DoF, 180-cm custom humanoid and roughly 40-hour motion prior also provide hand/body coordination beyond lower-body-only teleoperation.

However, shadowing quality sets a ceiling on behavior-cloning data. A human may compensate visually for a robot error during demonstration, creating correlated corrections that an autonomous policy cannot interpret without the operator. Dataset documentation should record operator view, latency, retargeting confidence, and intervention. Closed-loop imitation methods, recovery demonstrations, and action-chunk or diffusion heads could reduce compounding error, while independent low-level safety limits should remain active when the visual policy encounters a novel object layout.

### Demonstration value and operator–policy separation

HumanPlus data should distinguish signals from the operator, retargeter, low-level tracker, and autonomous visual policy. Executed motor actions are embodiment-aligned, but operator corrections can correlate with tracking error and create demonstrations an autonomous policy cannot interpret. Logging pose-estimation confidence, tracker residual, operator view, intervention, object state, and action latency supports filtering and recovery learning. Train/test splits should separate operators, objects, and layouts, not only episodes.

Recovery demonstrations and deliberate perturbations should be collected alongside successes so autonomous policies learn what to do after visual or tracking error rather than only reproducing nominal shadowing trajectories.

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

## MOSAIC: Bridging the Sim-to-Real Gap in Generalist Humanoid Motion Tracking and Teleoperation with Rapid Residual Adaptation

Source: [arXiv 2602.08594](https://arxiv.org/abs/2602.08594).

MOSAIC builds a general motion tracker that remains robust during long-horizon hardware teleoperation across multiple interfaces and realistic noise/latency. It supports offline replay and online agile motions, including jumps and single-leg support.

A general RL tracker learns from optical mocap, inertial mocap, AMASS/OMOMO, and generated motion with adaptive resampling and world-frame consistency rewards. For a new interface, a small interface-specific policy learns residual corrections from little data; multi-teacher distillation inserts the correction as an additive residual without overwriting general competence.

Interface signals can come from VR or inertial mocap; the low-level policy uses proprioception and reference motion. Residual adaptation explicitly models interface and dynamics mismatch.

The approach offers a practical compromise between universal training and device-specific tuning. Additive residuals may not cover fundamentally new reference semantics, and each interface still requires representative adaptation data.

### Residual adaptation as interface calibration

The general PPO tracker learns world-frame-consistent motion from optical/inertial mocap, AMASS/OMOMO and generated references with adaptive hard-motion resampling. A small policy then trains on a particular VR/IMU interface's latency, bias, noise and dropout. Multi-teacher distillation adds its correction as a residual to the frozen general action rather than overwriting the base.

This separation preserves offline replay and unrelated motions better than full fine-tuning while making long-horizon hardware teleoperation less brittle. It also defines the limitation: a residual can correct calibration/dynamics error but cannot invent a missing semantic variable or repair a reference outside the base manifold. Residual magnitude should be uncertainty-gated and safety-bounded; adapter composition, data-efficiency curves, and comparison with improved calibration would make the interface claims stronger.


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

## OASIS: From Simulation Data Collection to Real-World Humanoid Loco-Manipulation

OASIS replaces expensive real-robot collection with VR teleoperation entirely in simulation and deploys the learned visual policy zero-shot. The paper reports higher real-task success than policies trained on its real-robot comparison data for most tasks, attributing this to broader rendering variation.

A 3D generative model reconstructs object assets from photos and a VLM estimates scale/material properties. PICO 4U controls a simulated humanoid; only states are recorded during lightweight real-time rendering. Offline replay creates high-quality observations with randomized texture, lighting, and camera extrinsics. A transformer/action-chunking flow-matching high-level policy predicts motion commands, and a low-level tracker produces joint targets.

Head RealSense D435i plus wrist D405 cameras provide multimodal visual observations on hardware; proprioception feeds the controller.

Decoupling teleoperation from offline rendering multiplies data cheaply. Transfer remains sensitive to asset physics, contact, hand modeling, and visual factors not covered by randomization.

### From asset reconstruction to a 43-DoF hierarchy

Photos feed a 3-D generative model; a VLM proposes object scale/material attributes, then PICO teleoperation records clean state trajectories in simulation. Expensive photorealistic observations are rendered afterward, allowing each state path to be replayed with randomized background texture, lighting intensity/color temperature and camera extrinsics without slowing the operator or redoing contacts.

The high-level Transformer/flow-matching policy conditions on frozen CLIP text, frozen DINOv2 features from head plus two wrist cameras, and two frames of reference-command history encoded by an MLP. It predicts 32 frames of a 67-D robot-native motion representation—root roll/pitch encoding, yaw/translation increments, height, 29 joints and joint increments. Ten Euler denoising steps run at 25 Hz. Teleopit's 50-Hz tracker realizes the 29-DoF body while 14 hand joints yield 43-DoF output.

Curriculum rollout reuses prior predictions across four segments, increasing self-history probability from zero to 0.8. Conditioning on commanded rather than measured state avoids feeding tracking noise into planning, but also hides physical divergence. Hardware uses D435i head and D405 wrist cameras. Simulation collection is 1.15–1.84 times faster for 50 successes and often transfers better than smaller real data, but model scale/material/contact errors remain. Future work should feed bounded execution error, randomize dynamics/occlusion beyond appearance, and validate generated assets with real interaction measurements.


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

## PHUMA:Physically Reliable Humanoid Locomotion Dataset

PHUMA cleans internet-scale human motion for stable robot imitation. Its 73-hour corpus combines mocap and video-derived motion; PHUMA-trained trackers outperform AMASS/Humanoid-X training and transfer zero-shot to Unitree G1 with 16.3% lower reported real tracking error than the AMASS-trained comparison.

Physics-aware curation removes root jitter, unsupported/object-dependent motions, and other infeasible sequences. PhySINK retargeting adapts human shape while softly enforcing joint limits, ground contact, anti-penetration, non-floating, and anti-skating constraints. MaskedMimic and BeyondMimic trackers test downstream utility.

PHUMA is an offline dataset pipeline built from AMASS and recovered internet-video motion; runtime tracking is proprioceptive plus reference trajectories.

The paper emphasizes that scale without physical reliability can hurt control. Filtering improves feasibility but can remove useful hard cases, and optimization constraints do not ensure every trajectory is dynamically realizable on every robot.

### Data construction and what it proves

Long sources are split before jerk, support-region, ground-contact and object-dependence filtering, preserving clean fragments. PhySINK fits robot shape and joints with soft limits, grounded inferred contacts, non-penetration and anti-skating. The result is about 76,000 clips/73 hours from Humanoid-X, LaFAN1, LocoMuJoCo and captures; only 69.1 of 231.4 raw Humanoid-X hours survive comparable screening.

Matched MaskedMimic/BeyondMimic trackers isolate corpus quality, including unseen and real G1 tests. This supports physically validated scale, not an online data interface. Static-ground IK may discard rare valid dynamics and does not check actuator/contact forces. Future releases should include provenance, contacts and confidence, source-family splits, forward-simulation acceptance and scene geometry for supported interaction.


### Curation and PhySINK

PHUMA splits long sources so a bad interval does not discard an entire clip, rejects excessive jerk, estimates ground, detects floating/penetrating contact, and removes object-supported poses such as chair sitting when no scene object is provided. Sources include Humanoid-X, LaFAN1, LocoMuJoCo, and captured motion. Shape-adaptive IK then combines keypoint fit with soft joint bounds, grounded feet during inferred contact, non-penetration, and anti-skating consistency.

The release contains about 73 hours and 76,000 clips, compared with only 69.1 usable hours retained from 231.4 raw Humanoid-X hours. Identical MaskedMimic/BeyondMimic-style trackers trained on alternative corpora are tested on unseen motion and real G1, isolating the contribution of reference quality.

The key metric is usable scale after physical validation. But static-ground kinematic constraints do not prove torque/contact feasibility and may filter rare valid dynamics into a conservative corpus. Future releases should retain scene geometry, contact labels, provenance and uncertainty; forward-simulate acceptance; and split by source/motion family to avoid near-duplicate leakage.


### Filtering tests, benchmark design, and dataset bias

The curation stage distinguishes several failure mechanisms rather than using one reconstruction score. It detects excessive temporal jerk, seated/object-supported behavior, a center of mass outside an estimated base of support, insufficient foot–ground contact, floating feet, below-ground penetration, and sliding during inferred stance. This matters because a visually plausible pose estimator can score well per frame while producing an impossible support sequence. Long videos are segmented so clean intervals survive even if another interval fails.

PhySINK then retargets body proportions while jointly minimizing human–robot keypoint error, joint-limit violation, contact-height error, penetration, and stance-foot displacement. These terms are kinematic but temporally coupled: stance consistency prevents the independent-frame IK solution from moving a planted foot to improve upper-body fit. Evaluation trains the same PPO imitation recipe on LAFAN1, AMASS, Humanoid-X, or PHUMA for both 29-DoF G1 and 21-DoF H1-2, then tests held-out motion families; this controls tracker architecture when comparing data quality. A latent-augmented multi-motion variant checks that conclusions persist beyond a simple full-state policy.

The reported 73 hours and roughly 76,000 clips are therefore *post-curation usable data*, not merely scraped duration. However, filtering can encode a flat-ground, no-object definition of reliability and disproportionately remove sitting, hand support, acrobatics, disabled gait, or culturally uncommon movement. A better release would distribute rejection reason codes and pre/post clips, scene/support metadata, annotator confidence, and evaluation partitions for intentionally contact-rich behavior. That would let future controllers expand physical scope instead of inheriting the curator's exclusions invisibly.

### Dataset-quality role and what PHUMA does not contain

For teleoperation, PHUMA is most useful as prior data for the low-level tracker that converts imperfect live commands into stable motion. It is not itself a teleoperation dataset: it lacks synchronized operator view, object/task state, network timing, and executed demonstrations. Evaluations should compare a tracker trained on raw versus curated references under identical live VR/RGB noise and report fall, lag, foot slip, and intervention. That quantifies whether physical curation improves collection throughput rather than only offline imitation.

Tracker training should also publish the fraction of source hours rejected or modified, since excellent results on a retained subset can hide lost rare contacts, kneeling, or asymmetric behaviors valuable to operators.

## Teleopit: A Full-Embodiment Humanoid Teleoperation System

Source: [arXiv 2608.01834](https://arxiv.org/abs/2608.01834), [code](https://github.com/BotRunner64/Teleopit).

Teleopit maps VR body, hand, and head signals to humanoid locomotion, configurable dexterous hands, and a 2-DoF active-vision neck. ACT and GR00T N1.7 policies trained on 96 successful demonstrations reportedly reach 90% and 95% task success.

A history encoder and failure-aware rewind sampling improve the body tracker. A hand retargeting optimizer combines normalized finger direction, fingertip closure, and thumb-frame alignment, allowing hand models to change without retuning solver weights. Timestamp-aligned streaming, safety handling, synchronized HDF5/video recording, and matched robot/policy assets form a reproducible systems contribution.

PICO VR supplies body/head/hand commands; robot RealSense/active neck supply operator and policy vision; joint/hand/neck feedback supports execution.

Teleopit is unusually complete at the peripheral, timing, recording, and training-policy levels. VR hand estimation and network timing remain failure points, and optimization-based retargeting still cannot guarantee collision-free grasp contact.

### Body, hand, vision, and asynchronous runtime

PICO supplies a 24-joint body skeleton, 26 keypoints per hand and head pose. A PPO body actor observes current reference, joints, base angular velocity, gravity and prior action; a ten-step 1-D convolution plus global pooling adds recent reference/proprioceptive context. It emits 29 scaled offsets around default joints at 50 Hz, tracked by PD at 200 Hz. The critic alone sees global position, linear velocity and 14 link poses. Failure-aware rewind retains a failed clip with probability 0.8 and restarts shortly before failure, focusing on difficult live-VR transitions.

Hand SLSQP optimization aligns normalized finger directions, activates a one-sided thumb–fingertip distance objective near 4 cm, and aligns a thumb base frame to preserve opposition. Joint limits, previous-solution warm start and temporal smoothing apply. Semantic link names are configured per hand, but weights/thresholds remain shared across morphologies. Head motion commands a two-DoF camera.

Timestamped TCP carries body/hand/head/control, WebRTC returns video, and latest-only asynchronous queues prevent stale backlogs. Ninety-six bottle-placement demonstrations train ACT and GR00T N1.7 to 90% and 95% success. Remaining concerns are VR occlusion, optimization deadline misses, lack of force/contact guarantee and hidden stream desynchronization. Future work should add tactile feedback, collision-aware hand/body co-optimization, explicit freshness/confidence inputs and failure recovery in recorded policies.

### Full-embodiment coordination, recording schema, and evaluation

Teleopit treats body, hands, and head as coupled but differently controlled channels. The body tracker handles balance from the VR skeleton; SLSQP maps finger geometry to configurable hand joints; neck commands control viewpoint. Coordination matters because an operator turns the camera while reaching and stepping, and independent queues can execute those intentions at different ages. Every packet and log sample should include source timestamp, receive time, sequence number, confidence, and applied robot time; stale commands should be exposed or discarded.

Failure-aware rewind sampling is a practical curriculum. Rather than restarting a failed reference from its beginning, training returns shortly before the transition with high probability, concentrating rollouts on abrupt live-VR changes. Ablations should compare uniform frames, failure oversampling, rewind probability/window, and history length on both mocap and raw VR. Report body success, hand objective error, neck tracking, falls, missed deadlines, and operator correction rate.

Normalized finger directions reduce dependence on finger length, closure terms preserve grasp aperture, and thumb-frame alignment captures opposition. Cross-hand claims should test different kinematic layouts with unchanged weights, fast motion, self-collision, and occlusion. Geometry does not guarantee stable contact; tactile feedback, torque limits, and collision-aware body/hand co-optimization are natural additions.

The 96 successful demonstrations connect teleoperation to autonomy: ACT and GR00T N1.7 reportedly achieve 90% and 95% deployment success. Stronger evidence includes failed/intervention episodes, learning curves versus count, held-out objects/layouts, and comparison with another interface at equal operator time. Setup, reset, calibration, and discarded trials must enter demonstrations per hour. The released asynchronous runtime is a genuine contribution, but long-horizon recovery and haptics remain open.

### Operator ergonomics, failures, and autonomy-facing data quality

Full embodiment increases operator cognitive and physical load. The operator coordinates walking, two hands, and gaze while watching delayed robot video; neck control can improve view but create visual–vestibular mismatch. Studies should report training time, fatigue, nausea, correction frequency, completion, and performance across novice and expert operators. Comparing active-neck and fixed-camera modes would quantify whether viewpoint control improves grasp success enough to justify another channel.

Only successful demonstrations are insufficient for robust imitation. The runtime should label tracker fall, hand optimizer failure, occluded fingers, stale packets, safety stop, collision, intervention, and recovery. Keeping raw failed episodes permits failure prediction and corrective-policy learning even if a clean subset trains behavior cloning. Splits should separate operator, object instance, pose, and scene; learning curves versus operator-hours are more informative than one 96-demo checkpoint.

The ACT and GR00T results also test the action schema. If policies predict the same high-level body/hand/neck command used by teleoperation, the tracker remains a reusable stabilizer; if they predict executed joints, they may overfit one robot/controller. Releases should distinguish raw command, optimized hand solution, desired joints, measured joints, images, timestamps, and overrides. This makes Teleopit an auditable conversion from human intent to autonomy-ready robot data.

## TWIST2: Scalable, Portable, and Holistic Humanoid Data Collection System

TWIST2 makes full-body teleoperation portable and mocap-free while preserving egocentric active vision. It reports collecting 100 successful demonstrations in 15–20 minutes and demonstrates long-horizon mobile/dexterous manipulation and kicking.

PICO 4U tracks the operator's body; a low-cost custom 2-DoF robot neck supplies egocentric viewpoint control. Full-body tracking replaces decoupled base/arm control. A hierarchical visuomotor learner uses a high-level visual policy and a low-level whole-body tracker; the action-generation component uses diffusion/action chunking for temporally coherent commands.

VR supplies operator motion; the neck-mounted egocentric camera streams to the operator and autonomous policy. Robot proprioception closes control.

TWIST2 directly targets setup time and throughput, two often ignored research constraints. VR tracking accuracy and calibration are weaker than optical mocap, so policy robustness and data-quality screening are essential.

### Multi-rate hierarchy and learned policy

PICO streams body estimates up to 100 Hz; retargeting and a convolutional-history-plus-MLP tracker convert recent proprioception/reference into 50-Hz joint targets, with PD below. Training uses about 20,000 clips, including 7,000 GMR motions and TWIST data. The two-DoF neck actively aims the egocentric camera rather than fixing gaze to torso heading.

Collected RGB/state/action feeds a high-level Diffusion Policy: images and proprioception condition temporal convolutions that predict 64-command chunks around 20 Hz, played as motion goals around 30 Hz. The low-level controller absorbs high-level timing noise. Throughput is impressive, but VR drift and calibration can silently lower data value. Logging confidence and measuring downstream success per collection hour, plus global/object/tactile closure, would strengthen the scale claim.


### Portable reference stream and low-level tracker

PICO supplies head, wrists and body estimates at up to 100 Hz; a retargeter converts them to robot motion. The learned low-level controller runs at 50 Hz and sends desired joints to PD. It is trained on about 20,000 clips—roughly 7,000 GMR-retargeted motions plus TWIST data—with tracking and small action regularizers. A convolutional history encoder compresses recent proprioception and reference motion before an MLP, helping smooth portable-sensor noise and delay. The active two-DoF neck moves the egocentric camera independently enough to inspect hands/workspace while maintaining whole-body control.

The same system collects RGB/proprioception/action demonstrations for a high-level visuomotor Diffusion Policy. That policy uses image plus normalized proprioception, predicts 64-command chunks with temporal convolution, runs around 20 Hz, and supplies motion references at 30 Hz; the tracker closes the faster physical loop. Hierarchy prevents visual inference jitter from directly becoming torque.

TWIST2's importance is throughput and one-operator portability, not better optical accuracy. VR drift, body self-occlusion, network latency, hand mapping, and accumulated global error can contaminate demonstrations. Future work should log calibration confidence, close global/object pose, integrate tactile hands, and measure downstream policy success per hour of collection rather than only teleoperation completion.


### Hierarchy, action bandwidth, and data-quality judgment

The autonomous stack deliberately predicts a motion-level command rather than raw motor torques. Egocentric stereo images and normalized proprioception enter a Diffusion Policy that emits chunks of base velocity/orientation, full-body pose, neck, and hand commands; the general tracker converts those commands to stable motor targets at the faster control rate. Chunking (the reported policy predicts 64-command sequences) reduces visual-action jitter and supplies short-horizon intent, while the low-level history encoder absorbs sensor noise and closes balance. This division is especially suitable for mobile manipulation because visual reasoning need not run at the servo frequency.

The active two-DoF neck is more than a camera mount. It decouples gaze from torso pose, allowing the operator and autonomous policy to inspect hands or the next foothold without forcing whole-body retargeting to turn the robot. The released state/action representation includes planar base velocity, base height, roll/pitch and yaw rate, body joints, hands, and neck; synchronized stereo images and last action make the dataset useful for closed-loop imitation. Reported collection rates—about 100 successful pick-and-place demonstrations in 15–20 minutes and roughly 50 mobile demonstrations in 20 minutes—show the throughput advantage over studio MoCap.

High success during collection does not automatically mean high information diversity. Action chunks can blur corrective timing, one environment can produce correlated backgrounds, and VR pose estimation may introduce systematic morphology bias that the tracker quietly repairs. The right comparison is autonomous success per operator-hour against MoCap, bilateral teleoperation, and scripted collection, with calibration/setup time included. Dataset audits should publish latency distributions, tracker residual error, camera coverage, intervention/collision labels, operator diversity, and train/test scene separation. Those measurements would establish whether TWIST2 scales not only the count of demonstrations but also the effective diversity needed for general visuomotor learning.

### Dataset throughput and independence of collection quality

TWIST2's collection rate should be decomposed into successful episode time, reset, operator correction, calibration, discarded trials, and setup. One hundred nearly identical successes can be less useful than fewer demonstrations spanning object pose, route, and recovery. Data should retain stereo images, normalized proprioception, body/hand/neck commands, executed low-level action, timestamps, and failure/intervention labels. Autonomous success per operator-hour on held-out scenes is the strongest scalability measure.

## TWIST: Teleoperated Whole-Body Imitation System

TWIST demonstrates unified whole-body humanoid teleoperation and uses collected demonstrations for autonomous visuomotor imitation. It emphasizes coordinating locomotion and manipulation rather than commanding a mobile base beneath an independent upper body.

Optical human mocap is retargeted to a dynamically robust whole-body tracking policy. The operator sees robot egocentric video, and synchronized state/action/image trajectories train task policies. The control hierarchy separates human-reference interpretation, whole-body stabilization, and learned autonomous behavior.

An RL motion tracker executes retargeted human pose; imitation policies consume egocentric images and proprioception, typically with temporal action prediction/chunking. Optical mocap provides accurate teleoperation commands.

TWIST established a high-quality full-body data pipeline, but its mocap studio is expensive and slow to relocate—the exact limitation addressed by TWIST2.

### Anticipatory teacher, causal student

The PPO teacher observes two seconds of future retargeted reference from AMASS, OMOMO and 150 custom clips, learning smooth anticipation. The deployable student receives proprioception and only the immediate live target and learns through joint RL plus teacher-action behavior cloning on its own visited states. This eliminates the train/deploy mismatch that makes a future-conditioned policy hesitate when streaming mocap cannot provide the future.

Optical mocap runs at 120 Hz, the tracker at 50 Hz and PD at 1 kHz. One network coordinates locomotion, torso and arms while egocentric video is logged. The student cannot infer genuinely unpredictable operator intent; it may smooth sudden decisions. Explicit intent forecasting, confidence-aware slowing, portable sensing and global drift closure are the next steps.


### Teacher, student, and live interface

Public AMASS/OMOMO and 150 in-house mocap clips are retargeted, with IK cleanup for foot placement and body orientation. A privileged PPO teacher sees two seconds of future reference, allowing anticipation of fast turns, squats, and contacts. The deployable student sees proprioception and only the immediate target; it is optimized jointly with PPO tracking reward and behavior-cloning/distillation from the teacher. This avoids the hesitation produced when a live controller is trained to expect future frames that streaming mocap cannot provide.

Mocap streams at 120 Hz, teleoperation policy at 50 Hz, and joint PD at 1 kHz. One network outputs whole-body joint targets, coordinating stepping, torso and arms while synchronized egocentric RGB/state/action logs become imitation data. Domain randomization and action/joint penalties support transfer.

TWIST's scientific contribution is aligning training information with live availability while retaining an anticipatory teacher. The student cannot reconstruct genuinely unknowable operator intent, so distillation learns the average best response and may smooth sudden changes. Optical infrastructure gives excellent pose but constrains workspace and scale. Portable sensing, explicit intent prediction, global drift correction, and uncertainty-aware slowing are natural successors.


### RL+BC architecture and why real-time data matters

TWIST represents each target by retargeted humanoid joint positions and root velocity rather than raw human skeletal coordinates. A privileged teacher trained with PPO receives robot state plus roughly two seconds of future reference, which lets it prepare support changes before a squat, kick, or lateral step. The causal student receives the present streaming target and proprioception and is optimized with both the physical tracking reward and behavior cloning from the teacher. BC transfers anticipatory structure where it is predictable from the current pose, while continued RL prevents the student from merely copying actions that are unsafe in states it reaches under its restricted observation.

Adding live in-house MoCap is important even when public AMASS/OMOMO motion exists. Real teleoperation contains pauses, reversals, calibration error, abrupt operator decisions, and transitions between reaching and walking that curated clips underrepresent. Training on those signals narrows the interface distribution gap. A single whole-body action vector then coordinates feet, torso, and arms at 50 Hz, with faster joint PD underneath, instead of switching between locomotion and manipulation controllers.

The main achievement is coordinated embodiment: carrying a box while walking, crouching to reach the floor, kicking, sideways locomotion, and expressive dance all use one learned balance mechanism. It is not yet autonomous adaptation—the operator sees the scene and supplies intent, and the controller has no explicit object geometry or force objective. Future work should add time-stamped latency compensation, predict a distribution over short future human motion, fuse force/tactile feedback, and train recovery demonstrations. Evaluation should separate operator error, retargeting error, and low-level tracking error and report contact/torque peaks, task completion, and correction latency in addition to pose fidelity.

### Information alignment and demonstration-learning implications

TWIST's log should distinguish raw human pose, retargeted robot reference, teacher/student input, desired joint action, measured state, and egocentric images. These layers diagnose whether autonomous failure comes from visual imitation, operator ambiguity, retargeting, or tracking. Precise timestamps at 120-Hz mocap, 50-Hz policy, and 1-kHz servo are essential; resampling should preserve original clocks and interpolation masks.

## X-OP: Cross-Morphology Whole-Body Teleoperation via MPC Retargeting

X-OP enables online teleoperation when operator and robot body proportions/kinematics differ substantially. It seeks feasible whole-body commands that preserve intent while respecting balance and contact.

Model-predictive retargeting optimizes a short horizon instead of independently solving each frame. Human targets, robot dynamics/kinematics, contact, smoothness, and feasibility are combined so locomotion and manipulation remain coordinated. A low-level whole-body controller tracks the receding-horizon solution.

Human pose comes from a teleoperation capture interface; the optimizer combines it with robot state and contact estimates. It is model-based rather than an image-to-action network.

MPC gives interpretable constraints and anticipates near-future feasibility, but computation, model error, contact-mode selection, and tuning can limit agility. It complements learned trackers when hard constraints matter more than motion-style fidelity.

### Closed-loop MPC retargeting details

Apple Vision Pro head/wrist targets enter a sampling-based horizon optimizer over the compact commands of an existing low-level controller, not raw torque. Rollouts include simulator plus policy; costs cover target alignment, smoothness, energy, stance and collision. Sequential state synchronization uses SLAM, IMU and encoders so the simulated prediction remains anchored to hardware.

For the humanoid/FALCON instance the high-level action is seven-dimensional, MPC runs at 25 Hz standing and 10 Hz walking, and the learned low-level controller at 50 Hz. The same formulation drives a wheeled manipulator by replacing its action interface/model, supporting the cross-morphology claim. Worst-case solve time, model/contact uncertainty and fallback behavior remain critical; chance constraints and learned residual dynamics could improve robustness without losing interpretability.

### Optimization interface and cross-morphology significance

X-OP does not optimize raw torque. The XR device supplies head and wrist goals, and sampling-based MPC searches the compact command space understood by an existing low-level controller. For FALCON the command is seven-dimensional; the learned 50-Hz controller, including state–action history, realizes balance and contact. A wheeled manipulator exposes a different command interface, but the retargeting objective remains applicable, avoiding a new end-to-end human-to-robot policy for every morphology.

Each update synchronizes the simulated rollout state to encoders, IMU, and SLAM pose. This reset is central: contact simulation is sensitive, and internal drift would optimize a horizon unrelated to hardware. Costs trade head/hand alignment against posture or locomotion preference, smoothness, energy, collision, and stance/contact constraints. Prediction can accept temporary hand error to take a feasible step, something independent-frame IK cannot reason about.

The cross-morphology result therefore means reuse of the optimizer above platform-specific low-level controllers, not one universal motor policy. Reported simulation gains include over 30% lower completion time and 20% lower power for the humanoid and zero collisions for the mobile manipulator. Preference weights also allow behavioral customization without retraining. Fair comparisons must hold low-level control constant so improvements are attributable to MPC.

### Latency, safety, and data-collection judgment

Optimization creates a different latency profile from a feed-forward retargeter: XR acquisition, network transport, state synchronization, simulated rollouts, and command delivery all contribute. Median frequency does not reveal missed deadlines; solve-time tails, warm-start failure, infeasibility, and fallback frequency should be reported. Ten-hertz walking updates may suit deliberate operation but cannot reproduce sudden intent without prediction or buffering.

Model error remains the central safety risk. Wrong contact, friction, payload, or obstacle state can make a simulated constraint-satisfying plan unsafe. Chance constraints, uncertainty-inflated margins, learned residual dynamics, and a verified low-level command set would help. When MPC fails, the system needs a defined hold, slow, or stop command instead of replaying a stale solution.

For data collection, X-OP records raw operator goals plus optimized feasible commands and executed state. This is valuable embodiment-aligned supervision, but it can remove expressive detail and encode optimizer preferences. Logs should include raw XR, synchronized robot state, chosen command, constraint slack, solver status, latency, and the magnitude of operator override. Downstream imitation can then distinguish intent from feasibility correction. X-OP is strongest where interpretability and platform reuse outweigh exact high-rate motion style.

### Operator study and fair comparison with learned retargeters

Technical feasibility does not establish teleoperation usability. X-OP should measure operator training time, perceived responsiveness, correction count, task completion, path/hand error, fatigue, and preference as MPC weights or update rates change. A conservative optimizer may save energy and prevent collision while feeling unresponsive; an aggressive one may track hands closely but force frequent balance corrections. Human-factor results should therefore accompany robot metrics.

A fair learned-retargeter comparison uses the same XR stream, low-level controller, collision model, task, and compute platform. Report tracking, completion, collisions, energy, latency distribution, OOD obstacle/payload behavior, and recovery after deliberate model mismatch. Per-frame IK isolates the value of horizon prediction, while oracle dynamics isolates model error. Cross-morphology experiments should include platforms with genuinely different constraints, not only altered humanoid proportions. These tests would show when online optimization's explicit feasibility justifies its cost and when an amortized learned controller is the better interface.
