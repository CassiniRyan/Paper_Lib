# Perceptive Robotics — Paper Notes

The papers here range from state estimation and mapping to raw-pixel control and active gaze. The discussion therefore follows each paper's own perception problem: what information is missing, why the chosen representation helps, how it enters the robot system, and what kinds of errors it cannot remove. A network is named only when its structure is part of that argument. Cross-listed papers are discussed from the perceptual viewpoint rather than mechanically repeating their locomotion note.

## A Hybrid Autoencoder for Robust Heightmap Generation from Fused Lidar and Depth Data for Humanoid Robot Locomotion

### Problem and system boundary

This paper focuses on the perception layer that sits between raw range sensors and a learned humanoid locomotion policy. A forward depth camera is dense but narrow and material-sensitive; a LiDAR has broader geometric coverage but sparse angular sampling and higher preprocessing cost. Neither observes all terrain beneath the robot at the current instant. The proposed Encoder–Decoder Structure (EDS) fuses both streams, IMU/proprioceptive context, the previous map, and temporal memory into one compact robot-centric height map. A separate PPO controller consumes that map, so perception quality can be inspected independently from motor behavior.

The chosen map is deliberately small: `0.98 m × 0.70 m`, shifted `0.20 m` forward, with `7 cm` cells and 165 height values. Experiments identify roughly 6–8 cm as a useful resolution range. Finer maps increase training cost and sensitivity to noise; coarser maps erase step edges. This is an important design result because “higher resolution” is not automatically better for a policy.

### Sensors, network, inputs, and outputs

An Intel RealSense supplies `160×120` depth and a Livox MID-360 is projected into a `276×40` spherical range image. The LiDAR covers approximately −7° to 52° vertically, about 1.3° horizontally, and updates at 10 Hz. Separate four-layer strided CNN encoders use `3×3` kernels and produce compact 256-D features. They are first pretrained as symmetric convolutional autoencoders on noisy simulated sensor data. In the final EDS, their features, robot state, and prior height-map information feed two stacked GRUs with hidden size 256. Two fully connected decoder layers output the 165-D current map.

Temporal context is explicit rather than implicit in one image. The selected sequence contains 32 updates, or 3.2 seconds at 10 Hz. This allows terrain that was visible ahead of the robot to remain represented after passing under the body. Supervised training uses simulation ground-truth height maps and pixelwise MSE; MSE reportedly outperformed MAE, Huber, and BerHu for the aggregate reconstruction metric. The downstream PPO actor receives proprioception plus the predicted map and outputs joint-position targets for PD control.

### What is demonstrated

Multimodal fusion improves reconstruction accuracy by 7.2% over depth only and 9.9% over LiDAR only. A 3.2-second window reduces drift compared with short histories, with longer windows showing diminishing returns. The fused model maintains average error below about 2 cm on tested terrain, and the customized map supports anticipatory leg lifting before a step. Map-noise tests suggest locomotion remains stable around 2 cm perturbations, while larger errors become practically dangerous.

### Critical assessment and next work

The paper’s strongest contribution is not a novel locomotion policy but a careful, deployable perception contract: sensor fusion, map geometry, temporal horizon, and control sensitivity are evaluated together. The explicit map is easy to visualize and can be reused by different controllers. Its limitation is representational. A single elevation per cell cannot describe overhangs, rails, holes with visible bottoms, or two traversable surfaces at one `(x,y)`. Pixelwise MSE also behaves as a low-pass filter and visibly smooths sharp step edges—the exact features a foot-placement controller cares about.

Edge-aware or asymmetric losses should penalize underestimating discontinuities more than harmless flat-surface noise. Uncertainty and observation age should accompany every cell so the controller can distinguish measured from remembered geometry. Multi-layer height maps or local voxels would cover overhangs, while a smaller safety-critical foot-region head could retain sharp boundaries without making the complete representation expensive. Closed-loop evaluation should report fall probability versus structured map errors—not just average MAE—because two centimeters on a flat patch and two centimeters at a beam edge have radically different consequences.

## AME-2: Agile and Generalized Legged Locomotion via Attention-Based Neural Map Encoding

### Goal and key design

AME-2 asks how a map-based policy can remain agile without overfitting to the terrain layouts used in training. Its answer is an attention-based map encoder coupled to a lightweight uncertainty-aware neural mapping pipeline. Local point features describe possible contacts; global map context and proprioception form the query that decides which locations matter. The policy therefore need not compress every map cell equally and its attention weights can be visualized as prospective footholds.

The method is evaluated on two quite different embodiments—ANYmal-D and the humanoid TRON1—so its main claim is architectural generality rather than one robot-specific gait. A teacher learns with ground-truth maps. A deployable student uses the same AME-2 controller architecture but receives learned online maps and noisy history, combining PPO, action distillation, and alignment between teacher/student map embeddings.

### Mapping and control signal path

Depth point clouds are projected into local grids. A lightweight neural process predicts elevation and uncertainty per grid region, and odometry fuses successive predictions into a global map. Querying this global map produces egocentric height and uncertainty channels. Randomized removal, partial observations, drift, and environments that sometimes expose complete maps teach reuse of old observations under occlusion.

The controller runs at 50 Hz and emits joint PD position targets; hardware joint control runs at 400 Hz. Proprioception contains base linear velocity for the teacher only, base angular velocity, projected gravity, joint positions and velocities, previous actions, and a relative position/heading goal. The student excludes base linear velocity and stacks 20 steps of most state channels. An LSIO temporal encoder turns those histories into a state/dynamics embedding.

AME-2 first builds pointwise local features from map values and positional coordinates. A second MLP plus max pooling produces global context. Proprioception is embedded into a query, while local features act as keys/values in multi-head attention. Attention-weighted local features, global features, and proprioception are concatenated and decoded by an MLP actor. The asymmetric critic uses clean state, ground-truth mapping, and per-link contact state; it uses an MoE value network rather than forcing the actor’s generalization-oriented structure onto value estimation.

### Training evidence

Training begins with a privileged teacher, then the student is initialized through distillation before PPO is enabled fully. Goal position and heading rewards are time masked, letting the robot choose intermediate maneuvers instead of following a fixed path. Terrain curricula span sparse and dense geometries; observation, map, actuator, and dynamics randomization support zero-shot transfer. Policies train at large scale in Isaac Gym, with reported 80,000 teacher and 40,000 student iterations.

The real contribution is the combination of interpretable spatial selection and deployable online mapping. The attention maps consistently emphasize likely future support locations, while learned uncertainty prevents sparse or occluded observations from being treated as certain. Results on both quadruped and humanoid hardware support cross-embodiment usefulness better than a single-platform demonstration would.

### Judgment and future directions

AME-2 occupies a productive middle ground between hand-engineered mapping and raw depth-to-action learning. It preserves a geometric interface, yet jointly trains how the controller reads it. The cost is a substantial stack: depth projection, uncertainty prediction, odometry, global fusion, teacher training, distillation, and PPO. Failure can originate in any layer. Attention is interpretable as *where* the policy looks, not necessarily *why* that point causes a particular joint action.

The most revealing next tests would separate mapping generalization from controller generalization: reuse one map across controllers, feed identical perfect maps to AME-1/AME-2, and evaluate systematic odometry bias rather than random noise. Attention calibration should be tested by masking the highest-weight cells and measuring performance change. Dynamic terrain also remains problematic because temporal fusion assumes persistent geometry. A future map should represent occupancy age and motion, and the actor should expose a confidence/slow-down output when the only available footholds are uncertain.

## APREBot: Active Perception System for Reflexive Evasion Robot

APREBot targets the blind spots and latency of a single exteroceptive sensor during high-speed quadruped obstacle avoidance. It combines omnidirectional LiDAR surveillance with a steerable camera: LiDAR supplies the reflex trigger and coarse direction, while the camera actively turns toward the threat for richer local evidence. A hierarchical controller separates fast evasive reactions from deliberate perception and locomotion. The important idea is not merely sensor fusion, but allocating sensing effort according to risk and reaction time. Sim-to-real tests cover obstacles approaching from different directions. The architecture is attractive for safety layers, though its reflex behavior is narrower than general path planning and depends on reliable timing and calibration between sensors.

### System architecture, information flow, and assessment

The design is best understood as an attention-allocation system rather than a conventional learned navigation policy. A 360-degree LiDAR continuously supplies coarse range observations over the full azimuth. From these observations the first stage detects nearby objects, estimates their relative approach, and ranks them with a threat measure involving separation, approach direction, and time-to-collision. The highest-risk object becomes the active-perception target. A pan/tilt or otherwise steerable RGB-D camera is then directed toward that bearing, where higher-resolution image processing performs detection, tracking, segmentation, and depth-based pose refinement. The controller therefore always retains a fast LiDAR-level threat estimate while opportunistically replacing it with a richer camera-level estimate when the camera has acquired the target.

The input/output boundary is unusually clear. Inputs are the LiDAR point/range stream, RGB-D images, robot state, and target-relative motion estimates; the perception output is a prioritized obstacle state; the behavioral output is an evasive locomotion command passed to the quadruped controller. This is not an end-to-end RGB-to-joint-action network and is not principally an RL contribution. Learned 2-D vision components may appear inside the camera branch, but the paper's core is hierarchical sensor scheduling and reflex logic. That distinction matters: the system remains interpretable and can react before the narrow-field camera finishes slewing.

The highlight is the explicit treatment of sensing latency. Ordinary fusion assumes all sensors are already observing the same object, whereas APREBot asks which object deserves scarce high-resolution sensing now. This is a useful pattern for robots that need omnidirectional safety but cannot carry panoramic high-resolution depth cameras. Its weakest cases follow directly from the architecture: rapid target switching can cause camera thrashing; multiple simultaneous threats are compressed to one focus target; LiDAR association errors contaminate the camera cue; and calibration, timestamping, and actuator slew delay enter the safety loop. The experiments support reflexive evasion, not a complete global-navigation claim. A strong next step would couple multi-hypothesis threat tracking and reachable-set prediction to the gaze scheduler, report end-to-end latency margins, and expose a conservative fallback whenever camera acquisition fails.

## CReF: Cross-modal and Recurrent Fusion for Depth-conditioned Humanoid Locomotion

### Problem and architecture

CReF deliberately avoids an explicit height-map target. A forward `64×48` depth image and current proprioception feed one end-to-end actor; an asymmetric critic alone sees the clean local height field. The aim is to learn only geometry useful for locomotion rather than inheriting a hand-designed map’s blind spots and reconstruction bias.

Proprioception is base angular velocity, projected gravity, motion command, joint offsets from nominal, joint velocities, and previous action. A lightweight LSTM auxiliary head estimates base linear velocity from proprioception plus compressed depth. A CNN tokenizes the current depth image. One proprioceptive token—augmented with estimated velocity—acts as query in multi-head cross-attention over depth tokens. Thus different posture/command states can select different pixels from the same image.

The attended depth feature and proprioception pass through a gated residual fusion block. A GRU then integrates temporal evidence. A highway output gate blends current fused features with recurrent state: ambiguous/risky phases can rely more on memory, while clear current observations bypass it. An MLP head outputs residual joint-position targets `q_target = q_nominal + action`. The critic adds true base linear velocity and privileged robot-centric height samples.

### Terrain-aware contact shaping and timing

CReF’s foothold reward builds local point-cloud buffers near each foot. Command-conditioned filtering looks roughly 0.5 seconds ahead, partitions candidates into `24×10 cm` windows at 4 cm stride, and rejects patches that are rough, tilted, or recessed. At liftoff it freezes candidate support centers; at touchdown it rewards proximity to the nearest valid center. This is directional positive guidance, unlike a binary “do not step outside” penalty.

PPO trains 4,096 Isaac Gym environments. Control runs at 50 Hz while depth updates at 20 Hz. On Unitree H1, a head-mounted RealSense D435i is pitched about 50° downward. Real depth is timestamp-aligned to control. Notably, the policy transfers without synthetic depth corruption, so robustness is attributed mainly to representation/fusion rather than a carefully matched artifact generator.

### Evidence and judgment

Ablations separately remove cross-attention, gated residual fusion, recurrent fusion, and the new foothold reward. Full CReF performs best across steps, gaps, platforms, and OOD settings. On descending stairs the proposed reward halves median placement deviation from roughly 3.0 cm to 1.5 cm versus the comparison reward. Gate analysis shows greater recurrent contribution during flight and risky posture, which supports the intended state-dependent memory interpretation.

CReF’s highlight is not “GRU plus camera,” but a carefully controlled information flow: state queries vision, residual gates regulate modality correction, and another gate regulates temporal memory. The main weakness is reliance on one forward active-depth sensor. Terrain under the body is remembered rather than re-observed, and sunlight, reflective surfaces, vegetation, or transparent obstacles can cause large invalid regions. A recurrent latent can preserve an error as easily as a correct surface.

Future work should add calibrated confidence and an explicit reset/update rule for stale memory. A small downward or rear sensor could cover the under-base blind region without requiring a full mapper. RGB/semantic features could classify non-supportable appearances that depth treats geometrically. Finally, the paper should compare equal-capacity state-space/transformer memory and measure recovery after deliberately injecting a false depth patch; nominal traversal scores do not reveal how quickly memory forgets a dangerous hallucination.

## DCReg: Decoupled Characterization for Efficient Degenerate LiDAR Registration

DCReg addresses LiDAR registration when corridors, tunnels, planes, or limited fields of view make some motion directions weakly observable. It explicitly characterizes degeneracy and decouples well-constrained from poorly constrained components instead of trusting a single ill-conditioned pose solve. This allows robust updates in observable directions while regularizing or obtaining assistance for the rest. The work belongs primarily to localization, but it matters to navigation because planner maps and obstacle positions become unsafe when odometry silently drifts. Its strength is interpretable geometry-aware handling of degeneracy; performance still depends on enough useful structure or auxiliary information eventually becoming available.

### Schur-complement diagnosis and selective preconditioning

DCReg begins from the Hessian of point-cloud registration and asks why standard eigenvalue tests can misdiagnose degeneracy. Rotation and translation are coupled in the full six-DoF system, so a small eigenvalue does not cleanly reveal the physical unconstrained motion. A Schur-complement decomposition eliminates the coupling and produces separate rotational and translational subproblems. Eigenvectors in these clean spaces are then mapped back to explicit physical directions, identifying, for example, translation along a corridor or rotation insufficiently constrained by visible planes.

The same characterization drives mitigation. Instead of damping the entire normal equation—thereby degrading good measurements—DCReg constructs a preconditioner that stabilizes only detected weak directions and solves with preconditioned conjugate gradients. Its interface is therefore geometric rather than learned: input correspondences/Jacobians and residuals yield a conditioned six-DoF pose increment plus interpretable observability diagnostics. A single physically meaningful threshold controls intervention.

This detect–interpret–repair chain is the highlight. The paper reports 20–50% localization improvement and 5–100x speedups over compared degeneracy methods across varied environments, suggesting that principled linear algebra can outperform heavier heuristics. But regularization cannot create missing information: prolonged motion in a truly featureless tunnel still needs IMU, wheel/contact constraints, loop closure, or active sensing. Results also depend on correspondence quality and linearization; dynamic objects or bad associations can look like conditioning trouble. Future integration should expose directional covariance to planners, adapt sensor viewpoint to restore observability, and distinguish geometric degeneracy from outlier-induced Hessian corruption.

## Distillation-PPO A Novel Two-Stage Reinforcement Learning Framework for Humanoid Robot Perceptive Locomotion

### Why distillation alone is insufficient

Distillation-PPO (D-PPO) addresses a common teacher–student failure: action imitation can make a deployable student approach its privileged teacher but never improve beyond it or recover from teacher error. Pure end-to-end PPO permits improvement but is unstable when noisy perception and high-dimensional history must be learned from sparse locomotion rewards. D-PPO first creates a strong teacher and initializes the student from it, then optimizes a weighted sum of action-distillation and PPO objectives. The teacher anchors learning; on-policy return allows the student to depart when its own partial observation demands a different action.

### Inputs, network, and training

The teacher receives accurate local terrain scan dots plus a 50-frame state/action history. Separate encoders compress terrain and history to 32-D latents, which a combined MLP maps to desired positions for every actuated joint. The student inherits the teacher structure and parameters but receives noisy deployment observations. Its history contains base motion, joint state for legs/waist/arms, projected gravity, command, and previous action. Terrain comes from an elevation mapper rather than perfect simulation.

Teacher training is standard PPO in a fully observable setting. Student loss is `0.5 L_distillation + 0.5 L_PPO`; the squared action error regularizes toward the teacher while clipped PPO optimizes student rollouts. Rewards combine command velocity tracking, periodic swing/stance shaping, action smoothness, acceleration, torque, limits, torso orientation/yaw, and stable posture. The symmetric teacher/student construction avoids giving the teacher excessive privileged variables that cannot be inferred reliably on hardware.

Deployment uses Livox MID-360 with Fast-LIO2 for state/mapping and an Orbbec 355L depth camera projected into the map. Cell heights are updated with a one-dimensional Kalman filter whose noise grows with sensor range. Noise injected during student training represents pose and depth uncertainty.

### Evidence, judgment, and next work

The reported learning curves show D-PPO converges more stably than student PPO and retains better control than pure DAgger under noisy terrain. Hardware trials cover stairs, slopes, and platforms. The conceptual contribution is simple but useful: behavioral cloning should be a soft prior during RL, not a hard ceiling.

The main limitation is that action-space MSE assumes the teacher’s action is the correct local target even when teacher and student observations imply different uncertainty. Equal weighting is fixed rather than confidence dependent. A teacher trained on exact scan dots may command an aggressive step that is unsafe under a blurred map. Future work should weight imitation by teacher feasibility and student perceptual confidence, distill distributions or values rather than only mean actions, and ablate history length. Comparing against residual-to-teacher actions would also reveal whether full policy optimization is necessary or whether a smaller corrective policy is safer.

## DORAEMON: A Unified Library for Visual Object Modeling and Representation Learning at Scale

[DORAEMON](https://arxiv.org/abs/2511.04394) is infrastructure for scalable visual representation learning rather than a robot policy. Its YAML-driven PyTorch workflow unifies classification, retrieval, and metric learning, exposing over a thousand pretrained backbones through a timm-compatible interface alongside modular losses, augmentation, distributed training, and ONNX/Hugging Face export. The contribution is reproducible experimentation and deployment across object-recognition tasks. It can supply encoders for robotic perception, but it does not itself solve geometry, temporal association, control, or embodied data shift.

### Unified training interface and supported model families

DORAEMON takes labeled images—category labels for classification, identities for face recognition, or instance relationships for retrieval—and maps them through a task-agnostic preprocessing/augmentation stage, a shared image encoder, and a task-specific head. More than 1,000 timm-compatible pretrained variants are exposed, including ResNet, Swin Transformer, ViT, MAE, and CLIP families. Fine-tuning may freeze selected layers or update the entire backbone; outputs are class probabilities or discriminative embeddings rather than a robot action.

The YAML configuration selects decoding/resizing, RandomCrop, ColorJitter, CutOut, Copy-Paste, MixUp, progressive augmentation, optimizer and layer-specific rates. Classification uses softmax/cross-entropy (with options such as label smoothing or focal loss), face recognition uses angular-margin objectives including ArcFace, CircleLoss, and MagFace, and retrieval uses triplet or cosine-margin learning with Recall@K evaluation. SGD, Adam, SAM, warmup/cosine scheduling, hard-example mining, distributed/fault-tolerant training, automatic metrics, and Grad-CAM are integrated. Export targets include ONNX, TensorRT, TorchScript, PyTorch checkpoints, and HuggingFace-compatible model/processor APIs.

The contribution is infrastructure, not a novel universal backbone. Its strongest value is controlled comparison: changing a backbone or loss need not silently change the data/evaluation pipeline, and recipes reportedly reproduce or exceed references on ImageNet-1K, MS-Celeb-1M, and Stanford Online Products. For this library, relevance to perceptive robotics is indirect—one could train a detector/retrieval encoder consistently, but the paper does not show temporal sensing, depth, calibration, uncertainty, closed-loop control, or sim-to-real locomotion. Robot-focused extensions should add video/RGB-D/point inputs, latency and calibration metrics, continual/domain adaptation, uncertainty, and downstream closed-loop evaluations rather than assuming image-benchmark accuracy predicts safe perception.

## DPL: Depth-only Perceptive Humanoid Locomotion via Realistic Depth Synthesis and Cross-Attention Terrain Reconstruction

### System idea

DPL builds a depth-only perceptive stack around a stable blind locomotion backbone. A vision modulator does not replace proprioceptive walking; it changes joint actions, gait clock, and velocity command when terrain demands it. A separate cross-attention reconstructor converts short depth history plus long proprioceptive history into a structured height representation. The final student is trained with multi-teacher distillation, PPO, and an adversarial motion prior.

### Observation and action contract

The base observation includes command velocity, body angular velocity, joint positions/velocities, periodic left/right gait signals, and previous action. A local height map provides privileged terrain. The policy updates at 100 Hz and returns 20 joint targets plus residual changes to locomotion clock/velocity, giving 22 action values; 1 kHz PD executes the joints. Blind and perceptive actions are combined rather than forcing vision to relearn stabilization.

The terrain reconstructor receives 50 proprioceptive states and five recent depth observations. CNNs compress depth; a history encoder forms a proprioceptive query; cross-attention treats depth features as keys/values. A decoder predicts a rough map, and a depth-conditioned U-Net refines sharp edges/flatness. Ground-truth height maps supervise reconstruction, so DPL is not purely end-to-end despite its depth-only deployment input.

### Realistic depth synthesis

GPU ray casting renders `600×480` depth from calibrated pinhole cameras and includes the articulated robot, which reproduces self-occlusion. Noise magnitude grows with range; lateral uncertainty, incidence effects, boundary cropping, and edge-conditioned missing pixels model structured holes around discontinuities and limbs. This cuts reconstruction MAE by more than 30% relative to simpler synthesis. The real Orbbec 355L runs at 30 Hz; the complete perception-action delay is about 20 ms, faster and less variable than the compared LiDAR/elevation stack.

### Evidence and critical reading

Ablations show the blind backbone prevents conservative freezing on hard terrain, while multi-teacher supervision and gait-command adaptation improve gaps/hurdles. Real trials include stairs, gaps, movable platforms, and disturbances. Compared with a mismatched depth pipeline, stair stumbling falls substantially.

DPL’s strongest design choice is to preserve a competent proprioceptive policy and let perception modulate it. Its weakness is pipeline complexity: simulated depth, transformer reconstruction, U-Net refinement, several terrain experts, distillation, PPO, and AMP all contribute. Exact attribution requires equal-compute ablations. Height reconstruction still inherits 2.5-D limitations, and five visual frames cannot guarantee recovery from persistent glare or unseen under-body geometry. An uncertainty-aware modulator should return to blind-safe behavior when reconstruction is unreliable. Raw-depth and reconstructed-map variants should be evaluated under identical noise, and tactile/contact feedback could correct errors after touchdown.

## Elevation Mapping for Locomotion and Navigation using GPU

This system builds robot-centric elevation maps fast enough for online legged locomotion and planning. Range measurements are projected into a grid whose cells maintain height and uncertainty; GPU parallelism accelerates fusion, ray processing, filtering, and map updates. The output is a compact 2.5-D terrain representation suitable for traversability scoring, footstep planning, or learned policies. The key achievement is practical high-rate mapping on onboard hardware. Compared with learned perception, it is transparent and reusable, but overhangs and multiple surfaces in one cell are difficult to represent, and map quality remains tied to state-estimation and sensor calibration.

### GPU mapping pipeline and traversability output

The system receives registered point clouds plus robot pose and pose uncertainty, transforms samples into a rolling robot-centered grid, and updates per-cell elevation mean/variance. CUDA/CuPy parallelization covers point association, uncertainty-weighted fusion, map shifting, ray/visibility cleanup, and post-processing. Height-drift compensation prevents state-estimator vertical drift from leaving duplicated terrain; an exclusion zone and ray casting remove robot/self and stale overhang artifacts. Plugins generate smoothed/inpainted height, surface normal, traversability, or planar-region layers for consumers.

A learned traversability filter operates on local geometric patches, but the mapper itself is not an end-to-end policy. Its outputs are explicit grid layers used by foothold optimizers, local planners, or RL actors; no joint action or fixed history window is inherent. This boundary is a strength: the same map can support several controllers, be visualized, and carry uncertainty rather than hiding terrain inside a policy latent.

The contribution is primarily systems throughput. Elevation fusion is embarrassingly parallel but traditional CPU implementations struggle with dense modern LiDARs and expensive visibility updates; GPU execution makes richer cleanup/filtering practical at locomotion rates. Yet one height distribution per `(x,y)` cannot represent both floor and ceiling, vertical walls are awkward, and a falsely low variance can be more dangerous than an obvious hole. Calibration/pose error smears obstacles, while inpainting may invent support. Multi-layer or voxel fallbacks, age/confidence channels, dynamic-object rejection, and risk-weighted evaluation of false free space are more valuable next steps than optimizing height RMSE alone.

## Elevator-LIO: Robust LiDAR-Inertial Odometry for Multi-Floor Navigation under Elevator-Induced Non-Inertial Motion

Elevator-LIO targets a failure case of conventional LiDAR-inertial odometry: elevator acceleration violates assumptions used to infer gravity and vertical motion, while the enclosed cabin has repetitive, weak geometry. The system detects or models elevator motion and adjusts inertial/LiDAR fusion so pose remains consistent across floors. This enables continuous multi-floor localization rather than resetting after a lift ride. The novel contribution is treating elevators as a distinct non-inertial regime inside the estimator. It is highly relevant for building-scale service robots, although detection errors and unusual elevators can still challenge the regime logic.

### Mode-dependent error-state filtering

Ordinary LIO interprets accelerometer readings in an inertial world frame. Inside an accelerating cabin, however, apparent vertical force mixes gravity, elevator acceleration, and robot motion while LiDAR sees a small reflective box. Elevator-LIO factorizes state into robot motion relative to the cabin and the cabin's own vertical motion, then embeds those variables in a mode-dependent iterated error-state Kalman filter. In normal indoor mode it reduces to conventional LIO; in elevator mode it propagates and constrains elevator-related states without falsely assigning cabin acceleration to robot pose.

An event manager detects entry, motion, stops, and exit from LiDAR range statistics and estimated state. At stops, event-triggered zero-velocity and zero-acceleration updates suppress accumulated vertical drift. Adaptive voxel downsampling maintains a stable number of useful points as the sensor transitions between a confined car and a large lobby. Inputs remain time-aligned LiDAR scans and IMU measurements; outputs are continuous pose, velocity, biases, gravity/cabin-motion estimates, and mode, rather than a learned navigation action.

Evaluation spans 20 real sequences and 79 rides with pedestrians, mirrors, large lobbies, and long vertical travel; 17 sequences finish below 1-cm height error, while standard-indoor tests remain competitive. The key insight is not treating elevator corruption as generic noise but changing the state model when the reference frame itself accelerates. The main fragility is mode recognition: a false transition applies the wrong process/measurement constraints, and unusual lifts, robot motion within the car, or prolonged door occlusion may violate event assumptions. Joint semantic/elevator detection, multiple-model probabilistic switching, explicit mode confidence, and building-map floor constraints would make the estimator safer for unattended deployment.

## FAST-LIO: A Fast, Robust LiDAR-inertial Odometry Package by Tightly-Coupled Iterated Kalman Filter

FAST-LIO provides low-latency, high-rate odometry by tightly coupling raw LiDAR points with IMU measurements in an iterated error-state Kalman filter. It directly registers points to the maintained map and jointly updates pose, velocity, biases, and other states, avoiding a slow feature-extraction front end. The second-generation implementation adds an efficient incremental spatial data structure. Its achievement is robust real-time estimation across several LiDAR types and aggressive motion, making it a common localization backbone. It supplies metric state rather than navigation decisions; severe geometric degeneracy, bad synchronization, or poorly calibrated sensors can still cause failure.

### Iterated filter, measurement scaling, and FAST-LIO2 map

IMU measurements propagate a nominal state containing rotation, position, velocity, gravity, and inertial biases (with LiDAR–IMU extrinsics where estimated). Deskewed LiDAR points are associated with local map planes; point-to-plane residuals enter an iterated error-state Kalman update, repeatedly relinearized on the manifold. The original FAST-LIO derives a Kalman-gain form whose dominant cost depends on small state dimension rather than thousands of point measurements, enabling more than 1,200 effective points and a full iterative update in under 25 ms on reported quadrotor hardware.

FAST-LIO2 removes handcrafted edge/plane feature extraction and registers raw points directly. That improves compatibility with nonrepetitive solid-state and spinning sensors and retains subtle geometry. Its ikd-Tree supports incremental insertion, deletion, downsampling, and rebalancing, allowing the local map to evolve without periodic reconstruction; reported operation reaches up to 100 Hz and survives rotations near 1,000 degrees/s in tested cases. Inputs are timestamped IMU and LiDAR packets, and outputs are odometric state/covariance plus an incrementally updated point map—not semantic perception, route choice, or control.

Tight coupling makes every LiDAR residual inform inertial state and allows IMU dynamics to carry short geometric gaps. The compact update and data structure explain the package's broad reuse better than simply calling it “fast.” Nevertheless, it is odometry without inherent loop closure, and observability still collapses in long planar/corridor motion. Timing/extrinsic errors, vibration, moving points, and incorrect planar associations can cause confident drift. Online temporal calibration, degeneracy-aware covariance, dynamic filtering, loop closure, and explicit health signals to the planner are essential around this excellent local estimator.

## Gait-Adaptive Perceptive Humanoid Locomotion with Real-Time Under-Base Terrain Reconstruction

### Under-base sensing and gait as an action

This paper places a downward RealSense D435i under the base, directly observing the support region rather than looking only ahead. Each single 60 Hz depth frame becomes a point cloud and raw ego-height map. A lightweight U-Net fills holes caused by legs/body and produces a dense 425-value local map in about 11 ms, allowing perception and policy to run together at 50 Hz without odometry or temporal map fusion.

The controller does not accept a fixed gait clock. Its 32-D output contains 31 joint targets plus one scalar gait-frequency action. That scalar, clipped to `[0.7,1.3] Hz` and low-pass/rate filtered, advances the global left/right phase. Thus perception, posture, and step timing are optimized jointly: the robot can slow on stairs/gaps and accelerate on flat ground.

### Successive teacher–student design

Proprioception contains `(vx,vy,yaw-rate)` commands, body angular velocity, projected gravity, joints, previous action, and gait signals. The student encodes a history of noisy proprioception plus reconstructed map; the teacher uses clean privileged state and ground-truth map. Teacher/student share the policy head and critic. All modules are ELU MLPs except the U-Net mapper.

Training begins with all 4,096 environments controlled by the teacher. The student encoder learns to match teacher latents but cannot change rollout distribution. After 4,000 iterations a gate gradually assigns more environments to the student; teacher and student PPO trajectories update the shared head while latent reconstruction and mirror losses train the student. This avoids conflicting poor student rollouts at initialization, a weakness of concurrent teacher–student training.

The U-Net has height and edge heads. Height uses L1; the training-only edge head uses BCE plus Dice to preserve discontinuities. It trains from 10,000 noisy simulated frames. Rewards cover velocity, posture, contacts, foot placement/flatness, joint/action smoothness, and gait limits.

### Evidence and judgment

The Oli humanoid shows omnidirectional stair/gap walking and automatically changes gait frequency with speed and geometry. Ablations support successive rollout transfer, explicit gait action, and edge supervision. The strongest idea is tight coupling of *what is under the feet* with *when to step*.

The downward camera has excellent local relevance but poor look-ahead; high speed may reveal a discontinuity too late. A single frame avoids stale memory but must hallucinate self-occluded cells through learned spatial priors. The scalar global frequency cannot independently delay one foot after slip. Future work should combine a low-rate forward view with the under-base map, predict uncertainty/edges, and allow phase residuals per leg or event-driven contact transitions. Map confidence should limit gait speed, and moving support experiments should test whether single-frame reconstruction truly outperforms temporal memory when geometry changes.

## Gallant Voxel Grid-based Humanoid Locomotion and Local-navigation across 3D Constrained Terrains

Gallant uses a local 3-D voxel representation to let a humanoid negotiate constrained terrain where a height map is insufficient, including obstacles affecting the torso and limbs. The learned policy consumes proprioception, goal information, and voxelized surroundings and outputs whole-body actions or joint targets. Training randomizes terrain and obstacles in simulation. Its central contribution is unifying local navigation, collision avoidance, and stable locomotion in a compact volumetric policy rather than planning for the base and body separately. Voxel resolution and sensing range bound the detail and horizon, so a global planner is still useful for large environments.

### Voxel encoder, observation history, and control rates

Gallant voxelizes LiDAR into a torso-centered 32 x 32 x 40 binary grid at 5-cm resolution. This preserves floors, ceilings, lateral clutter, platforms, and drops without the cost of a general 3-D network. The actor additionally sees a relative target/time command and six-frame histories of base angular velocity, projected gravity, joint positions/velocities, and previous actions. Its critic receives privileged base linear velocity and a height map. Non-voxel signals pass through a two-layer, 256-D Mish/LayerNorm MLP; a three-layer 2-D CNN treats the 40 vertical bins as channels and emits a 64-D geometric feature. Fusion produces a 256-D latent, followed by 29 joint targets for the G1 actor and a scalar value for the critic.

Treating height as channels is the architectural highlight: 2-D kernels efficiently model horizontal layout while channel mixing captures vertical obstruction. PPO training spans 1,024 x 8 environments over eight curricula. The raycaster includes the robot's moving links, self-scans, and occlusion holes, plus randomized LiDAR mounting, hit noise, 100–200 ms latency, and 2% missing voxels. On hardware, two Hesai JT128 sensors approximate spherical coverage; voxel updates run at 10 Hz, policy at 50 Hz, and PD at 500 Hz. A separate head Mid-360 with Fast-LIO2 supplies the local target at 25 Hz.

The result is one local policy that crouches, clears doors, climbs stairs/platforms, and crosses clutter where height maps fail. Ablations favor the height-channel CNN over heavier 3-D/sparse alternatives and show that simulated self-occlusion/latency matters for transfer. Still, binary cells omit confidence, surface normal, age, support semantics, and velocity; 5 cm is coarse for footholds, while finer grids reduce field of view. Ten-Hz scans are weak around dynamic obstacles. Multiresolution, probabilistic, time-aware voxels plus a global planner and independent safety fallback are the clearest extensions.

## GaussGym An open-source real-to-sim framework for learning locomotion from pixels

### A simulator/data contribution

GaussGym is primarily infrastructure for pixel-based control. It converts smartphone captures, calibrated scans, and even generated video into paired visual and collision worlds. VGGT estimates camera calibration, dense points, and normals; NKSR produces a collision mesh; the point cloud initializes 3-D Gaussian splats for rendering. Separating photorealistic appearance from physics geometry makes rasterization fast enough for vectorized RL, but creates an alignment problem that must be monitored.

The open dataset contains roughly 2,500 scenes. On one RTX 4090, the system reports about 100,000 simulator steps/s across 4,096 robots and 128 unique scenes at `640×480`, with 50 Hz control and 10 Hz camera updates. Gaussian rasterization produces RGB and depth together. Camera-rate rendering is decoupled from physics/control, and motion blur is simulated by alpha-blending renders offset along camera velocity.

### Example visual locomotion policy

The actor observes RGB, base angular velocity, projected gravity, joint positions/velocities, and swing phase. DINOv2 embeds the image. An LSTM fuses visual and proprioceptive streams over time. One auxiliary head reshapes the latent into a coarse 3-D volume and uses transposed 3-D convolutions to predict occupancy/terrain, forcing geometric information into the shared representation. A second LSTM outputs a Gaussian distribution over joint-position offsets. An asymmetric critic permits privileged simulation state while the actor remains pixel based.

Policies are trained end to end rather than via teacher distillation. A1 and Booster T1 learn stair/slope traversal; an A1 policy transfers zero-shot in a proof-of-concept hardware test. In a semantic navigation experiment, RGB avoids a yellow penalty patch that is geometrically indistinguishable to depth, demonstrating the reason to retain appearance.

### Judgment and research opportunities

GaussGym’s important result is scale: photorealistic pixels can be rendered inside the massive-parallel workflow locomotion RL expects. The auxiliary voxel head and DINO features show how to use those pixels, but the supplied policy is not the paper’s only contribution and is not yet a universal visual controller.

The collision mesh and Gaussian appearance can disagree near thin objects, reflective surfaces, and generated scenes. A policy may learn a visually plausible affordance with incorrect physics. The paper also notes performance degradation on unseen staircases and continued sensitivity to latency/egocentric coverage. Scene diversity is not the same as physical diversity if many reconstructions share similar traversability.

Future tooling should quantify render–collision registration and reject unsafe scenes automatically. Dynamic objects, changing lighting, and online real-camera adaptation would broaden use. Language/semantic labels attached to splats could produce tasks richer than color avoidance. Most importantly, benchmark policies should control for visual backbone and scene count to separate simulator realism from representation quality.

## GeoLoco: Leveraging 3D Geometric Priors from Visual Foundation Model for Robust RGB-Only Humanoid Locomotion

### Turning RGB into a geometric latent

GeoLoco argues that monocular RGB should not be treated as raw texture. A frozen metric Depth-Anything-V2 ViT-S already contains scale-aware surface structure; the controller consumes its intermediate spatial tokens rather than scalar predicted depth. This preserves appearance/semantic information for future higher-level tasks while providing geometric cues for stairs, slopes, gaps, and blocks.

### Exact visual and control path

The input image is `112×112`. The 12-block, 24.8M-parameter ViT-S yields 64 `8×8` patch tokens of width 384. Tokens from blocks 4, 8, and 12 capture multiple scales. Channels are divided into 32 groups and averaged, compressing each location 12×; concatenating three layers produces a `96×8×8` descriptor without trainable projection.

Two most recent visual updates form 128 spatial-temporal tokens with learned frame embeddings. Instantaneous 96-D proprioception is projected to one query; visual tokens become keys/values for eight-head cross-attention at width 64. The final policy vector concatenates current state, a five-step proprioceptive history, and attention context into 288 dimensions. PPO outputs 29 joint-position commands. An asymmetric critic sees 371 privileged state values.

VFM inference runs asynchronously at 10 Hz with zero-order-held features; balance/control runs at 50 Hz. Training injects 0–100 ms visual delay, lighting/exposure/color changes, textures, camera shifts, and 15% motion blur. Auxiliary heads estimate base velocity from history and a 99-cell forward height map from fused features, grounding the latent in physical geometry; both heads disappear at deployment.

### Evidence and interpretation

GeoLoco reaches 86.4% success at its final curriculum versus 60.4% for a scratch CNN and 66.1% for DINOv2 tokens. Removing cross-attention, history, or terrain reconstruction causes large drops. On real G1, it succeeds 80% on 0.23 m stairs and 70% on a 0.25 m gap across ten trials, outperforming blind/CNN RGB baselines. Explicit depth/height maps still lead on the hardest terrain, honestly exposing monocular limits.

The key insight is that the *kind of pretraining* matters: a geometric VFM is more useful than a semantic VFM for low-level contact. Proprioceptive-query attention then makes vision state dependent rather than a fixed image embedding.

### Limitations and next work

Metric monocular features are not guaranteed metric under unusual cameras, lighting, materials, or scale. The real policy uses an RTX 4090 offboard, so “RGB-only sensor” does not mean lightweight computation. At 10 Hz it can miss rapidly changing terrain, and attention maps are not calibrated safety estimates.

A distilled onboard encoder, visual uncertainty, and camera-intrinsics conditioning are natural next steps. RGB should be combined with tactile/foot contact after touchdown, not asked to replace every geometric sensor. Tests with transparent, reflective, textureless, moving, and adversarially scaled obstacles would better characterize the foundation prior. Finally, semantic advantages should be demonstrated in tasks where appearance changes traversability—not only geometric locomotion.

## Hier-SLAM: Scaling-up Semantics in SLAM with a Hierarchically Categorical Gaussian Splatting

Hier-SLAM adds scalable semantics to dense Gaussian-splatting SLAM. Rather than attach one flat, fixed class label to every primitive, it organizes categories hierarchically so a map can represent coarse structures, object groups, and fine classes without an exploding semantic state. Camera tracking and Gaussian-map optimization share the geometric/appearance representation, while semantic observations update categorical features. The result supports photorealistic reconstruction and queryable semantic maps for planning. The hierarchy is the key novelty and improves scalability, though the system inherits dependencies on visual tracking, segmentation quality, and compute-heavy dense mapping.

### Hierarchical encoding and joint SLAM optimization

Hier-SLAM accepts an RGB-D stream and incrementally maintains camera poses plus a global 3-D Gaussian map. Each Gaussian stores center, radius/covariance, opacity, color, and a semantic embedding. Incoming poses are initialized with a constant-velocity model and refined against the fixed map using rendered color and depth losses over pixels with sufficient silhouette visibility. Mapping then fixes the camera poses and optimizes geometry, appearance, and semantic parameters jointly. Consequently its primary outputs are a metrically tracked trajectory and renderable color/depth/semantic maps—not navigation actions.

The semantic novelty is a class tree rather than one independent vector dimension per leaf class. Before SLAM, GPT-4o-mini groups the supplied class vocabulary from leaves toward progressively coarser parents; a validator/critic loop checks omitted labels, and tree construction stops when the coarse level is sufficiently compact. This LLM is an offline taxonomy builder, not an online navigator. Every Gaussian receives embeddings along its root-to-leaf path. An inter-level loss learns predictions at individual resolutions, while a cross-level loss flattens the combined hierarchy back to the leaf target, keeping coarse and fine meanings mutually consistent. Training schedules the cross-level term after initial inter-level fitting.

This representation is most convincing as a memory-scaling contribution: the reported system handles over 500 semantic classes and can render coarse-to-fine interpretations while reducing semantic storage and optimization time. It also avoids carrying a large vision-language feature at every primitive. The tradeoff is that open-world understanding is only as good as the predefined labels and offline tree. Ambiguous objects may reasonably have multiple parents, whereas the chosen taxonomy imposes one path; an LLM-generated grouping can also encode arbitrary ontology errors. RGB-D blur and noisy depth still degrade tracking, and segmentation mistakes can be baked into the map. Useful extensions are online insertion/reparenting of categories, uncertainty over hierarchical labels, loop closure and long-term map maintenance, and task-conditioned rendering that measures whether the hierarchy actually improves robot decisions rather than only 2-D mIoU.

## Hiking in the Wild A Scalable Perceptive Parkour Framework for Humanoids

### Direct high-bandwidth perception

Hiking in the Wild targets a specific bottleneck in perceptive humanoid control: LiDAR/elevation-map stacks are accurate but can be slow and sensitive to localization drift, while many raw-depth policies process vision too slowly for running and jumping. The paper trains an end-to-end depth-and-proprioception policy and deploys it without an external pose estimator. A factory Intel RealSense D435i supplies depth at up to 60 Hz on a Unitree G1, and the policy maps this stream directly to 29 joint-position targets tracked by PD control.

### Inputs, temporal encoding, and RL objective

The actor observes commanded motion, base angular velocity, projected gravity, 29 joint positions and velocities, its previous 29-dimensional action, and depth. The critic is asymmetric and additionally sees noise-free observations and true base linear velocity. The depth encoder is designed as a mixture of experts so visual capacity can scale without evaluating every expert densely. More importantly, depth history is *strided*: rather than encoding only adjacent frames, the buffer selects frames separated by a temporal stride. This gives a longer look-back horizon at approximately fixed compute and lets the controller infer approach speed and recover through transient missing frames.

Training uses PPO with task, regularization, safety, and adversarial-motion-prior reward groups. Task terms drive velocity/goal tracking; regularizers limit torque, action change, joint motion, and impacts; safety terms discourage collision and bad edge interaction; AMP supplies a broad natural-motion preference without dictating one trajectory. The final action is position control, not an abstract footstep or velocity, so the policy learns perception, contact timing, and whole-body stabilization jointly.

### Realistic depth and scalable terrain generation

The simulated camera models quantization, precision decay with range, Gaussian noise, missing pixels/artifacts, and real sensor filtering. The paper also derives a spatial collision grid from terrain geometry and emphasizes edge geometry during training. Its Flat Patch Sampling is a deceptively important contribution: naïvely sampling navigation goals on procedural terrain can reward circling or other “reward hacking.” Sampling reachable flat patches and generating command directions toward them makes progress reward correspond to genuine terrain traversal.

The resulting system traverses stairs, slopes, platforms, ramps, grass, and discrete gaps and reports speeds up to roughly 2 m/s. The notable achievement is less a single stunt than a reusable pipeline whose raw-depth loop remains fast enough for full-size humanoid parkour.

### Judgment and future directions

The strongest idea is matching representation to control bandwidth. Avoiding global mapping removes accumulated pose drift and reduces engineering, while strided history gives the local controller enough temporal context to act proactively. The MoE also offers a plausible route to scaling terrain diversity without a dense network of equal size.

The same locality is its limitation. A forward camera cannot see behind the robot, underneath the feet, or far around corners; raw depth has no persistent world memory. Transparent/reflective surfaces, sunlight, rain, and vegetation can break active stereo in ways a simulation noise model may not cover. AMP can improve appearance without certifying contact forces. Next work should add uncertainty-aware temporal memory, fuse a low-rate global route with the high-rate local policy, report expert utilization/collapse in the MoE, and test sensor dropout and outdoor lighting as explicit held-out conditions rather than random perturbations.

## Holistic Fusion: Task- and Setup-Agnostic Robot Localization and State Estimation with Factor Graphs

Holistic Fusion builds a configurable factor-graph estimator that can combine IMU, joint kinematics, contact, cameras, LiDAR, GNSS, or other measurements without redesigning an estimator for every robot and task. Each source contributes factors to a shared state trajectory, and robust optimization resolves their complementary constraints. The novelty is a task/setup-agnostic software and modeling framework rather than a new navigation policy. It improves resilience to individual sensor failures and supplies globally consistent state to downstream mapping and control. Accuracy still depends on valid noise models, calibration, synchronization, and correct contact assumptions.

### Joint local/global state and dynamic context variables

Holistic Fusion formulates both robot state and reference-frame relationships as variables in one factor graph. It accepts an arbitrary number of absolute measurements (for example GNSS/global pose), local relative-motion estimates, and landmark observations expressed in different frames. Instead of demanding pre-alignment, it estimates frame transforms and other “context” variables jointly and lets them evolve as random walks when they may drift. This covers sensor extrinsics, map/world alignment, landmarks, or other setup-specific quantities without changing the conceptual estimator.

The implementation gives special attention to the tension between global correction and control-quality local motion. Global observations suppress long-term drift, but an abrupt graph correction can destabilize a controller. HF produces a low-latency, smooth local belief while maintaining globally accurate localization, propagating results at IMU rate on ordinary robot computers. Five real scenarios on three platforms demonstrate different modality/configuration combinations; there is no neural network or RL policy at the core, and output is a state distribution/trajectory for downstream mapping and control.

The strongest contribution is usability and a common abstraction: adding a measurement type means defining its factor rather than rebuilding a bespoke fusion pipeline. Modeling context in the graph also makes automatic frame alignment part of estimation rather than fragile preprocessing. Generality does not remove observability, however. Unconstrained frame transforms, correlated front-end errors treated as independent, timestamp offsets, contact-model failures, and incorrectly chosen random-walk noise can still yield consistent-looking but wrong estimates. Online observability checks, automated noise learning, robust switchable factors, and explicit compute/memory behavior over lifelong missions are important extensions.

## LadderMan: Learning Humanoid Perceptive Ladder Climbing

### Why ladder climbing is not ordinary locomotion

LadderMan addresses a contact-rich task in which hands are load-bearing locomotion effectors rather than balance accessories. The robot must discover and maintain four-limb contact on sparse rungs, keep its torso clear of the ladder, move upward or downward, and retain enough stability to manipulate while supported. Standard lower-body locomotion rewards tend to ignore the hands; strict full-body motion imitation, by contrast, breaks when ladder spacing and inclination differ from the reference.

### Hybrid motion tracking and expert construction

The method begins with a single captured climbing reference but tracks upper and lower body asymmetrically. Upper-body keypoints are followed more closely to preserve reaching style, while lower-body tracking is relaxed so feet can adapt to the actual rung geometry. Ladder-centric rewards explicitly supervise hand/foot contact, vertical progress, body alignment, and stability. Experts are trained across parameterized ladder inclination, rung spacing, width, and dimensions; contact loss persisting for more than 30 frames terminates an episode.

The paper also decomposes upper- and lower-body learning during analysis: each agent sees whole-body proprioception and outputs its portion of the joint target vector, with upper-body motion tracking and lower-body contact/stabilization rewards. This exposes an important fact—successful climbing is not just replay of a reference but negotiation between reach and support.

### Visual policy, network I/O, and training

The final unified policy receives whole-body proprioception, depth observations, and a binary upward/downward command and outputs 29 joint-position targets for a Unitree G1. Expert policies additionally receive the reference phase and privileged ladder/contact state. Rung-focused masking suppresses irrelevant depth structure so the encoder emphasizes the thin horizontal supports that actually determine contacts. Experts are distilled with a hybrid objective: a DAgger-style KL/imitation term keeps useful behavior, while PPO continues optimizing the real climbing reward. The imitation weight is annealed, allowing the student to move from safe initialization toward adaptation instead of being permanently bounded by imperfect expert actions.

Deployment uses the standard head-mounted RealSense D435i and onboard Jetson Orin for the 50 Hz control policy. For cleaner real depth, the reported system applies Fast FoundationStereo to the camera’s stereo grayscale images on an external RTX 4090, with the infrared emitter and auto-exposure disabled. That distinction is important: motor inference is onboard, but the demonstrated perception pipeline is not entirely self-contained.

### What it achieves and why it matters

The system climbs unseen ladders with varied spacing/inclination and supports stable on-ladder teleoperation tasks. The strongest contribution is hybrid tracking: it preserves a humanlike coordination template while explicitly granting the policy freedom where contact geometry demands it. Rung-focused perception and RL-augmented distillation solve two concrete failure modes—background distraction and compounding imitation errors.

### Limitations and next work

External GPU stereo processing weakens the “no hardware modification” autonomy story. Thin-rung depth remains vulnerable to glare, texturelessness, and self-occlusion; the binary direction command does not encode a task-level route; and the experiments do not constitute a guarantee against catastrophic missed contacts. Future versions should run an optimized stereo/rung detector onboard, predict contact probability and load margin, and include a verified retreat/freeze behavior. Irregular ladders, missing rungs, flexible rails, transition onto the top platform, and manipulation forces should be evaluated as held-out structural changes, not just parameter randomization.

## Learning Autonomous and Safe Quadruped Traversal of Complex Terrains Using Multi-Layer Elevation Maps

The paper represents terrain with multiple elevation layers so bridges, steps, overhangs, and stacked surfaces are not collapsed into a single height. From these maps it estimates traversability and plans routes, while a learned locomotion policy follows commands over rough ground. The main novelty is bringing a richer yet planning-friendly terrain model into an autonomous legged stack. This improves safety over ordinary 2.5-D maps in vertically complex scenes. It still discretizes geometry and depends on accurate localization and map fusion; unseen dynamic obstacles require an additional reactive layer.

### Multi-layer representation and hierarchical policies

Each horizontal cell holds several non-intersecting vertical surface intervals, representing floor and tabletop, bridge underside and deck, or several stacked obstacles without paying for a full voxel volume. Convolutional encoders process regular layers, while a Transformer handles variable-length surface elements. A terrain compressor is trained with L1 reconstruction loss (roughly 3.0-cm train and 4.9-cm test MAE), and its features condition a local navigation policy.

Below navigation, several terrain-specialist locomotion policies are trained and distilled into one generalist. The deployed locomotion actor observes desired body velocity, joint positions/velocities, proprioception, its previous 12-D action, and the multi-layer map, and emits 12 joint-position targets. During distillation the student sees extra clutter not shown to every expert, increasing robustness without destabilizing specialist training. Navigation fuses goal and map features to command this generalist. PPO rewards goal progress, collision avoidance, maneuverability, and stable execution, with terrain curricula and symmetry augmentation.

The representation is the durable contribution: it retains grid/CNN efficiency while expressing multiple surfaces at identical `(x,y)`, enabling routes beneath overhangs and across constrained platforms that one-height maps cannot describe. But “safe” here is empirical reward shaping and randomized evaluation, not a verified shield. Mean height error is also a weak metric at a clearance boundary: a small average error can create false free space exactly where a head or flank collides. Fixed layer count/resolution loses thin structure and has no natural dynamic-obstacle state. Per-layer confidence and timestamps, swept-body collision filtering, false-free-space metrics, and equal-memory comparisons against sparse voxels/SDFs would strengthen the safety and representation claims.

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

## Learning Perceptive Humanoid Locomotion over Challenging Terrain

### A world model for corrupted terrain observations

This work, named the Humanoid Perception Controller (HPC), focuses on a failure mode often hidden by simulation: a terrain observation can be wrong, not just noisy in a frame-independent way. If a height map overestimates a stair or shifts over time, a feed-forward actor repeatedly commits to the wrong foothold. HPC trains a temporal variational world model to denoise proprioception and terrain history while a student imitates a privileged oracle.

### Oracle and student architectures

Stage one trains an oracle actor–critic with PPO on clean simulator state. A terrain encoder maps the noise-free local height map into a spatial feature, which is fused with robot state and command. Actor and critic use separate LSTM branches followed by MLP heads; the actor returns the mean of a joint-target action distribution, while the critic sees privileged state. Rewards deliberately avoid reference trajectories and instead combine velocity tracking, posture, contact, energy, smoothness, and safety constraints.

Stage two rolls out the *student*, labels its visited states with oracle actions, and aggregates them DAgger-style. The student world model receives a sequence of corrupted proprioception and terrain features. An LSTM produces a temporal hidden state; separate MLP heads output Gaussian mean and variance, and a sampled latent drives both a decoder that reconstructs privileged state and the locomotion policy. Training combines action MSE, reconstruction likelihood, and a KL penalty toward the latent prior. At deployment the decoder is discarded and the policy uses the posterior mean, leaving only terrain encoder, recurrent encoder, and actor.

The corruption curriculum includes height displacement and compounding state-estimation errors rather than only white pixel noise. This is an important distinction: temporal recurrence can average independent noise, but systematic drift requires combining predicted dynamics, contact feedback, and changing observations. The authors report 72.9% retained performance under a prolonged-noise condition and demonstrate conservative stable stair ascent where less temporal baselines compound placement error.

### Contribution and relationship to other approaches

HPC combines three familiar tools—privileged RL, variational sequence modeling, and DAgger—but assigns each a clear role. The oracle establishes what action clean geometry permits; reconstruction forces the latent to retain physical state rather than only mimic an action; student rollouts expose it to its own error distribution. Compared with direct depth-to-action distillation, it explicitly models uncertainty and history. Compared with explicit filtering, its state is optimized for control.

### Critical assessment and future work

The VAE’s Gaussian latent and KL term regularize information but do not automatically produce calibrated terrain uncertainty. Using the posterior mean at deployment discards sampled uncertainty exactly when caution might be valuable. The oracle itself is not optimal or risk-certified, and DAgger transfers its biases. Local height maps also exclude overhangs and semantics.

A strong next step is to propagate posterior variance into action speed or a safety supervisor, evaluate calibration against known map error, and separate epistemic unfamiliarity from sensor noise. Longer occlusion tests would reveal when recurrence becomes hallucination. Adding a consistency loss between predicted contacts and subsequent proprioception could make the world model correct terrain belief from physical interaction—the human-like mechanism that motivates the paper.

## Learning robust perceptive locomotion for quadrupedal robots in the wild

### Learning a belief rather than trusting the map

This influential system addresses the mismatch between impressive simulated perceptive locomotion and unreliable field sensing. Snow, grass, puddles, glare, fog, textureless surfaces, and occlusion can make an elevation map missing, biased, or physically misleading. A teacher first learns with perfect terrain and dynamics information. A student then imitates the teacher using only deployable proprioception and a robot-centric elevation map, while a recurrent belief encoder infers missing privileged state from temporal evidence.

### Actor structure and sensor interface

The policy accepts desired base velocity, IMU/joint signals, previous action, and a local elevation grid. It outputs a target position for every joint, followed by PD control. The map is deliberately an abstraction layer: stereo cameras or LiDAR can populate the same grid without changing the controller. The recurrent student separates an explicit terrain encoding from a latent belief. When vision is trustworthy, the map enables anticipatory foot lift and placement; when corrupted, history and contact response permit behavior closer to a robust blind controller.

Training uses privileged RL for the teacher and imitation for the student, with aggressive dynamics, observation, delay, map-shift, dropout, and terrain randomization. A curriculum adapts difficulty according to success. Crucially, the student sees combinations of noise, bias, missing regions, and delayed measurements, not only mild Gaussian error. It learns that exteroception is evidence rather than truth.

### Field evidence and contribution

The paper tests ANYmal with LiDAR- and active-stereo-derived maps on stairs, blocks, snow, vegetation, mud, water, and mountain trails. In the headline hike, the robot reached a summit in 31 minutes and completed the route in 78 minutes, close to the cited 76-minute human estimate, with interventions for a detached shoe and battery exchange. Controlled comparisons show why vision matters: a blind policy cannot anticipate steps near its physical limit.

The novelty is not elevation mapping itself but learned arbitration between unreliable external perception and dependable-but-reactive body sensing. Sensor interchangeability and long outdoor validation made this a foundation for later teacher–student locomotion systems.

### Limits and next steps

The grid is 2.5-D and geometric: it cannot represent overhangs, moving objects, or whether a visually high surface is soft grass or load-bearing rock. Robustness is empirical; the belief has no calibrated confidence and could be confidently wrong outside the corruption model.

A modern extension should fuse semantics and force feedback, predict map uncertainty, and couple uncertainty to speed or foothold margin. Held-out failure suites—glare, transparent obstacles, vegetation, and total sensor dropout—would measure extrapolation. A sparse 3-D map plus recurrent contact correction could retain the strong abstraction while handling confined geometry.

## Learning Vision-Based Bipedal Locomotion for Challenging Terrain

### Two learned modules instead of global mapping

This Cassie work presents an early real demonstration of vision-based biped locomotion over genuinely challenging geometry. It deliberately avoids global pose estimation. A PPO locomotion policy consumes proprioception, user commands, and a robot-local height map; a supervised predictor reconstructs that map from a history of egocentric depth images and robot states. The local interface keeps the controller independent of camera coordinates and avoids odometry drift during impact-heavy motion.

### Controller and predictor

The control policy outputs PD setpoints for Cassie’s actuators and is first trained against perfect simulated local height. Terrain, command, dynamics, latency, and observation randomization yield behaviors that adapt step height and timing. Once trained, the policy generates rollouts for the perception dataset. A convolutional visual encoder and temporal fusion of depth with robot-state history estimate the same local grid expected by the controller. History is essential because a single head view does not cover the ground under both feet and body motion changes the camera frame.

Simulation pairs rendered depth and randomized state histories with ground-truth maps. Image artifacts, camera pose, intrinsic/extrinsic error, latency, and depth noise are randomized before zero-shot transfer. Split training is practical: expensive images are not rendered throughout PPO, and control can be debugged separately from perception.

### Capability and contribution

The camera-equipped Cassie crosses random blocks, stairs, and a 0.5 m step—around 60% of leg length—at commanded speeds up to 1 m/s. The contribution is system integration: a learned, robot-relative visual terrain estimate drives a learned biped controller without explicit world odometry or real-data fine-tuning. Compared with direct pixel-to-action training, the predicted map is inspectable and forms a stable contract between vision and control.

### Critical assessment

The separation creates a hard bottleneck. Height loss weights cells similarly even though centimeter error at a landing edge is far more dangerous than background error. A single-valued surface cannot express an overhang or support quality, and the locomotion policy is trained first on true maps rather than its predictor’s structured errors.

Future work should use task-weighted, uncertainty-aware map loss, supply confidence to the controller, and jointly fine-tune the modules while retaining the interpretable grid. Multi-layer geometry and contact-driven correction would extend it beyond laboratory blocks. Long-run camera occlusion and calibration-drift tests would clarify how much robustness comes from history.

## Legged Locomotion in Challenging Terrains using Egocentric Vision

### End-to-end vision on a small quadruped

This CoRL work demonstrates a compact quadruped traversing stairs, curbs, stepping stones, gaps, rocks, and outdoor terrain using one front-facing depth camera. The egocentric camera loses sight of terrain once it passes below the body, so hind-foot placement must depend on memory; onboard hardware also cannot afford a heavy mapping/planning stack.

### Cheap privileged training and visual distillation

Training has two phases. An oracle is optimized with PPO using inexpensive simulated depth-like terrain scans rather than rendered images. This permits massive parallel control learning and discovery of specialized gaits. The final student replaces the scan encoder with a depth CNN and learns through supervised imitation on student rollouts. Proprioceptive history and recurrence preserve terrain information after it leaves view.

Inputs include commanded velocity, base orientation/angular motion, joint state, previous action, and camera depth. The action is desired joint position for PD tracking. The visual encoder compresses depth; a recurrent belief combines it with body history and feeds an MLP actor. Image randomization covers clipping, noise, missing regions, camera pose, and latency, while physical randomization and pushes harden motor control.

### Why it matters

Unlike explicit elevation-map pipelines, the policy learns memory and the control-relevant visual representation jointly, reducing dependence on precise localization. The robot operates by day and night in indoor/outdoor settings, despite being smaller than platforms normally used for obstacles of similar height. The core insight is temporal: end-to-end does not mean memoryless. Distilling from a cheap oracle also separates large-scale control learning from expensive realistic rendering.

### Limitations and opportunities

The latent memory is opaque, so an incorrect remembered ledge cannot be detected explicitly. Front-only active depth provides no rear/side awareness and can fail in sunlight or on reflective surfaces. Imitation inherits oracle bias and may compound error on novel observations.

Useful follow-up work would decode the recurrent state into occupancy/contact predictions, attach uncertainty and an emergency blind gait, and evaluate deliberately moved obstacles. A small explicit spatial memory or world-model objective could preserve low-cost control while improving failure diagnosis and safety gating.

## MARCH: Model-Assisted Reinforcement Learning for the Perceptive Control of Humanoids over Sparse Footholds

### Combining a safe reference with flexible RL

MARCH targets sparse stepping stones with lateral constraints, where model-free PPO wastes samples before discovering a viable contact sequence and pure model-based control is brittle to model/perception error. A simplified biped model first generates a receding-horizon safe footstep trajectory. A privileged teacher is trained around that reference with a control-Lyapunov-function-inspired reward, then distilled to a depth-based student.

### Teacher and student details

Teacher actor and critic are MLPs. They receive a five-step proprioceptive history and privileged access to the reference, indirectly revealing safe footholds. PPO optimizes normal locomotion terms plus decrease in a Lyapunov candidate measuring trajectory error. The authors appropriately note that this is not necessarily a true CLF for the full humanoid, so it guides learning toward a useful basin rather than proving safety.

The student uses a CNN for torso-mounted depth, a Transformer to fuse visual tokens with proprioceptive history, and a mixture-density-network head producing a bimodal action distribution. Multiple modes matter: left-first versus right-first or alternative stones may both be safe, while a unimodal Gaussian can average them into an unsafe action. DAgger-style distillation trains on student-visited states without privileged height or reference information. Actions are Unitree G1 joint targets.

### Evidence and contribution

Simulation comparisons show better sample efficiency and smoother motion than model-free baselines; ablations support both Transformer and MDN. Hardware trials cover variable gaps and laterally offset stones. The contribution is the division of labor: approximate mechanics solves exploration, RL absorbs full-body mismatch, attention integrates temporal vision, and a multimodal head preserves discrete alternatives.

### Critical assessment

The learned system does not inherit the planner’s formal guarantees. Distillation can violate reference safety, depth can miss an edge, and selecting the wrong MDN component is catastrophic. A bimodal output also hard-codes limited ambiguity and may collapse.

Future work should calibrate component probabilities, connect planner viability margins to student confidence, and impose a runtime barrier/reachability filter. More than two modes or discrete contact-token prediction could cover richer foothold sets. Long sequences, moving supports, camera dropout, and recovery after a missed stone remain necessary tests.

## MEM: Multi-Modal Elevation Mapping for Robotics and Learning

MEM enriches an elevation map with aligned signals such as height, color, semantic class, surface normals, and uncertainty. A modular GPU pipeline projects multiple sensors into a common robot-centric grid and exposes the result to classical planners or learning policies. The achievement is a reusable interface between raw multimodal perception and terrain-aware navigation. Its explicit uncertainty and synchronized layers are valuable for safety and training. Like other elevation maps, it cannot fully express arbitrary 3-D occupancy, and fusion quality depends on calibration, pose estimation, and modality-specific confidence.

### Unified image/point input and layer-specific fusion

MEM extends the GPU robot-centric elevation mapper with a generic route for both point-cloud attributes and image-valued modalities. Pose and calibration project each measurement into map cells; the framework then selects a fusion rule appropriate to the data type instead of averaging everything numerically. Continuous appearance or learned features can use weighted mean/variance, class evidence can use Bayesian or class-max/average updates, and confidence or latest-observation layers can follow their own temporal semantics. Plugins derive new layers such as traversability, line detections, surface descriptors, or masks without changing the core map.

The map is therefore an explicit tensor-like interface: geometry, RGB, semantic labels, normals/features, uncertainty, and application layers share coordinates and can be consumed by classical code or neural policies. There is no single network architecture or action output; the paper demonstrates the framework across robots/sensor configurations and applications including colorization, human detection, and line detection. GPU association and fusion keep the richer representation online-capable.

The important design choice is postponing information collapse. A scalar traversability cost cannot explain whether risk comes from slope, vegetation class, low confidence, or a person; aligned layers let downstream tasks make different decisions from the same observations. But common grid coordinates do not imply statistically valid fusion. Semantic probabilities may be miscalibrated, modalities have different latency/resolution, and pose error creates systematic cross-layer ghosting. The 2.5-D surface also loses ceilings and multiple levels. Future work should model per-modality timestamps/correlation, preserve provenance, decay dynamic classes differently from static terrain, carry multi-surface geometry where necessary, and evaluate downstream decision risk—not just attractive map visualizations.

## MeshMimic - Geometry-Aware Humanoid Motion Learning through 3D Scene Reconstruction

### Recovering motion *with* its terrain

MeshMimic learns terrain-interacting humanoid skills from monocular videos. Scene-agnostic imitation can recover a plausible body trajectory while losing the stair, wall, or platform that made it meaningful. MeshMimic constructs a real-to-sim-to-real pipeline in which scene geometry, human trajectory, retargeting, and policy training stay spatially consistent.

### Reconstruction and retargeting

An off-the-shelf 3-D scene model estimates depth, camera poses, and intrinsics; a mesh is reconstructed rather than replacing terrain with planes. SAM3D-Body supplies SMPL-X pose, shape, joints, camera-relative orientation, and translation. Because scene and person have independent scale/frame ambiguity, the method jointly optimizes human trajectory and scene geometry into a metric world frame using image, contact, and collision consistency.

MeshRetargeting maps the human motion to humanoid morphology while prioritizing contact invariance and collision avoidance. This is more than joint-angle fitting: feet/hands must land on corresponding surfaces after limb lengths change. The robot-and-mesh sequence initializes a whole-body RL tracker in physics simulation. The controller receives reference motion and robot state, emits joint targets, and learns tolerance to dynamics and reconstruction residuals through randomization.

### Why it is different

The system can mine internet video in complex environments and reproduce human–scene interactions. Its strongest contribution is making the scene part of the demonstration. Compared with motion-only pipelines it retains contact meaning; compared with hand-built digital twins it scales to visual data; compared with direct video-to-policy imitation, intermediate geometry makes errors diagnosable.

### Limitations and next steps

Errors compound across camera motion, scale, body recovery, meshing, contacts, retargeting, and simulation. Joint optimization can be visually plausible but physically wrong under occlusion. Each video requires substantial processing, and the tracker may memorize one mesh rather than learn a transferable skill.

Future work should propagate reconstruction uncertainty into retargeting and RL, optimize contact forces as well as geometry, and train over reconstructed variants. Held-out geometry tests are essential. An online visual policy conditioned on live scene geometry would turn MeshMimic from scene-specific playback into reusable perceptive locomotion.

## Now You See That Learning End-to-End Humanoid Locomotion from Raw Pixels

### What “raw pixels” means here

Despite the broad phrase in the title, the deployment input is raw *stereo depth*, not RGB appearance. The paper trains one humanoid policy for precise long stair sequences and aggressive obstacles such as high platforms, wide gaps, debris, stones, trolleys, and grid holes. It avoids global reconstruction and odometry drift by mapping egocentric depth plus proprioception directly to joint targets.

### Privileged policy and heterogeneous objectives

Stage one trains a PPO teacher from dense height scans covering a `1.6 x 1.0 m` robot-centric window at 5 cm spacing. A single value function performed poorly because gap jumping, stair control, and rough-ground walking have different reward landscapes. The authors therefore use three terrain-specialized critic heads and three adversarial motion discriminators—for stairs/platforms, gaps, and rough terrain—while sharing the actor. This preserves one runtime policy but reduces gradient conflict during learning.

Stage two replaces height encoding with a depth CNN and performs DAgger-style vision-aware distillation. Behavior cloning matches teacher actions; latent alignment keeps the depth feature compatible with the height-map feature; denoising auxiliary losses make features invariant to corruption. The student acts during collection, preventing an expert-state-only dataset from hiding recovery errors.

### Sensor simulation and I/O

The stereo simulator models consistency filtering, random convolutional distortion, Gaussian and multi-scale Perlin noise, scale error, zero/max-value failures, clipping, cropping, camera extrinsic offsets up to 5 cm/0.1 rad, and a 2–4-frame observation delay. Depth is clipped to roughly 0.3–2.0 m and cropped from `30 x 40` to `24 x 32`. The actor fuses this compact depth feature with command, projected gravity/angular velocity, joint state, and previous action, and outputs humanoid joint-position targets.

### Achievement and judgment

The same policy is validated across two humanoid platforms and different stereo cameras, including a Unitree G1 with RealSense D435i. Its distinctive strength is covering fine control and parkour without a runtime map. The multi-head training and unusually detailed stereo artifact model address two real bottlenecks rather than merely enlarging the network.

The cost is opacity and local sensing. A successful action cannot be traced to a reconstructed foothold, and the tiny depth image may erase thin edges. The authors note stair descent remains particularly sensitive to near-field stereo artifacts. Terrain-specialized critics also assume a known training category even though real scenes mix them.

Next work should predict depth confidence/contact risk, fuse a short explicit memory, and evaluate abrupt sensor/camera changes. Reporting end-to-end onboard latency and category-head behavior on composite terrains would clarify whether the unified policy truly combines skills or selects learned regimes.

## Omni-Perception Omnidirectional Collision Avoidance for Legged Locomotion in Dynamic Environments

Omni-Perception gives a legged robot full-surround awareness for dynamic collision avoidance rather than relying on a forward camera. Multiple range views are fused into a compact representation and combined with proprioception and navigation commands in a learned locomotion policy. Training exposes the policy to moving obstacles, sensing noise, and varied approach directions. Its contribution is reactive avoidance integrated with gait control, allowing evasive whole-body behavior without stopping to replan. It is principally a local safety capability; occlusion, prediction of purposeful human motion, and global deadlock resolution need higher-level support.

### Temporal raw LiDAR and proximal–distal risk encoding

The policy directly consumes temporal raw LiDAR point clouds instead of first constructing an elevation or occupancy map. Its remaining input is a matching history of joint position/velocity, base linear/angular velocity, projected gravity, and the desired linear/yaw command. PD-RiskNet divides geometry by control urgency: proximal points receive fine, high-priority processing for imminent collision, while distal structure is represented more coarsely for anticipation. Hierarchical spatial features are fused over time and with proprioception, after which a PPO actor outputs joint-position targets.

Training jointly rewards velocity tracking and clearance while regularizing collision, gait stability, smoothness, torque, and posture. A GPU raycaster spanning Isaac Gym, Genesis, and MuJoCo supplies millions of temporal scans with range/beam noise, dropout, motion latency, and moving obstacles. Real tests include side/rear approaches, overhead, transparent, slender, and floor-level obstacles. The key insight is that control does not need uniform reconstruction fidelity: near geometry deserves resolution and immediate response, whereas distant geometry mainly establishes trend. Omnidirectional LiDAR also avoids the lighting and field-of-view weaknesses of a forward depth camera.

This remains reactive rather than predictive social navigation. A short scan history only implicitly separates self-motion from obstacle motion and can deadlock in crowds or passages. LiDAR has minimum-range, thin/dark-return, motion-distortion, and self-occlusion failure modes, and PPO provides no formal clearance guarantee. Explicit scene flow/time-to-collision, uncertainty calibration, a safety shield, and coupling to global route planning are natural next steps. Equal-budget ablations against range images and voxels would also show whether gains come from raw point processing or simply better sensor coverage.

## OpenGS-SLAM: Open-Set Dense Semantic SLAM with 3D Gaussian Splatting for Object-Level Scene Understanding

OpenGS-SLAM combines dense Gaussian-splatting SLAM with open-vocabulary 2-D foundation models. Explicit semantic labels are fused into 3-D Gaussians; Gaussian Voting Splatting renders label maps efficiently, while confidence-aware association reduces noisy or inconsistent 2-D predictions. The map supports object-level queries beyond a fixed training taxonomy, which is useful for language-driven navigation. Its novelty is explicit, online open-set semantics inside a dense SLAM representation. It inherits the compute and memory demands of Gaussian maps and can preserve systematic mistakes from the vision foundation model.

### RGB-D pipeline, label consensus, and open-set map output

The input is a live RGB-D sequence. RGB frames pass through an interchangeable semantic generator—YOLO-World for open-vocabulary category evidence and a full-scene segmenter such as MobileSAMv2—while generalized ICP estimates the current camera pose. Keyframes densify a 3-D Gaussian map, with G-ICP covariance used to initialize new primitives. Each Gaussian stores ordinary differentiable appearance/geometry parameters plus an explicit, non-differentiable semantic label; the output is a tracked pose and dense object-level 3-D map that can render RGB, depth, or labeled views.

Gaussian Voting Splatting is the important efficiency mechanism. For each pixel the renderer records the top 50 contributing, depth-sorted Gaussians and accumulates alpha-compositing weight by label; the label with greatest total contribution wins. This both renders a map prediction and identifies exactly which primitives should change when new evidence disagrees. Confidence-based 2-D label consensus then classifies input/map overlap as full match, partial match, whole match, or new object. Confidence uses object completeness in the current view; repeated evidence decays spurious part labels created by over-segmentation. “Counter Gaussians” inconsistent with better-supported segment boundaries can be pruned, allowing cleaner replacements.

Unlike feature-distillation methods, this design does not store a large CLIP-like vector in every primitive and needs no predefined closed class list. Its highlight is solving the mundane but central online problem that the same object is assigned different segment IDs—or split into parts—from different viewpoints. Explicit labels make queries and debugging easier and yield real-time rendering.

Open-set nevertheless should not be confused with error-free semantics. It is inherited from the 2-D foundation detectors; missed objects, category hallucinations, and similar adjacent instances can still be fused incorrectly. G-ICP assumes usable depth/geometry, and irreversible consensus may turn an early mistake into persistent map state. The Replica/TUM-style evaluation is also much shorter and cleaner than lifelong robot operation. Natural next steps include reversible data association, label/posterior uncertainty, loop-closure-aware semantic correction, memory bounds, dynamic-object handling, and task-level tests showing that object queries improve actual navigation success.

## Opening the Sim-to-Real Door for Humanoid Pixel-to-Action Policy Transfer

### Locomotion tested through door interaction

DoorMan uses door opening as a difficult loco-manipulation benchmark: a humanoid must keep a handle visible, reach and rotate it, counter spring/hinge forces, move with the swinging panel, and walk through. The deployed student uses RGB and proprioception only—no depth, object pose, or scripted primitive—and is trained entirely in simulation.

### Teacher–student–bootstrap training

A privileged PPO teacher observes exact door/object state and builds on a pretrained whole-body controller. Staged resets initialize training at progressively earlier phases of the task, preventing long-horizon sparse reward from overwhelming exploration. The DAgger student acts from partial observations while the teacher labels its states.

The RGB frame passes through a jointly fine-tuned visual encoder. Its latent is concatenated with joint angles/velocities and root angular velocity, then processed by two 512-unit LSTM layers; a `[512,256,128]` MLP outputs target joint angles. The action covers the 29-DoF G1 body plus controlled hand joints (the paper describes a 33-dimensional control action), and inference runs at 50 Hz.

After imitation, GRPO fine-tunes the student from grouped rollout returns without a learned critic. This bootstrap matters under partial observability: the teacher never needed to move to keep the handle in view, whereas the RGB student can discover viewpoint-preserving behavior. Physics randomization spans door type/dimensions, hinge damping, latch behavior, handle position, and resistance; photorealistic rendering randomizes materials, lighting, intrinsics/extrinsics, motion blur, and auto white balance.

### Hardware evidence and assessment

The Unitree G1 uses a RealSense D435i’s RGB stream only; inference is reported on an external i9/RTX 4090 workstation. Across diverse handles and panels, the system opens/passes through doors and reports up to 31.7% faster completion than human teleoperators using the same underlying whole-body stack.

The important general lesson is that distillation is an initialization, not the endpoint, when the student lacks teacher information. Online RL lets perception and body motion co-adapt. But the method is still task-specific, computationally heavy, and dependent on photorealistic coverage; pure RGB provides no explicit geometric confidence. External compute and successful door types limit autonomy claims.

Future work should add tactile/force feedback and uncertainty without restoring object-pose dependence, run inference onboard, and test glass doors, occlusion, changing illumination, and failed grasps. Reusing the training recipe across drawers, gates, and cabinets would establish that it is a general pixel-to-action framework rather than an excellent door specialist.

## Real-Time Polygonal Semantic Mapping for Humanoid Robot Stair Climbing

### Mapping paper, not a locomotion policy

This work builds a globally consistent map of traversable planar polygons from depth images and arbitrary odometry, aimed at humanoid gait planning on straight and spiral stairs. A polygonal tread representation directly answers whether a foot-shaped region fits, unlike a dense cloud or generic voxel. It proposes no RL actor or motor network.

### Per-frame GPU pipeline

Raw depth passes through anisotropic diffusion, reducing within-plane noise while preserving discontinuities at stair edges. The system computes normals, detects contours, segments candidate regions, and runs GPU-parallel RANSAC to fit a plane to each polygon. Filtering and fitting achieve single-frame rates above 30 Hz.

Polygons are transformed with robot pose and incrementally associated/merged into the global map using normal, distance, overlap, and height consistency. Stored information includes boundary, normal, centroid height, and semantic orientation. Vertical-direction constraints stabilize tread/riser estimates. A gait planner then chooses sufficiently large horizontal polygons as footholds.

### Contribution and evidence

The main contribution is showing that edge-preserving filtering closes much of the simulation-to-real plane-extraction gap before fitting. GPU parallelism makes structured geometry available during walking. Tests cover straight and spiral stairs and connect mapping to gait planning rather than reporting segmentation alone.

### Assessment

The representation fits built environments but not rocks, vegetation, curved/deformable surfaces, or cluttered treads. Accuracy depends on odometry; merging can turn pose bias into a confidently wrong global plane. Dynamic obstacles and uncertainty remain open.

Next work should maintain covariance on plane boundaries/poses, age or retract changed surfaces, and retain residual occupancy for non-planar objects. Planners should erode polygons by foot size plus uncertainty. Missed and false footholds matter more for safety than average IoU and processing rate.

## START: Traversing Sparse Footholds with Terrain Reconstruction

### Explicit local memory without global SLAM

START targets stones, balance beams, stepping beams, and gaps using a low-cost, view-limited depth camera. Direct visual latents can forget centimeter-scale edges after they leave view, while global height maps require localization and costly fusion. START’s TR-Net reconstructs a robot-local height map—including terrain beneath and behind the body—from current depth and proprioceptive memory.

### Single-stage reconstruction and control

TR-Net encodes egocentric depth and temporal body state, recurrently aligns information in a local robot frame, and decodes a dense height grid. Ground-truth simulation height supplies an auxiliary reconstruction loss. The locomotion actor consumes both the reconstructed map and latent features plus command/proprioception and emits joint-position targets; an asymmetric critic sees privileged state. All modules train together with PPO rather than freezing a perception model or distilling an oracle.

Adaptive Sampling (AdaSmpl) biases training toward terrain/configurations where reconstruction or policy error is informative. This reduces the huge exploration cost of uniformly sampling either impossible gaps or already-mastered steps. Standard progress, command, posture, contact, effort, and collision rewards are augmented by precise sparse-support objectives.

### Evidence and distinctive contribution

The low-cost quadruped transfers zero-shot to indoor and outdoor sparse terrain with agile, less rigid gaits than implicit-embedding baselines. Visualizations show TR-Net retaining geometry outside the current camera view. The strongest idea is using explicit reconstruction as a *task-trained local memory*: it keeps physical meaning and precision without global odometry.

### Limits and extensions

Local-frame recurrence still depends on estimated body motion and can smear old geometry. L1-like height accuracy may hide edge/topology errors most relevant to a foot. A 2.5-D surface cannot represent overhangs; moved supports leave stale memory; the reconstruction has no calibrated confidence.

Next work should predict occupancy/support probability and uncertainty, weight loss around reachable landing boundaries, and erase geometry using change detection. Comparing map and implicit latent at equal capacity/latency would isolate the benefit of explicit supervision. Long sequences with deliberately moved stones would test whether memory is robust or merely persistent.

## TAGA: Terrain-aware Active Gaze Learning for Generalizable Agile Humanoid Locomotion

### “Gaze” means selecting map detail

TAGA combines a forward depth preview with a precise local height scan. It does not physically rotate the head; its active gaze predicts which region of the height scan deserves high-resolution processing. Vision sees distant terrain, while registered geometry supports exact contacts near the body.

### Exact inputs and architecture

The actor receives a `36 x 64` depth image, a `3 x 21 x 21` XYZ height scan, and five proprioceptive frames. Each frame contains torso angular velocity, projected gravity, 29 joint positions/velocities, previous action, and forward/lateral/yaw command. Output is 29 joint targets.

A CNN maps depth to a 128-D look-ahead embedding and an MLP maps proprioception to 128-D. A lightweight head predicts normalized ROI crop parameters in the height scan. Cross-attention fuses the selected geometry with visuomotor context; a mixture-of-experts decoder/gate produces action. An auxiliary head aids representation, but the ROI has no gaze label—useful attention emerges from PPO reward.

### Results and importance

Onboard Jetson Orin drives a G1 across stairs, beams, stones, outdoor ground, elevated platforms, and reported real gaps up to 1.2 m. The system remains robust under perception disturbance and trains faster than full-context processing. The key contribution is allocating compute: a large map is affordable when only task-relevant detail enters expensive fusion.

### Critical assessment

Cropping creates a blind spot: once gaze is wrong, omitted hazards cannot influence action. Height scans still require mapping/localization, so TAGA inherits drift. Its “active perception” is attention routing, not camera motion.

Next work should preserve a cheap global hazard branch, model ROI uncertainty or multiple glimpses, and test distractors. Physical gaze control could improve coverage but must price blur and balance. Correlating ROI with actual future contacts on held-out terrain would test whether gaze is causal.

## VB-Com: Learning Vision-Blind Composite Humanoid Locomotion Against Deficient Perception

### Choose the policy likely to survive the next few seconds

VB-Com pairs a proactive vision policy with a reactive blind policy. The former handles gaps, hurdles, and walls quickly when geometry is correct; the latter relies on reliable proprioception and can recover when a surface deforms, an obstacle suddenly appears, or the sensor is occluded. Rather than detecting bad pixels explicitly, two return estimators predict the future return each candidate policy would obtain from the current proprioceptive state.

### Policies and compositor

Both PPO policies share action space/reward. The vision actor receives goal/direction command, IMU/joint state, prior action, and an external terrain observation; the blind actor receives the same deployable body/command signals without vision. Training uses a goal-reaching formulation better suited to dynamic maneuvers than strict yaw-rate tracking. The vision actor’s terrain area is roughly `1.2 x 0.7 m`, while the training critic sees a larger `1.6 x 1.0 m` privileged map.

Each return estimator is trained alongside its corresponding policy from generalized-advantage/return targets but takes hardware-available proprioception. At runtime both actors propose an action, the estimators score their expected cumulative outcome, and a categorical selection mechanism executes one. Thus a sudden proprioceptive mismatch can favor the blind recovery action even when the erroneous visual map still looks confident.

### Results and distinction

Unitree G1 and H1 experiments cover suddenly appearing hurdles, deformable gaps, sensor occlusion, and dynamic terrain. VB-Com recovers where a vision-only policy follows stale geometry, while retaining anticipatory speed unavailable to a blind-only baseline. The contribution is capability-based arbitration: estimate which controller will succeed rather than classify every failure type.

### Critical assessment

The estimators themselves face distribution shift and may share errors; return is not calibrated failure probability. Per-step switching can chatter and issue discontinuous joint targets. The blind policy cannot save a robot before contact with a large invisible gap, and running two actors adds compute.

Future work should add hysteresis/options-level commitment, uncertainty ensembles, and a conservative stop action. Training the estimator on explicit held-out perception failures and calibrating it against fall probability would make selection safety meaningful. Smooth residual composition might retain useful vision while allowing proprioceptive correction instead of hard switching.

## ViBe: Visual Behavior Adaptation for Perceptive Humanoid Whole-Body Control

[ViBe](https://arxiv.org/abs/2609.09918) post-trains an existing motion tracker for visual tasks instead of rebuilding control from scratch. Frozen pretrained visual features pass through a multi-query extractor, and low-rank adapters graft task-relevant feedback into the tracker. Policy optimization then adapts only a compact set of parameters using task reward and reference motion. The method transfers across curb walking, parkour, object interaction, and dodgeball, including difficult lighting and distractors. Its modular efficiency is compelling, though visual backbone bias and reward design still bound what the adapted behavior perceives.

### Post-training visual graft onto a motion tracker

ViBe starts with an already competent reference-motion tracker, whose proprioceptive actor can reproduce broad humanoid behaviors but has no exteroception. A pretrained visual foundation encoder processes RGB; a multi-query extractor learns several task-relevant visual summaries rather than compressing the image with one generic pooled token. These visual features are grafted into selected tracker layers through low-rank adapters. Most tracker and visual-backbone weights can remain fixed, so adaptation changes a small parameter set and retains the motion prior instead of retraining a new geometry policy or distilling a privileged teacher.

Training uses direct policy optimization with the target task reward and a reference-motion dataset. Inputs are RGB, tracker proprioception/state history, and motion/task command; outputs remain the original whole-body joint action/target interface. The task reward teaches visual queries and adapters which features matter for interaction, while motion tracking prevents the controller from forgetting plausible, stable humanoid movement. This is RL fine-tuning of a modular perception-conditioned policy, not behavioral cloning from a separate perceptive teacher.

The breadth of reported zero-shot sim-to-real transfer is the highlight: curb and parkour walking, Repose Cube, omni-object loco-manipulation, and dodgeball, including outdoor, low-light, and RGB-distractor conditions. A deliberately simple high-level planner solving goal-directed Repose Cube suggests the adapted controller contains meaningful closed-loop visual competence rather than requiring a sophisticated planner.

The design makes a strong efficiency claim—reuse large visual and motor priors, train only task-facing connections—but also couples safety to opaque foundation features. RGB lacks explicit metric geometry, several tasks may demand different queries, and low-rank updates can still create rare catastrophic behavior outside the reward distribution. Precise ablations should compare frozen/random/pretrained encoders, adapter rank/query count, and full fine-tuning at equal samples. Future work should add temporal visual memory, uncertainty and out-of-distribution detection, depth/tactile fusion where metric contact matters, independent collision constraints, and continual adapters that compose behaviors without interference.
