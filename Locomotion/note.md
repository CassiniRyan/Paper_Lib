# Locomotion — Paper Notes

These papers pursue different kinds of locomotion progress: faster or more dynamic skills, safer traversal, better terrain representations, scalable training, recovery, or broader motion priors. Each entry is organized around the paper's distinctive idea and the evidence offered for it. Perception, controller architecture, observations/actions, temporal context, training method, or rewards are discussed when they are central—not as a forced checklist—and every entry closes with a technical judgment and useful follow-up directions. The analysis is based on the locally archived full papers and supplements wherever available; the two entries without local PDFs use their linked primary project/arXiv sources.

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

## ANYmal Parkour Learning Agile Navigation for Quadrupedal Robots

### Problem decomposition

ANYmal Parkour is a complete autonomous stack rather than one policy. Five low-level RL skills handle walking, jumping, climbing up, climbing down, and crouching. A neural perception system reconstructs local 3-D terrain and maintains a compact belief. A learned navigation policy receives that belief plus the goal, selects one of the five skills, and produces continuous intermediate commands. This separation lets each motor expert train on a single obstacle family while the navigation layer learns timing, routing, and switching across randomized courses.

### Sensors and learned interfaces

ANYmal-D carries six RealSense depth cameras—two front, two rear, one left, one right—and a Velodyne Puck LiDAR. Point clouds enter a multi-resolution neural reconstruction model: high resolution near the robot supports exact contact, while coarser distant geometry expands look-ahead and represents overhangs that elevation maps cannot. The perception node runs asynchronously on a Jetson Orin. Navigation and locomotion consume its latest latent/map on the onboard computer, so training-to-deployment delay is not perfectly synchronous.

The navigation actor uses a modified PPO output: a categorical distribution chooses the skill and a Gaussian distribution produces continuous position, heading, and timing commands. This is a more important detail than simply saying “hierarchical RL,” because the high level decides both *what controller* and *how to parameterize it*. Low-level skills use a position-based goal formulation and symmetry augmentation. They output joint commands that exploit the full 12-actuator quadruped, including knees and shanks as environmental contacts.

### Results and emergent behavior

The real robot reaches 2 m/s, clears gaps up to about 1 m, climbs or descends platforms up to about 1 m, traverses 40° slopes and 0.25 m stairs, and crouches through 0.4 m passages. Learned routing changes with obstacle height: it takes a direct climb/descend when within a skill’s capability and chooses a longer route when the geometry becomes marginal. Across three randomized test layouts, navigation succeeds at 96.3–98.2%, while hand-coded target sequences fall to 60.9% and 75.3% on two layouts.

The skills go beyond nominal labels. The jump expert is sometimes selected to turn rapidly in a narrow passage because its training required heading changes on small platforms. Climb-down learns to lower the body, hook shanks/knees, and limit impact rather than simply drop. The mapping model remembers an overhead tabletop after it leaves view and corrects its belief after localization jumps. These behaviors show why the learned latent and navigation policy matter more than a fixed skill state machine.

### Critical assessment

The work convincingly demonstrates autonomous agile chaining, but modularity is both strength and weakness. Each expert can be optimized to its actuation limit, yet a categorical switch cannot synthesize a behavior between experts. Navigation performance depends on perception latency, reconstructed collision geometry, and skill coverage. When a novel obstacle requires a new contact sequence, the system can only misuse an existing expert or route around it.

Peak motor saturation and 160° joint excursions show impressive capability but also leave little thermal or safety margin. Reported success should be accompanied by impact, temperature, intervention, and hardware-wear statistics. An uncertainty-aware selector could slow or refuse a maneuver when reconstructed dimensions lie near an expert’s limit. Longer-term work should replace hard skill identity with a compositional latent or residual mixture while retaining the useful modular debugging boundary. Dynamic-object prediction and a recover-or-retreat skill would also improve deployment outside staged courses.

## APEX: Learning Adaptive High-Platform Traversal for Humanoid Robots

### What problem APEX solves

APEX targets platforms around `0.8 m`, roughly 114% of the robot’s leg length—well above the range of ordinary foot-only stepping. Instead of learning an impulsive jump, it learns controlled multi-contact maneuvers that distribute load across hands, torso, knees, and feet. Six skills cover walking, climb-up, climb-down, stand-up, lie-down, and the transitions required for complete ascent/descent. They are first trained separately and then distilled into one context-aware perceptive policy.

### Ratchet progress as the central idea

Contact-rich ascent has no clean per-timestep reference: useful behavior may pause while establishing a hand or knee support. A distance reward can be exploited by moving forward and backward, while velocity shaping encourages dangerous lunges. APEX maintains the best task-space progress achieved so far and rewards only strict improvement. This history-dependent “ratchet” supplies dense feedback, prevents reward from retracing, and permits patient intermediate contacts. The critic observes the best-so-far task state so value estimation remains Markovian enough for PPO.

Different maneuvers define progress in task variables appropriate to their goal—body height, CoM/head placement, or terminal posture. Rewards combine ratchet task progress, a final-pose term after success, ordinary stability/regularization, termination, and an exponential penalty above a contact-force safety threshold. Comparative reward experiments show distance, direction, curiosity, sparse success, and incremental-distance baselines either fail to discover the maneuver or exploit unsafe shortcuts.

### Observation, action, and network design

The actor sees five frames of proprioception (base/angular state, 29 joint positions/velocities, and prior 29-D action), current task state, and for climb skills a `21×21 = 441` local elevation map. It outputs 29 joint-position targets. Each specialist actor/critic is an MLP `[512,256,128]`; PPO trains the teachers. The unified distilled student is much larger, `[2048,1024,512,256]`, and is pretrained with behavioral cloning then refined through teacher action supervision, balanced “divide-and-conquer” sampling, symmetry augmentation, and action noise.

Deployment uses a Livox MID-360 with LiDAR–inertial mapping. Training injects vertical offsets, rotations, slopes, missing/outdated cells, and other map artifacts; runtime spatial filtering and inpainting remove outliers and NaN holes. These two defenses are complementary: policy randomization handles residual error while preprocessing prevents pathological maps from reaching the network.

### Evidence and judgment

The unified policy completes end-to-end traversal with 95.4% success over more than 1,000 simulated trials and transfers zero-shot to Unitree G1. It adapts climbing to approach angles around ±65°, survives strong disturbances, and works on a soft foam/vinyl platform because its quasi-static multi-contact strategy does not rely on a rigid impulsive launch.

APEX’s standout result is a *safe-learning formulation for progress without a reference*, not merely high climbing. The ratchet principle should transfer to opening heavy doors, crawling through windows, or manipulating while braced. Still, platform traversal is represented through a 2.5-D map and a fixed skill library. It does not reason about surface load capacity, friction uncertainty, or semantic permission to use hands. The student may also smooth rare specialist actions during distillation. Future work should carry teacher uncertainty into routing, add contact-force/tactile observations, certify surface/support feasibility, and test whether one generative policy can retain multiple safe strategies instead of collapsing each maneuver to its dominant solution.

## Architecture Is All You Need Diversity-Enabled Sweet Spots for Robust Humanoid Locomotion

### Research question

This paper challenges the assumption that stronger perceptive locomotion requires a more exotic network. It studies whether a simple layered control architecture (LCA)—a slow terrain encoder above a fast proprioceptive stabilizer—creates the useful optimization structure by itself. The central claim is a diversity-enabled “sweet spot”: once terrain perception and fast feedback occupy different layers/timescales, an ordinary MLP can rival more complex recurrent or monolithic designs.

### Inputs, architecture, and training

The policy history contains projected gravity, base angular velocity, joint positions and velocities, previous action, and planar velocity commands. The actor also receives an `11×11` height map covering `1×1 m`; the asymmetric critic gets a clean `1.5×1.5 m` map and privileged base/terrain information. Height values are base-relative, zero-centered, and clipped. A small perception encoder is either a CNN (`3×3`, stride 1) or an MLP with `[256,256]` hidden sizes. Its latent is concatenated with proprioception and passed to an actor that is either an MLP or LSTM with `[512,256,128]` capacity. The critic uses the same hidden sizes.

Training has two stages. Stage 1 disables actor height-map input and learns the fast stabilizer on diverse locomotion. Stage 2 re-enables perception and fine-tunes terrain-critical behavior. This differs from merely pretraining a visual encoder: the lower-level policy begins from a competent blind control solution. Rewards cover command tracking, contact phase, foot strike/slide/orientation, swing clearance, smoothness, energy, and limits.

Real deployment uses two downward-facing depth cameras—one low near the hip to see between/behind the legs and one at the chest for look-ahead. Their clouds merge at 30 Hz into a local map. The policy runs at 50 Hz and sends position setpoints to 1 kHz PD control.

### Evidence and interpretation

Layered policies consistently reduce harsh contacts relative to one-stage equivalents and dominate on the most perception-critical physical tasks. On four hardware tasks, the two-stage MLP succeeds in 4/5, 5/5, 5/5, and 4/5 trials, whereas the one-stage MLP largely fails and the monolithic variant is omitted because it never solves some tasks. The CNN can help, but the major gain comes from stage/layer separation, supporting the architectural thesis.

The result should not be overgeneralized as “architecture is literally all.” Success still depends on terrain diversity, height-map construction, reward design, and a privileged critic. The experiment establishes that a well-posed layered optimization can matter more than choosing CNN versus MLP on a small map. It does not show the same outcome for raw high-resolution vision, dynamic obstacles, or multi-contact parkour where the perception/control boundary is less clean.

### What should come next

An equal-compute study should vary history length, encoder update rate, and stage-one diversity independently. Freezing versus fine-tuning the proprioceptive core would clarify whether the benefit is modular preservation or simply better initialization. The paper’s control-theoretic framing also invites frequency-domain analysis: measure which disturbances are rejected by the fast branch and which terrain variations require the slow branch. Adding uncertainty to the map latent could let the architecture fall back smoothly to blind walking. The most valuable practical lesson is to establish a strong proprioceptive controller before increasing perceptual capacity—not to assume that small networks solve every environment.

## Attention-Based Map Encoding for Learning Generalized Legged Locomotion

### Core contribution

This earlier AME work introduces the central observation later extended by AME-2: a terrain map should be read relative to the robot’s current state. A CNN forms a feature for every point in a robot-centric height map; a multi-head attention module uses proprioception as the query and point features as keys/values. Without foothold labels, high attention tends to appear on future support regions. The same architecture is trained on ANYmal-D and the humanoid Fourier GR-1, demonstrating that the inductive bias is not tied to one morphology.

### Exact information flow

Each map sample carries `(x,y,z)`. Two convolutional layers with kernel size 5 first build local terrain features: the first has 16 channels and the second produces the remaining dimensions of a `d`-dimensional point token; coordinates are concatenated back so spatial location is never lost. An MLP embeds proprioception into one query. Multi-head attention pools the `L×W` terrain tokens into one task-conditioned map encoding, which an MLP maps to desired joint positions.

The actor observes base linear/angular velocity, projected gravity, joint positions/velocities, the previous action, command, and height-map points. The critic receives clean versions. PPO runs across 4,096 parallel environments. Task rewards track linear/yaw velocity; regularizers penalize action rate, acceleration, torque, limits, undesirable contact and unstable torso/feet; style terms shape stance and arm posture. Stage 1 trains from scratch on a small set of base terrain families. Stage 2 adds novel sparse layouts, disturbance, perception uncertainty, and more difficult curricula.

### Results and meaning

On hardware, GR-1 and ANYmal-D traverse beams, gaps, random stones, boxes, and mixed sparse terrain. Whole-body reflexes emerge: GR-1 uses arms against a wall and ANYmal uses knees when a nominal foot-only solution becomes marginal. Both cross a previously unseen obstacle parkour at 100% success in the reported test. Visualized attention shifts with command direction and concentrates around prospective contacts, making the representation more interpretable than an undifferentiated map latent.

The model resembles a learned contact planner followed by a whole-body controller, but both functions are optimized end to end. That is its highlight: attention supplies a soft, differentiable support selection mechanism without a discrete foothold optimizer.

### Limitations and next work

A 2.5-D height map still assumes one relevant surface per cell and inherits localization/map errors. Attention weights are not a safety certificate or necessarily a causal explanation. The stage-two curriculum may teach recovery only for the particular disturbances and sparse geometries sampled. Network comparisons must also equalize parameter count and spatial resolution before attributing gains solely to attention.

AME-2 improves several of these issues through uncertainty-aware neural mapping and a stronger teacher–student pipeline. For this original paper, the clean next step is to attach uncertainty/age to map tokens and penalize confident attention on poorly observed cells. Counterfactual masking can test causality. Multi-layer or voxel tokens would extend the method to overhangs, while a feasibility head could translate the visually intuitive attention map into an explicit probability of safe contact.

## BeamDojo Learning Agile Humanoid Locomotion on Sparse Footholds

### Why sparse footholds require a different training scheme

BeamDojo addresses two problems specific to humanoids. First, a foot is a polygon, so rewarding only its center can call a dangerous edge placement correct. Second, a sparse foothold reward is rare and easily drowned by dense velocity/stability rewards. The method samples points over each sole and scores the fraction lying on safe support, then gives that reward its own critic and independently normalized advantage.

Two-stage training solves exploration. In the “soft terrain dynamics” stage, the robot physically walks on flat ground while observing the height map of stepping stones/beams; foothold rewards are computed against the virtual sparse terrain, but a miss does not cause a fall or termination. Stage 2 restores the real gaps and fine-tunes under hard dynamics. This lets the actor experience many positive and negative placements before it must also solve survival.

### Observations, actions, and networks

The actor receives proprioception—base angular velocity, projected gravity, 29 joint positions, 29 joint velocities, and previous action—plus a `15×15` local elevation map and motion command. Although the G1 has 29 DoFs, the learned action is 12 target positions for the lower-body joints; upper-body targets are prescribed for balance. Actor and both critics are MLPs with hidden sizes reported as approximately `[512,216,128]`. PPO trains one shared actor, while each critic learns either dense locomotion return or sparse foothold return; their normalized advantages are summed for the policy update.

Deployment uses Livox MID-360 data fused with its IMU through FAST-LIO and a robot-centric elevation mapper. Maps update at 10 Hz, the actor runs at 50 Hz, and PD control at 500 Hz. Training randomizes vertical map offset, rotation/odometry error, tilt gradients, latency/repeated frames, actuator gains, friction, restitution, and dynamics.

### Evidence

Ablations show both soft-dynamics pretraining and the double critic raise curriculum level, success, and placement accuracy. The double critic also preserves gait smoothness because sparse foothold advantages no longer distort dense regularization learning. On hardware the policy crosses stepping stones, balancing beams, and gaps—including layouts held out of training—moves backward, tracks commands up to 1.0 m/s, survives payloads and pushes, and stabilizes on flexible supports.

### Critical assessment

BeamDojo’s most reusable insight is reward/critic decomposition. The method does not merely add a foothold penalty; it changes exploration and credit assignment so exact placement can coexist with natural gait. The polygon-sampling reward is also better aligned with support area than a point-foot metric.

The controller remains limited by a single-layer elevation map and lower-body action space. It cannot duck under overhangs or deliberately use hands/knees, and a map bias of only a few centimeters can move the safe polygon outside a narrow beam. Stage 1 also presents a perceptual/dynamic inconsistency—the robot sees holes while being supported by flat ground—which might teach optimistic behavior. Future work should compare this to safety nets or curriculum-width interpolation, add confidence-aware foot margins, and allow whole-body contacts. A distributional foothold critic could estimate failure probability, giving a planner more useful information than one expected value.

## Collision-Free Humanoid Traversal in Cluttered Indoor Scenes

### HumanoidPF as both observation and reward

This work targets clutter around the entire humanoid, not only floor hazards. A planar base footprint cannot express shoulders hitting a doorway, a head striking a shelf, or a knee colliding during a crouch. HumanoidPF reformulates an artificial potential field over the full articulated body. An attractive term uses obstacle-aware geodesic distance toward the goal; repulsive gradients come from scene signed distance. The field is sampled at 13 key body parts and supplied directly as a compact policy observation.

The same field defines a dense directional reward. For every body part, desired direction and concentration parameter form a von Mises–Fisher distribution, and the log likelihood of actual link velocity becomes the reward. Root motion receives higher static priority, while a dynamic urgency weight increases when a moving link approaches an obstacle. This resolves a common potential-field ambiguity: equal left/right repulsion can cancel. Tiny asymmetries and body-part priority are amplified into one coherent passage choice.

### Training and deployment stack

Specialist PPO policies train on 139 cropped `5×5 m` blocks from 3D-FRONT and 216 procedurally generated scenes. The synthetic layouts add hanging, floor, and closely spaced obstacles that ordinary household datasets undersample. Each specialist uses 32,768 parallel environments and millions of episodes. DAgger then distills scene-specific teachers into one generalist, with observation noise, force perturbations, and progressively tighter curriculum.

At deployment, Click-and-Traverse combines Fast-LIO2, OctoMap, and field construction at 10 Hz. Forward kinematics queries HumanoidPF at the 13 current body locations; a user clicks a goal, and the controller generates whole-body traversal without joystick or mocap. Tests include crouch, hurdle, sideways passages, and combined low/high obstacles.

### Evidence and interpretation

HumanoidPF outperforms multilayer elevation and voxel observations under real perception, and field-derived rewards generalize better than hand-tuned collision penalties. Hybrid scene generation is critical: real-scene crops provide realistic layouts, while procedural “hard” scenes force skills near the robot’s geometric limits. The representation acts as a spatial low-pass filter, making small mapping noise less consequential than in raw occupancy.

The paper’s real novelty is an *interface between mapping and RL*: instead of sending dense geometry and hoping the network learns collision relationships, it sends an actionable direction at each body part. That improves learning efficiency, interpretability, and transfer.

### Limitations and extensions

Potential fields still have structural failure modes: local minima, narrow-passage oscillation, and purely reactive behavior around moving obstacles. Priority weighting mitigates but does not eliminate them. Fast-LIO/OctoMap assumes sufficiently observable, mostly static geometry; 10 Hz updates can lag humans or doors. The controller also avoids contact entirely, so it cannot brace, lean, or intentionally touch a rail when avoidance is impossible.

A global planner or learned topological memory should guide around potential-field minima. Dynamic occupancy and velocity prediction are needed for people. Safety evaluation should report minimum body clearance, collision impulse, and mapping latency, not only arrival. Finally, merging HumanoidPF with contact-affordance fields could let the same representation distinguish forbidden collision from useful support, broadening the method from collision-free traversal to contact-rich navigation.

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

## Deep Whole-body Parkour

### From motion imitation to scene-conditioned interaction

Deep Whole-body Parkour augments motion tracking with depth so a humanoid can align agile, multi-contact human skills to actual obstacle geometry. The library includes vaulting, hurdling, scrambling, and dive/roll-like behavior in which hands, knees, torso, and feet may all matter. A blind tracker can reproduce the pose sequence yet miss a box edge; the visual policy can shift its root and contact timing while retaining the reference’s dynamic structure.

Human demonstrations are captured around physical obstacles, cleaned to preserve contact, and paired with terrain. Training randomizes motion–terrain placement so the policy cannot solve the task from clip time alone. A custom grouped GPU ray caster assigns every parallel robot/scene a collision group and checks rays only against static geometry plus that agent’s meshes, yielding about 10× higher rendering throughput for thousands of environments.

### Network inputs, training, and deployment

At every step the actor receives ten future reference frames spaced 0.1 seconds apart—a full one-second expectation of joint position, joint velocity, relative base position, and base rotation—plus one noisy depth frame and eight frames of proprioceptive history. It contains no global odometry. Reference translation is expressed in a hybrid relative frame: robot planar position/yaw with reference height/roll/pitch, allowing local alignment without accumulating a world-frame trajectory error.

A CNN encodes depth and a three-layer MLP fuses it with proprioception/reference features. Asymmetric PPO trains the actor; the critic receives future reference in actual-root coordinates, key-link poses, height scans, and the same eight-frame history. Rewards track root pose, local key-link pose/rotation, global link velocities, and regularize action rate, limits, undesired contacts, and torque. Failure-based curriculum bins every motion in one-second intervals, oversampling segments where rollouts terminate; stuck rollouts are truncated.

The deployed G1 uses an Intel RealSense D435i and runs both depth and policy around 50 Hz with ONNX CPU acceleration. Gaussian and patch artifacts plus GPU inpainting narrow the depth gap. A state machine still chooses the active reference skill; the learned policy performs geometric alignment and execution, not autonomous semantic selection.

### What depth actually adds

Depth reduces global alignment error especially for interaction-intensive motions. Relative-frame rewards prevent the controller from being penalized for necessary planar correction. Distractor experiments are revealing: walls or moderately wide objects often leave local pose tracking intact, while a large plane disrupts behavior because its visual structure resembles the task obstacle. This shows both genuine feedback and shortcut risk.

The paper’s important contribution is a one-stage infrastructure for depth-conditioned, reference-driven multi-contact motion—not a general parkour planner. It closes the loop between obstacle geometry and expressive motion, while high-throughput ray casting makes such training practical.

### Critical assessment and next directions

Reference selection, obstacle identity, and approximate initial alignment remain external. Ten future frames give strong privileged intent; a policy without the correct clip cannot invent a new maneuver. One image at a time plus recurrent proprioception may be insufficient when hands or torso occlude the surface. Contact forces and tactile state are absent even though the tasks rely on hands and body impacts.

Next work should learn retrieval or generation of references from scene geometry, attach feasibility confidence to each motion–obstacle pair, and switch/replan before commitment. Wrist or chest depth and contact sensing could correct late-stage alignment. Training should include visually similar but physically irrelevant distractors to reduce shortcut learning. Reporting peak force, fall severity, and hardware intervention is essential: reproducing a dramatic vault is not enough if its impact distribution is unsafe.

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

## Extreme Parkour with Legged Robots

### One policy rather than an expert library

Extreme Parkour trains one Unitree A1 policy for high jumps, gaps, hurdles, tilted ramps, and an optional front-leg handstand style. Unlike ANYmal Parkour, it does not select among separate motor experts. Phase 1 uses privileged terrain scan dots and waypoint-derived heading; Phase 2 distills both terrain representation and heading inference into a front-depth policy.

### Reward and control interface

The key reward is a world-frame inner product between robot velocity and the direction to the next waypoint, clipped by desired speed. It rewards progress without prescribing body orientation or gait and prevents the agent from turning around an obstacle to exploit base-frame tracking. A foot-clearance penalty marks contacts within 5 cm of terrain edges. A second inner-product style reward can rotate the body’s forward vector toward a commanded direction, enabling the handstand mode.

Phase-1 actor input is proprioception, scan dots, desired heading, speed, and a style flag. Regularized Online Adaptation estimates hidden environment/dynamics variables from observation history. Curriculum promotes an environment after crossing half the obstacle field. Phase 2 replaces scan dots with a ConvNet–GRU depth encoder and trains through DAgger while student actions drive rollout. A Mixture of Teacher and Student rule uses predicted yaw only when it lies within 0.6 rad of oracle direction, preventing early heading error from corrupting teacher labels.

### Hardware timing and results

A single RealSense D435 produces `58×87` processed depth at about `10±2 Hz`; the depth network runs at 10 Hz and the base policy at 50 Hz on Jetson NX. Visual latency is deliberately fixed at 80 ms and proprioceptive latency at 16 ms, turning jitter into a learnable constant. The robot crosses gaps up to twice body length and obstacles around twice standing height, adjusts direction across tilted ramps, and uses emergent front/hind-leg roles.

### Assessment

The standout result is a simple reward/interface that permits morphology-appropriate strategies rather than scripting jump phases. The MTS curriculum is also a practical response to compounding errors in perceptual DAgger. However, the policy’s “one network” claim still relies on privileged waypoint direction during teacher learning and fixed obstacle families. A front camera cannot see landing surfaces after takeoff, and 10 Hz vision leaves long open-loop intervals during impacts. Constant latency improves reproducibility but not reaction speed.

Future work should predict a distribution over landing/heading and abstain near uncertainty, add side/rear sensing for tilted transitions, and measure impact/thermal load. Comparison with a skill-mixture policy at equal parameters would clarify whether one continuous policy genuinely improves transitions. A planner is still needed to decide whether a jump is necessary or safe in an open environment.

## FastStair Learning to Run Up Stairs with Humanoid Robots

### Safety first, agility second

FastStair explicitly separates safe-contact acquisition from high-speed optimization. A GPU-parallel Divergent Component of Motion (DCM) foothold search first supplies dynamically feasible tread centers and swing trajectories. A PPO policy learns to track them strongly. Once the policy has a safe prior, reward weights shift toward speed and two specialists are fine-tuned for low and high command ranges. LoRA then integrates them to reduce the discontinuity created by switching.

### Model-assisted training

The DCM model is a variable-height inverted pendulum. Candidate footholds come from a `1.8×1.2 m` elevation map; a local steepness score rejects poor patches. Each candidate’s DCM offset and nominal commanded step are evaluated through tensorized discrete search, taking roughly 4 ms for 4,096 environments—about 25× faster than the paper’s compared parallel MPC setup. Bezier-style swing targets connect liftoff, a clearance apex, and chosen landing. PPO receives an exponential foot-target reward.

Actor observations are planar velocity/yaw command, base angular velocity, projected gravity, all joint positions/velocities, prior action, sine/cosine gait clock, and terrain scan dots. The critic additionally sees base linear velocity/height, forces, accelerations, torques, and the planned foothold. All 31 Oli joints are controlled. A RealSense D435i reconstructs a 5 cm elevation grid; policy runs at 100 Hz.

### Three policy regimes

Pretraining emphasizes foothold accuracy. Post-training lowers foothold weight and increases speed tracking, producing experts for `[-0.3,0.8] m/s` and `[0.8,1.6] m/s`. A command threshold at 0.8 m/s identifies the regime; LoRA (`α=16`, rank 8) fine-tunes merged parameters across the full range. This decomposition prevents one policy from collapsing toward moderate speed and avoids raw expert-switch action jumps.

### Evidence and judgment

The final policy climbs varied stairs on the 55 kg, 1.65 m LimX Oli, tracks full-range speeds, and outperforms planner-guided pretraining alone, AMP, and monolithic velocity training. The contribution is a useful curriculum of *model-based safety → specialized agility → unified transition*.

Yet the final system still contains rule-based speed switching inside a supposedly unified network, and LoRA smoothness is empirical rather than guaranteed. DCM guidance simplifies whole-body dynamics and assumes stair geometry is measured accurately. Running at 1.6 m/s on steps raises impact and thermal risks not captured by success alone. Future evaluation should report center-of-pressure margin, peak force/torque, temperature, and failure severity. A continuous learned gate with safety constraints could replace the 0.8 m/s threshold, while uncertainty in stair geometry should widen foothold margins or slow commands automatically.

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

### Why voxels matter

Gallant targets ceilings, doors, lateral clutter, platforms, stairs, piles, and vertical drops in one local-navigation policy. Elevation maps collapse multiple surfaces and cannot represent an overhead slab. Raw 3-D point clouds are irregular and expensive. Gallant voxelizes LiDAR into a fixed torso-centric `32×32×40` binary grid at 5 cm resolution, preserving full vertical structure while remaining small enough for dense convolution.

### Policy architecture and observations

The actor sees a relative target/time command; six-frame histories of base angular velocity, gravity, joint positions/velocities, and actions; and the current voxel grid. The critic additionally gets base linear velocity and a privileged height map. Non-voxel input passes through a two-layer 256-D Mish/LayerNorm MLP. A three-layer 2-D CNN treats the 40 vertical bins as channels, yielding a 64-D voxel feature. Fusion through another MLP produces a 256-D latent; the actor outputs 29 joint targets and the critic one value.

Treating height as channels is the architectural highlight. Compared with full 3-D CNNs it reduces memory/compute while 2-D kernels capture horizontal layout and channel mixing captures vertical occupancy. Sparse convolutions add little because the grid is relatively dense in `(x,y)` at this scale.

### LiDAR simulation and deployment

Warp-based ray casting includes static terrain and the robot’s moving links. Self-scans and corresponding occlusion holes are essential; omitting them produces unrealistically clear floor observations. Simulation randomizes LiDAR pose, range hit, 100–200 ms latency, and 2% missing voxels. Eight terrain curricula progressively tighten clearances and remove temporary support planes.

Training uses PPO across `1024×8` environments. Voxel sensing runs at 10 Hz, policy at 50 Hz, and hardware PD at 500 Hz. Two Hesai JT128 units provide near-spherical training/deployment coverage; the full G1 deployment uses onboard Orin and OctoMap preprocessing. A head Mid-360/Fast-LIO2 supplies local target position at 25 Hz.

### Evidence and critical assessment

Ablations show the z-grouped CNN is generally faster and more successful than 3-D/sparse alternatives. Modeling self-scan and latency is necessary for hardware. One policy crouches under ceilings, passes doors, climbs stairs/platforms, and navigates piles; height-map baselines fail on overhead/lateral geometry.

Gallant’s contribution is a practical representational sweet spot, not a new control algorithm. Binary occupancy discards surface normals, confidence, age, and material/support semantics. A 5 cm cell balances FoV and precision, but is coarse for narrow footholds; 2.5 cm shrinks FoV too far. Ten-Hz LiDAR with 100–200 ms delay is slow around dynamic obstacles. Future grids should include uncertainty, motion, and traversability channels; multiresolution voxels could preserve foot precision and torso FoV. A global planner and explicit safety fallback remain necessary beyond the local 10-second goal-reaching task.

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

## GRAIL: Generating Humanoid Loco-Manipulation from 3D Assets and Video Priors

### Why GRAIL appears in locomotion

GRAIL is a data-generation system spanning manipulation and scene locomotion. For terrain/chair/stair behavior, it begins with known 3-D assets, camera calibration, and metric background depth, asks a video generator to animate a robot-proportioned human, reconstructs SMPL-X/object motion, and jointly optimizes pose, depth, contact, and temporal consistency. Starting inside known geometry removes much of the metric ambiguity of arbitrary internet video while retaining a generative video model’s motion diversity.

### Locomotion controller path

Scene-interaction skills use a pretrained FSQ whole-body motion controller augmented with a local height-map CNN. Reconstructed references supervise root/body pose, orientation, and velocity; reference-state initialization and mixed original flat-ground data preserve base locomotion while fine-tuning terrain interaction. An auxiliary kinematic decoder reconstructs targets from the latent and regularizes it. For object tasks, a separate adaptor consumes object pose, shape/contact features, and adds a 64-D latent residual plus binary hands; that branch should not be confused with the terrain controller.

Privileged state/reference policies generate robust behaviors first. Egocentric visual students are then distilled for tasks including stair climbing and pickup. Hardware uses a head OAK-D W; a visual policy produces SONIC latent tokens around 10 Hz and the pretrained decoder supplies motor actions. The paper reports about 90% physical stair success and large-scale generation of more than 20,000 retained sequences.

### Distinguishing contribution and critique

GRAIL’s key move is to use generated video as a *motion proposal inside a controlled 3-D world*. This differs from MeshMimic, which reconstructs a real scene/video, and HumanoidMimicGen, which transforms successful simulator demonstrations. It can propose unusual scene contacts without requiring a mocap clip, while asset geometry anchors scale.

The pipeline filters failed generations/reconstructions, so effective data diversity depends on acceptance rate. A height map cannot represent every 3-D contact used in generated video; the visual student also loses privileged geometry/contact available to the tracker. The binary-hand abstraction limits climbing styles that need grasp adjustment. Future work should report failures at generation, reconstruction, physical tracking, and visual distillation separately; use hardware failures to reweight prompts; and attach confidence to every reference. A voxel/contact-aware scene controller and tactile student would extend the locomotion side beyond stairs and height-field terrain.

## GuideWalk: Learning Unified Autonomous Navigation and Locomotion for Humanoid Robots across Versatile Terrains

### Composite teacher and unified student

GuideWalk removes the runtime hierarchy between local navigation and humanoid locomotion by distilling both into one 26-D whole-body policy. Its composite teacher remains hierarchical: a Dynamic Window Approach (DWA) samples short-horizon linear/angular velocities using goal and obstacle geometry; a privileged AMP locomotion teacher converts the chosen command plus terrain/state into stable joint actions. DAgger labels states actually visited by the student, teaching a direct mapping from perception/goal to motors.

Stage 2 then fine-tunes the student with PPO while retaining an online behavior-cloning loss to the composite teacher. Rewards include DWA command tracking, goal distance/terminal heading, collision avoidance, posture, action rate/smoothness, progress/agility, and stall penalty. The BC term prevents catastrophic loss of natural/stable teacher gait, while PPO can discover lateral body shifts or arm retraction that DWA plus the locomotion teacher never explicitly produced.

### Observations and network

The teacher uses clean base velocities, angular velocity, projected gravity, joints, previous action, terrain elevation, contacts/dynamics, motor delay, and DWA command. Its map CNN feeds an MLP `[512,256,128]`. The student receives the same deployable proprioception plus relative target, an elevation map, and an `18×32` forward depth image. Separate CNNs encode elevation and depth; their features plus proprioception/goal feed a shared MLP yielding 26 joint actions.

Hardware runs the policy at 50 Hz and PD at 1 kHz. A torso Livox MID360 supplies elevation mapping and an Orbbec Gemini 335Lg supplies depth (downsampled to `32×18`) for obstacle structure. All inference and mapping are onboard.

### Evidence and judgment

Ablations show navigation guidance is needed for efficient goal progress, the locomotion teacher for stability, DAgger for fast convergence, and PPO for collision-aware improvement beyond imitation. Stage-1-only students imitate but lack penalties for new collision configurations; PPO plus BC performs best across terrain/clutter tests.

GuideWalk’s strength is that the final motor policy can coordinate navigation and body shape without command-tracking lag between modules. The cost is lost modular guarantees: the student can no longer expose a planned velocity trajectory or be checked easily by DWA at runtime. It also uses both LiDAR elevation and depth, adding sensor/computation redundancy, and remains local rather than semantic/global.

Future work should distill DWA cost or safety margins as explicit outputs, allowing runtime monitoring. Reconstructing elevation from depth would simplify hardware, as the paper notes, but should preserve uncertainty. A global route planner is still needed for large scenes. Comparing unified and hierarchical stacks under identical latency and perception would determine when removal of interface lag outweighs loss of planner transparency.

## High-speed control and navigation for quadrupedal robots on complex and discrete terrain (2025.06)(Science Robotics 2025)

### Problem and central design

This Science Robotics work asks how a quadruped can move quickly when the ground is not merely rough but *discrete*: stepping stones, gaps, stairs, isolated pads, and large height changes create situations in which a slightly misplaced foot means failure. The system separates the problem into a foothold planner and an RL tracker. The planner searches over feasible future contacts on a terrain map; the tracker converts the next front- and rear-foot targets into high-frequency joint commands. This division is important. It keeps geometric route choice explicit while allowing a learned controller to handle impacts, coupled dynamics, and timing that are difficult to model accurately.

### Planner, learned controller, and signals

The planner uses a rapidly explored random tree–style search over foothold candidates, but its sampling and cost are adapted to legged contacts. A boundary-estimation network identifies usable support regions from terrain height, and the search chooses a low-cost sequence subject to reachability and collision constraints. The paper reports that planning is substantially faster than the interval at which the tracker requests new targets, leaving useful online replanning margin rather than producing a one-shot offline route.

The tracker is trained with PPO. Its 167-dimensional actor observation combines current proprioception, short proprioceptive history at 10, 20, and 30 ms, future foothold targets, and a learned estimate of base linear velocity. A GRU state estimator receives current proprioception and the action from 10 ms earlier; its 128-dimensional recurrent state feeds a `[64,16]` MLP that predicts velocity. The actor and critic use `[512,128]` MLPs, and the actor outputs 12 joint-position targets for the quadruped. A separate four-output contact estimator predicts each foot's contact state. Control runs at 100 Hz, so this is a genuinely high-bandwidth target tracker, not a slow planner directly issuing motor torques.

Rewards combine target placement, physical constraints, and motion style. Particularly useful is the distinction between accurate touchdown and merely passing near a target: the former supplies a precise learning signal for sparse supports. The terrain generator is also trained adversarially. A CVAE represents stepping-stone parameters, while a staged curriculum broadens their radius, orientation, spacing, and yaw as tracking improves. This generates meaningful failures instead of wasting simulation on either trivial or impossible maps.

### Achievement and evidence

The robot negotiates stairs, stepping stones, gaps, and parkour-like sequences, including a reported 0.6 m ascent/descent obstacle and randomized pad arrangements. The contribution is not one unprecedented maneuver; it is sustaining speed while repeatedly solving discrete contact decisions. The explicit planner also makes user preference legible: changing its cost can trade path length, risk, or maneuver type without retraining the motor policy.

### Assessment and next steps

The modularity is a highlight. Compared with end-to-end pixel-to-joint policies, a visible contact plan is easier to inspect and replan. Compared with pure model-based control, the learned tracker tolerates modeling error and hard impacts. The price is dependence on a 2.5-D map and a prescribed notion of traversable support. Overhangs, moving objects, deformable ground, poor state estimation, and objects outside the camera view remain explicit limitations. The tests also treat non-support ground as strongly hazardous, which sharpens the benchmark but is more binary than many real scenes.

A valuable continuation would attach calibrated uncertainty and capture probability to every candidate foothold, then optimize expected risk rather than geometric cost alone. Multi-layer or voxel maps would cover overhangs, while force/impact-aware costs could distinguish a reachable step from a hardware-safe one. An emergency reactive policy should also be evaluated when the map changes after a foot sequence has been committed.

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

## Humanoid Parkour Learning

### Aim and unusual training choice

This work learns one vision-based whole-body policy for multiple humanoid parkour behaviors—walking, stairs, jumping onto platforms, leaping gaps, and robust outdoor travel—without relying on a library of motion-capture references. The central claim is that carefully shaped task geometry and staged distillation are sufficient to discover agile motion. In particular, fractal-noise terrain during initial locomotion training teaches foot clearance more generally than a hand-written “feet air time” term.

### Three-stage pipeline

First, a command-conditioned walking policy is trained on flat and fractal terrain. Second, an oracle parkour policy receives exact simulated terrain samples and is optimized with PPO across ten terrain/skill families. Third, the privileged terrain encoder is replaced by a depth CNN and the student is distilled with DAgger. The student is initialized from the oracle’s recurrent state estimator and actor rather than from scratch; multiple GPU collector processes roll out the student, query the oracle on those visited states, and train with an L1 action loss. This matters because humanoid failures quickly move the student outside an expert-only demonstration distribution.

The oracle samples an `11 x 19` scandot grid, embeds it with an MLP into a 32-dimensional terrain code, and combines that with a GRU–MLP state estimator. Its proprioceptive input contains body roll/pitch, angular velocity, joint positions/velocities, and the previous 19-dimensional action. The student substitutes a CNN operating on `48 x 64` depth images but preserves the recurrent control backbone and learned velocity estimator. An Intel RealSense D435i on the robot supplies depth; simulation applies clipping, range-dependent Gaussian noise, holes/artifacts, and spatial/temporal filters chosen to resemble the device.

Rewards divide into task tracking, regularization, and safety. Besides command tracking and progress, they penalize torque, acceleration, excessive joint velocity, contact misuse, and unnecessary arm motion. Terrain-specific terms reward accurate takeoff/touchdown and penalize penetration near platform edges. A yaw command remains active throughout, allowing the robot to turn rather than executing only fixed straight-line obstacle clips.

### Results and what is novel

The reported G1 system jumps onto a 0.42 m platform and across a 0.8 m gap, handles stairs and outdoor ground, and operates onboard. The key contribution is the combination of reference-free multi-skill oracle learning and scalable vision distillation. Ablations indicate that a randomly initialized student struggles to balance and that single-GPU collection covers far fewer transitions in the same wall-clock time; here compute scale directly addresses covariate shift rather than merely shortening training.

### Critical assessment

The design is admirably simple at deployment—one depth camera and a recurrent joint policy—but expensive during training and still reactive. “No motion prior” removes data dependence but shifts responsibility to reward engineering; platform/footstep rewards encode substantial task knowledge. The front-view depth image cannot resolve hidden landing surfaces, and joint-position commands do not impose impact-force guarantees. The most useful extension would learn a predictive contact/risk head, combine it with landing uncertainty, and permit abort/recovery behaviors. Testing different camera poses and unseen obstacle sequences would show whether the network learned terrain reasoning or the visual signatures of the procedural generator.

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

## Learning Agile Locomotion on Risky Terrains

### Sparse reward is the main obstacle

This paper studies fast quadruped traversal on stepping stones, balance beams, gaps, and disorderly supports where ordinary terrain curricula fail. A hard terrain level creates almost no successful trajectories, so PPO sees little signal; a smooth level curriculum can also stall at a sudden difficulty jump. The proposed solution is not a new perception network but an exploration and transfer recipe: train a reusable generalist, fine-tune specialists, add curiosity, and exploit the robot’s front–back symmetry.

### Policy and task formulation

The task is expressed as local goal navigation rather than strict velocity following. Proprioception, terrain measurements, and the relative target feed a PPO actor at 50 Hz; the action is a vector of 12 desired joint positions. By rewarding arrival and direction while allowing the controller to modulate instantaneous speed and gait, the robot can pause, shorten a step, or accelerate when support geometry requires it. Real experiments use motion capture for state estimation, so the paper isolates agile control and learning rather than claiming fully onboard autonomy.

A generalist is first trained on relatively accessible stepping-stone distributions. Its actor and critic initialize terrain-specific specialists, greatly increasing useful experience on sparse hard tasks. The curriculum can jump to random difficulty after mastering its top level, preventing over-concentration on a narrow boundary. Intrinsic curiosity is computed from prediction error: a learned forward model has high error for unfamiliar state–action transitions, encouraging exploration, and its bonus decays once those transitions become familiar. Front–back mirrored state/action pairs share values, advantages, and returns, explicitly enforcing morphology symmetry and doubling useful supervision.

Rewards include goal progress/arrival, movement in the desired direction, “do not wait,” and task-specific balance or standing terms, with standard torque, smoothness, contact, and posture constraints. The same core reward is reused across terrain families, while specialists add only limited task structure and randomization.

### Evidence and contribution

The trained ANYmal reaches speeds of at least 2.5 m/s on risky terrain and learns behaviors such as rapid stone-to-stone travel and beam traversal. Ablations attribute gains to all three additions: transfer provides an initial viable gait, curiosity discovers rare successful contacts, and symmetry improves data efficiency and action consistency. This is a valuable reminder that dramatic agility can come from improving the *learning distribution*, not merely enlarging the actor.

### Critical assessment

The word “risk” here means terrain with a high empirical probability/cost of falling; the policy does not output calibrated risk and PPO does not enforce a formal chance constraint. Curiosity may reward dangerous novelty, and a forward-model error conflates useful unexplored behavior with stochastic contacts. Specialist fine-tuning also yields multiple policies rather than one broadly deployable controller.

Next work should learn an ensemble-based uncertainty estimate and constrain catastrophic-state probability separately from exploration reward. A gating or skill-composition model could consolidate specialists without losing their rare-contact expertise. Finally, repeating tests with onboard visual/inertial localization, perception noise, and hardware stress measurements would establish whether the learning strategy survives the full autonomy stack.

## Learning Autonomous and Safe Quadruped Traversal of Complex Terrains Using Multi-Layer Elevation Maps

### The representation is the main idea

A conventional elevation map stores one height per horizontal cell. It cannot represent both the floor and a table top, the lower and upper surfaces of a bridge, or the clearance under an overhang. This work introduces a multi-layer elevation map for autonomous quadruped navigation: each cell can hold several non-intersecting vertical surface intervals. It retains the efficient grid/CNN structure of a height map while representing constrained 3-D scenes that would otherwise require a full voxel volume.

### Hierarchical learned system

The system contains a terrain-aware locomotion policy and an outer local-navigation policy. Several locomotion specialists are first trained for different skills, then jointly distilled into a generalist. The generalist observes commanded body velocity, joint positions and velocities, previous 12-dimensional action, proprioception, and the multi-layer map and emits 12 joint-position targets. During distillation, extra terrain clutter is shown only to the student; this broadens the observation distribution without forcing each expert into unfamiliar situations.

For navigation, map layers are processed by convolutional encoders. A terrain compressor learns to reproduce layer values with L1 loss, reporting approximately 3.0 cm training and 4.9 cm test MAE. The local-navigation policy receives its goal and map features and generates motion through the locomotion controller. Variable-length geometric elements are handled with a Transformer encoder, after which fused features pass through output MLPs for actions or value. This hybrid CNN/Transformer design reflects the data: regular grids benefit from convolution, while a varying collection of surfaces requires set-like attention.

Reward shaping encourages goal progress, collision avoidance, and maneuverability, supplemented by terrain curricula and symmetry augmentation. The claimed safety is thus learned/empirical—policies receive strong collision and failure penalties and are tested over randomized scenes—rather than a formally verified barrier or reachability guarantee.

### What it solves better than single-layer maps

The robot can choose paths through environments containing overhead obstacles, narrow passages, raised platforms, and terrain with multiple surfaces at the same `(x,y)` location. The representation is the lasting contribution: it offers much of a voxel grid’s expressiveness at lower cost and exposes geometry in a form compatible with existing elevation-map pipelines.

### Judgment and extensions

Separating locomotion and navigation is sensible for long-range autonomy and makes each policy’s role understandable. Multi-expert distillation also supplies diverse low-level behavior without runtime switching. However, compression error is most consequential exactly at low-clearance boundaries; a mean absolute height error does not measure collision risk. Fixed layer capacity and grid resolution can lose thin objects, and dynamic obstacles are not naturally represented. The navigation policy may exploit systematic map artifacts learned in simulation.

Future work should store per-layer confidence and time, evaluate false-free-space rates rather than only MAE, and incorporate moving-object velocity. A geometric safety filter could reject commands whose swept body volume intersects any plausible layer. Comparisons at equal memory and latency against sparse voxels, signed-distance fields, and point transformers would clarify when the proposed representation is truly the best trade-off.

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

## Learning Robust Autonomous Navigation and Locomotion for Wheeled-Legged Robots

### Why wheels and legs must be planned together

This work develops an autonomous stack for a wheeled-legged ANYmal variant. Rolling is faster and cheaper than stepping on smooth ground, but wheels cannot simply follow a conventional planar path when stairs, curbs, rubble, people, and narrow passages appear. The system therefore uses hierarchical RL: a high-level controller (HLC) decides local velocity and navigation behavior, while a recurrent low-level controller (LLC) decides wheel speeds, joint targets, gait, and transitions between driving and walking.

### Rates, observations, and learned hierarchy

The HLC runs at 10 Hz, aligned with the local elevation-map update, and emits base-velocity targets. The LLC runs at 50 Hz and outputs joint positions plus wheel velocities. Rather than giving the HLC raw proprioception, the authors expose the recurrent LLC belief state, which already encodes terrain properties and disturbances useful for deciding whether to drive or step. The HLC also sees two upcoming waypoints, the local height map with a safety margin, and 20 previously visited positions spaced at 0.5 m—about 10 m of route memory. That memory discourages oscillation and repeated exploration when local geometry is ambiguous.

Both levels use PPO and asymmetric privileged training. The LLC builds on robust perceptive locomotion but removes a hand-designed CPG, allowing the network itself to blend rolling and stepping. Navigation worlds are procedurally assembled with Wave Function Collapse from compatible terrain tiles; random paths, stairs, obstacles, narrow corridors, and dynamic agents give the HLC structured but diverse experience.

### Full autonomy and results

At deployment, multiple LiDARs construct elevation maps and localize the robot against a pre-scanned point cloud; a front stereo/RGB camera detects people, whose regions are inflated in the map for safety. A global shortest-path search supplies waypoints, while the HLC adapts locally. In a city-scale mock delivery workflow, remote goals are sent over 5G and the robot navigates grass, gravel, stairs, ramps, and pedestrians. The paper reports about three times the speed and 53% lower mechanical cost of transport than its conventional ANYmal comparison; wheels consume roughly 1.2 times the mechanical power while producing 3.4 times the speed.

### Assessment

The highlight is the interface between levels: passing the LLC belief gives navigation direct knowledge of what the body can currently handle, and route memory reduces zigzag behavior. Yet the system is not map-free—the global route depends on a manually prepared scan and graph, and an unexpected tall-grass event required manual global replanning. Human avoidance is map inflation rather than intent prediction.

Future work should train the hierarchy jointly with an explicit switching/energy cost, replace the pre-scan with online semantic SLAM, and represent moving obstacles with trajectories rather than static height offsets. Mode-transition counts, near-collisions, intervention rate, and end-to-end latency would measure safe autonomy better than speed alone.

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

## Locomotion Beyond Feet

### Whole-body contact as locomotion

Locomotion Beyond Feet expands the vocabulary from walking to nine chained skills: get down, crawl under a chair, get up, climb over a wall, climb onto and down from a platform, walk, and ascend/descend steep stairs. Hands, knees, elbows, and torso can support the body. The work argues that one monolithic policy is undesirable because these behaviors have fundamentally different contact structures.

### Keyframes, robust trackers, and visual planner

Each skill begins as sparse embodiment-specific keyframe animation. Unlike human mocap, poses are authored directly in robot joint space, avoiding retargeting error and allowing nonhuman strategies. Open-loop simulation/hardware playback verifies geometry, then PPO motion tracking converts the kinematic sequence into a robust controller. Terrain parameters and dynamics are randomized around each skill.

Runtime is hierarchical. Terrain-specific tracking policies execute skills; failure/recovery logic handles bad initializations and transitions; a vision planner classifies the obstacle and estimates its dimensions, then selects and parameterizes a skill. This lets a crawler specialize in low clearance and a stair controller specialize in foot contacts, with explicit transition points at which stability can be checked.

### Achievement and distinctive value

Real humanoid sequences contain chairs, knee-high walls/platforms, and steep stairs, with generalization across object instances, sizes, and ordering. The standout idea is using keyframes as a compact language for human contact knowledge. A few support configurations can be cheaper and more controllable than collecting/retargeting mocap or hoping sparse reward discovers hand and knee use.

### Judgment and future work

The hierarchy is inspectable and extensible, but the skill set is manually enumerated and transitions are brittle. Classification error can select an incompatible controller; geometry may change mid-skill; hard contacts raise hardware-load and surface-friction concerns. Keyframes also inject designer bias even if they reduce reward shaping.

Next work should learn a common contact-conditioned policy or compositional latent while retaining skill confidence and recovery. Planning should reason over contact affordances and load limits, not object classes alone. Peak forces, joint temperature, surface damage, and recovery success would make evaluation match the safety implications of using the whole body.

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

## Mind Your Steps: A General Learning Framework for Accurate Humanoid Foothold Tracking

### A reusable footstep interface

Mind Your Steps argues that velocity commands cannot guarantee where a foot lands, which is unacceptable near people, narrow supports, or manipulation poses. It trains a standalone model-free foothold tracker below many possible planners. The policy uses only proprioception, gait phase, previous action, and a 3-D target expressed relative to the stance foot; it outputs joint-position targets for PD control and requires no terrain map.

### Why stance-relative commands matter

A world- or base-frame target moves when pose estimation drifts or the torso oscillates. Once the stance foot is planted, expressing the next foothold in its frame makes the command nearly constant during swing and reduces global-odometry dependence. The target can also be latched at a stable support transition, improving robustness to contact-timing error.

During training, a procedural sampler generates reachable 3-D steps and dynamically provides support beneath the requested contact. The controller thus learns tracking without coupling to one staircase or planner. PPO rewards touchdown position/orientation, timing, balance, posture, smoothness, and effort. Curricula expand displacement and height as accuracy improves; dynamics and observation randomization support transfer.

### Results and modularity

The same tracker is paired with multiple upstream generators for target reaching, clutter avoidance, rough terrain, and loco-manipulation-style positioning, with accurate real humanoid steps. The significant contribution is architectural: a clean discrete-contact API lets a planner reason in feet while low-level RL handles full dynamics.

### Critical assessment

Terrain agnosticism makes feasibility entirely the planner’s responsibility. The controller cannot know whether support exists, is large enough, or is slippery. Stance-relative coordinates reduce but do not remove contact-estimation error; a false stance switch corrupts the frame. Single-step accuracy also does not ensure long-horizon viability.

The best extension is to expose learned reachability/value and uncertainty to planners, plus a contact-confidence signal. Training on recoverable planner mistakes would improve robustness. Combining this interface with map-based generation and a capture-region safety filter could preserve modularity while adding guarantees.

## MoRE: Mixture of Residual Experts for Humanoid Lifelike Gaits Learning on Complex Terrains

### Robustness first, style second

MoRE separates two objectives that often interfere: traverse difficult terrain and look anthropomorphic. Stage one trains a depth-conditioned PPO base policy only with locomotion rewards. Stage two inserts a gait-conditioned mixture of latent residual experts and multiple adversarial discriminators, adding style without relearning the entire terrain controller.

### Inputs and architecture

The actor receives commanded linear/yaw velocity, joint position/velocity, last action, two consecutive head-camera depth images, proprioceptive history, and a one-hot gait command. A 2-D CNN encodes `64 x 64` depth at 10 Hz; a 1-D CNN encodes history; MLPs fuse features and predict action. The policy runs at 50 Hz and outputs 16 joint targets. The critic instead receives privileged elevation/state information.

Residual experts do not produce separate actions. Each MLP outputs a latent correction to the base actor’s last hidden representation; a gating MLP weights these corrections. This preserves base robustness and lowers parameters. A gait-specific AMP discriminator—walk, run, crouch-walk, and related styles—compares robot transitions with the appropriate reference and supplies style reward.

Curricula span gaps around 0.05–0.45 m, steps 0.05–0.30 m, and stairs to about 0.15 m. Depth noise, filtering, camera pose, deviation to 15 cm, and delay are randomized. The residual/style stage runs about 20,000 iterations on four RTX 4090s.

### Contribution and judgment

The latent-residual design creates specialization without hard switching among policies and reduces gradient conflict between gait styles. Explicit commands make behavior controllable. However, naturalness is judged by discriminators, not biomechanics or human preference, and style can oppose a safe reaction. Soft gates may collapse or blend incoherently; one-hot commands encourage abrupt transitions.

Future work should report expert usage/load balance, learn continuous style coordinates, and condition discriminators on terrain so adaptation is not penalized. Force, impact, and energy comparisons would test whether lifelike appearance improves mechanics. Attenuating residuals under perceptual uncertainty could protect the robust base.

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

### Direct raw-LiDAR reactive control

Omni-Perception targets collision avoidance in all directions and at different heights: ground clutter, slender objects, transparent surfaces, overhead obstacles, and moving agents can defeat front depth cameras or 2.5-D maps. Its RL policy directly consumes temporal raw LiDAR point clouds rather than first constructing an elevation/occupancy map.

### PD-RiskNet and policy inputs

The full observation contains a history of joint position/velocity, base linear/angular velocity and projected gravity; a matching history of 3-D point clouds; and commanded linear/yaw velocity. PD-RiskNet (“Proximal–Distal Risk-Aware Network”) partitions perception by control urgency. Near points receive fine, high-priority processing for imminent contact, while farther structure supplies coarser anticipation. Hierarchical spatial features are fused across time and with proprioception before the actor emits joint-position targets.

The policy is trained end-to-end with PPO for velocity tracking and collision avoidance. Rewards cover command following, clearance, collision, gait stability, smoothness, torque, and posture. A custom GPU raycaster supports Isaac Gym, Genesis, and MuJoCo and simulates range noise, dropout, beam/angular effects, latency, and moving objects. This simulator is a substantial secondary contribution because raw scan policies need millions of consistent temporal clouds.

### Results and useful distinction

Real tests show a legged robot responding to obstacles approaching from the side/rear and negotiating aerial, transparent, slender, and ground-level structure. LiDAR is largely lighting invariant and directly measures 3-D points, so the approach covers failures that active stereo and elevation maps systematically suffer.

The strongest idea is proximity-dependent representation: control does not need uniform reconstruction quality everywhere. Immediate geometry deserves resolution and fast reaction; distant geometry mainly informs trend. Direct processing also avoids mapping latency and drift.

### Limitations and extensions

Reactive avoidance is not predictive social navigation. A short point-cloud history estimates motion only implicitly, can confuse self-motion with object motion, and may deadlock in crowds or narrow passages. LiDAR has its own failures—minimum range, sparse returns from thin/dark objects, motion distortion, self-occlusion—and the policy has no formal clearance guarantee.

Future work should explicitly estimate scene flow and collision time, couple the local controller to a global planner, and expose calibrated risk to a safety layer. Ablations at equal point count/latency against voxels and range images would establish whether the raw representation itself or simply omnidirectional sensing drives the gain.

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

## Parkour in the Wild Learning a General and Extensible Agile Locomotion Policy Using Multi-expert Distillation and RL Fine-tuning

### Build a foundation policy from specialists

This ANYmal-D work asks how to consolidate terrain-specific agility without sacrificing the specialists’ competence or freezing the resulting student at their limits. Experts are first trained separately on terrain families. DAgger distills them into one depth-conditioned foundation policy, and PPO then fine-tunes that policy on a broader mix including real 3-D scans. New terrain can be added by repeating adaptation rather than rebuilding the whole library.

### Networks, inputs, and learning stages

Each expert uses privileged terrain geometry and proprioception to output 12 joint-position targets. The student retains the recurrent proprioceptive controller but replaces privileged perception with a CNN over egocentric depth. During DAgger the student visits states and the appropriate expert labels actions, preventing expert-only distribution shift. The important final stage resumes PPO with task reward: it repairs action averaging, discovers transitions between regimes, and can surpass the frozen teachers on mixed terrain.

Depth and robot dynamics are randomized for transfer; real scanned terrain exposes irregular geometry absent from simple procedural blocks. Commands specify desired motion rather than named skills, so running, climbing, jumping, and careful stepping emerge according to geometry. Deployment uses onboard perception/control on ANYmal D.

### Contribution and evidence

The method frames multi-expert distillation as the beginning of generalization, not the final compression step. Pure imitation compromises when experts disagree and cannot discover a better cross-skill action. RL fine-tuning supplies exactly that missing optimization signal. Experiments show the unified policy traversing diverse indoor/outdoor and scanned terrain and being further extensible.

### Critical assessment

The recipe is powerful but does not eliminate catastrophic forgetting; fine-tuning on a new terrain can still erode rare old skills unless replay/distribution balance is maintained. An expert-selection rule supplies labels during distillation and can be ambiguous on composite terrain. Depth remains local and active sensing can fail outdoors. “Foundation” describes reuse within agile quadruped locomotion, not broad semantic reasoning.

Future work should use explicit continual-learning metrics, retain a balanced replay/evaluation suite, and identify expert disagreement as an uncertainty signal. A modular residual adapter or routing network could add skills with less interference. Long autonomous routes and recovery statistics would test whether isolated terrain competence becomes reliable chaining.

## Perceptive Behavior Foundation Model: Adapting Human Motion Priors to Robot-Centric Terrain

### Keep the motion command, adapt its realization

Perceptive BFM addresses a useful mismatch: a broad human-motion controller can track walking, turning, or expressive commands, but a flat-ground reference does not specify how feet should clear and contact a stair or sparse support. The system preserves the raw kinematic reference as its user-facing command while terrain perception contributes only the local corrections required for feasible execution.

### Terrain-conformal supervision

Offline Terrain-Conformal Reference Synthesis (TCRS) estimates stance/swing intervals, latches support feet to terrain, optimizes a toe/heel-aware mid-foot swing for clearance, reconstructs a support-aware root trajectory, repairs collisions, and applies multi-point Jacobian IK. TCRS is never needed online. It creates terrain-adapted targets from ordinary motion clips so a blind Transformer teacher can learn them with PPO.

The deployed student receives projected gravity, base angular velocity, joints, previous action, a 10-step proprioceptive history, a 21-step future reference window, a 21-step anchor-displacement window, and a `17 x 11` torso-centered height map with validity mask. A Transformer encodes command/history tokens; a terrain encoder processes the map. Zero-initialized intent-gating and action-residual branches inject terrain information. At initialization they output zero, so the network exactly preserves the motion tracker rather than destroying the prior.

Because teacher and student use adapted versus raw reference frames, the student matches the teacher’s effective PD target in a common target frame, not its raw residual. PPO then conservatively fine-tunes the complete 50 Hz policy. Output is a residual joint-position target around the raw reference.

### Why this is a strong design

The identity-gated residual architecture makes “foundation model adaptation” concrete: retain broad behavior, learn a narrow terrain correction, and keep the command interface unchanged. TCRS supplies contact-aware supervision without manually authoring every obstacle clip. Simulated and real experiments cover varied motions and terrain, including operator commands that were not recorded for the encountered geometry.

### Limits and next directions

TCRS itself encodes assumptions—fixed phase, conservative horizontal footholds, height-map geometry—and a flawed synthesized target biases both teacher and student. The term foundation model should be read within motion-conditioned humanoid control; the system is not a language/vision generalist. A `17 x 11` 2.5-D scan cannot represent overhangs or semantics.

Future work should infer contact/phase adjustments rather than preserving them, propagate TCRS uncertainty, and use volumetric or semantic perception. Evaluating genuinely held-out motion–terrain compositions and severe sensor failure would show whether identity gating preserves capability or merely delays forgetting.

## Perceptive Humanoid Parkour: Chaining Dynamic Human Skills via Motion Matching

### Motion matching as the skill planner

This work combines animation-style motion matching with physics-based humanoid control. Rather than train a high-level selector over named behaviors, it builds a kinematic database of locomotion and atomic parkour skills—stepping, climbing, vaulting, and jumping—and repeatedly retrieves a compatible future frame. Playback of the match forms a smooth reference, then learned experts and a unified visual student make it physically executable.

### Kinematic composition and tracking

Each database frame stores pose plus a feature vector containing short-horizon future trajectory and motion/contact descriptors. Given desired velocity and obstacle geometry, nearest-neighbor search selects the frame minimizing feature mismatch. Search occurs periodically or after a significant command change; playback advances from that frame. This supplies explicit transitions without generating a long trajectory autoregressively.

The kinematic stage composes locomotion with atomic clips into long sequences. Because matching ignores robot dynamics, skill-specific PPO trackers use privileged observations: reference joint position/velocity, pelvis pose error, pelvis linear/angular velocity, current joints, previous action, and terrain height. Outputs are whole-body joint targets.

### Visual unification

Experts are consolidated with DAgger plus PPO. The student observes pelvis gravity/angular velocity, joints, previous action, commanded velocity, and onboard depth rendered with NVIDIA Warp. Camera extrinsics are calibrated by matching self-visible image regions in simulation and hardware; artifact randomization bridges transfer. Imitation preserves expert skills, while PPO repairs states and transitions poorly covered by labels.

### Assessment

The robot autonomously selects and chains skills from obstacles instead of requiring manual triggers. Motion matching is more controllable than a latent selector yet more flexible than a state machine. Human clips provide timing while physics tracking absorbs morphology/contact mismatch.

Its ceiling is the library. Retrieval cannot invent a missing contact pattern, and feature weights can choose a visually close but dynamically bad transition. Future work should rank candidates with tracker reachability/value, output retrieval uncertainty, and learn from successful robot rollouts. Recovery and abort clips are as important as successful maneuvers.

## PIE: Parkour with Implicit-Explicit Learning Framework for Legged Robots

### One-stage alternative to teacher distillation

PIE trains a single end-to-end quadruped parkour network rather than first learning a privileged height-map teacher and then imitating it. Noisy, delayed depth and imperfect actuation make explicit estimates unreliable, but an entirely implicit latent is hard to supervise. PIE uses explicit auxiliary predictions and implicit task features at two levels: robot/environment estimation and action-consequence estimation.

### Dual-level estimation and I/O

The policy fuses egocentric depth, proprioceptive history, and command. At the perception level, an encoder explicitly reconstructs useful terrain/state quantities while its latent may retain control information a fixed map loses. It predicts successor proprioception, forcing visual/body features to explain dynamics rather than merely correlate with action. At the action level, explicit target/state prediction and an implicit latent correction help anticipate how a movement changes the robot.

All components are optimized in one PPO stage. The actor outputs joint-position targets for a low-cost DEEP Robotics Lite3. Privileged information is confined to training targets and the asymmetric critic, not an oracle action policy. Rewards cover command/progress, posture, contact, energy, smoothness, and terrain completion, leaving auxiliary structure to carry much of representation learning.

### Contribution and evidence

The real robot traverses gaps, narrow platforms, stairs, and uneven outdoor terrain with zero-shot simulation transfer. Avoiding a teacher handoff removes imitation information loss and simplifies the pipeline. Successor-state prediction is especially compelling because it ties terrain understanding to likely physical response.

### Critical assessment

One-stage learning couples failures: bad visual features, state prediction, and control gradients can destabilize one another. Auxiliary accuracy does not guarantee safety, and the “explicit” heads are not a planner or constraint. Depth remains FOV/range limited and uncertainty is uncalibrated.

Future work should expose prediction confidence, enforce multi-step consistency, and ablate explicit targets at equal capacity. A safety filter using predicted successor distributions could turn the auxiliary model into a safeguard. Moving obstacles and altered camera calibration would test causal understanding rather than simulation correlation.

## PHUMA:Physically Reliable Humanoid Locomotion Dataset

### Dataset problem rather than controller problem

PHUMA observes that internet-scale recovered human motion is diverse but often unusable for robots: root jitter, floating/penetrating feet, impossible joints, foot skating, and sitting on absent objects poison imitation learning. It contributes a curated, physically retargeted locomotion dataset, not a deployment architecture. MaskedMimic and BeyondMimic controllers test downstream quality.

### Curation pipeline

The source pool is primarily Humanoid-X, augmented with LaFAN1, LocoMuJoCo, and captured clips. Processing splits sequences into clips so one bad interval does not discard an entire video. Filters reject excessive jerk; identify object-dependent instability by center-of-mass distance from the foot support region; estimate a ground plane; and reject inconsistent contact indicating floating or penetration. Chair-sitting clips are removed when the environment contains no chair.

### PhySINK retargeting

Physically constrained Shape-adaptive Inverse Kinematics maps retained motion to each humanoid. Besides keypoint fit, PhySINK uses soft joint limits, foot grounding during inferred contact, anti-sliding consistency, and shape-aware correspondence. These constraints sacrifice some visual match to produce motions a torque-limited robot can track without inventing support.

The release contains 73.0 hours and about 76,000 clips—reported as 3.5 times AMASS and more physically usable than the 69.1 retained hours from Humanoid-X’s 231.4 raw hours. Identical imitation methods trained on PHUMA and alternatives are compared on Unitree G1/H1-2 for unseen-motion survival/tracking and real G1 zero-shot tracking.

### Judgment and future work

The key insight is measuring scale *after* physical validation. Clip-level filtering preserves good fragments and constrained IK targets the exact artifacts destabilizing RL. Yet filtering may remove rare feasible dynamics and bias toward conservative motion. Static-ground assumptions exclude object interaction; feasibility is embodiment/actuator dependent; IK still does not prove dynamic feasibility.

Future releases should include contacts, force/feasibility scores, provenance, and uncertainty; forward-simulate clips before acceptance; and retain scene geometry for supported interactions. Splits should isolate source videos and motion families to prevent near-duplicate leakage.

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

## Robot Parkour Learning

### Diverse skills from a simple objective

Robot Parkour Learning produces climbing, leaping, crawling, squeezing, and running on inexpensive quadrupeds without animal motion data. The core is a direct-collocation-inspired two-stage RL curriculum. Soft collision constraints first let the robot temporarily penetrate geometry so PPO can discover a rough path through a hard obstacle; hard collision simulation then fine-tunes that behavior into a physically valid skill.

### Expert policies and visual student

Each specialist is a GRU receiving 29-dimensional proprioception (roll/pitch, angular velocity, joint positions/velocities), previous 12-dimensional action, privileged terrain geometry, and randomized physics properties. It outputs 12 joint-position targets. A common progress reward avoids per-skill reward engineering, while collision penalties approximate penetration volume/depth from sampled body points. Multiplying penetration cost by forward speed closes the loophole of sprinting through an obstacle before accumulating penalty.

A unified GRU student replaces privileged geometry with a small CNN embedding of egocentric depth and is distilled on its own states from the appropriate expert. Terrain type selects the teacher during training; at runtime the student implicitly recognizes geometry and chooses behavior. Simulated depth receives clipping, Gaussian noise, artifacts, and latency; real depth receives clipping, hole filling, and spatial/temporal filtering. Motor-safety measures limit torque, speed, and abrupt action.

### Results and significance

Two low-cost robots traverse 0.40 m obstacles (about 1.53 body heights), 0.60 m gaps (1.5 body lengths), 0.20 m overhead clearance, and 0.28 m slits narrower than nominal body width. The policy controls every joint from depth and proprioception rather than relying on a scripted high-level selector.

The most reusable idea is soft-to-hard physics as exploration shaping. It supplies a continuous signal when strict collision makes success essentially unobservable, then removes the nonphysical shortcut before deployment.

### Limits and future work

Soft penetration can seed motions that hard physics cannot repair, and teacher selection assumes labeled terrain during distillation. The visual student inherits the experts’ skill set and front-camera blind zones; extreme contacts stress low-cost hardware. A learned safety critic, recovery expert, and uncertainty-aware expert routing would improve reliability. Training on mixed obstacle sequences and reporting motor temperature/impact/failure recovery would test whether individual stunt competence becomes dependable autonomy.

## RPL: Learning Robust Humanoid Perceptive Locomotion on Challenging Terrains

### Multidirectional perception under payloads

RPL targets forward *and backward* humanoid travel on slopes, stairs, and sparse stones while the upper body carries or moves a payload. A single front camera and a monolithic actor are brittle: backward travel lacks look-ahead, arms cause self-occlusion, and payload forces disturb balance. RPL trains terrain experts from privileged maps, then distills them into a multi-camera Transformer student.

### Decoupled expert and unified visual policy

Stage one uses separate lower- and upper-body agents whose actions concatenate for the PD controller. The lower policy sees a `1.6 x 1.0 m` height grid at 0.1 m resolution and learns slopes, up/down stairs, and stepping stones; the upper policy learns manipulation/pose under randomized end-effector force. PPO optimizes independent reward groups so arm objectives do not erase precise feet.

Stage two uses front and rear depth CNNs plus noisy proprioception/task goals. Transformer fusion combines asymmetric views. Depth-feature scaling by velocity command emphasizes the camera facing the travel direction. Random side masking hides variable image widths during training, forcing generalization to stairs/platforms with unseen lateral extent. A custom raycaster includes both terrain and moving robot meshes—important for realistic arm/payload self-occlusion—and is reported five times faster than existing multi-depth rendering, with latency, Gaussian noise, and dropout.

### Evidence and contribution

The G1 performs long-horizon bidirectional traversal with a 2 kg payload on 20° slopes, stairs with 22/25/30 cm tread length, and `25 x 25 cm` stones separated by 60 cm gaps. The key contribution is not merely another depth policy but explicit treatment of directional camera asymmetry and upper/lower-body interference.

### Assessment

Velocity-conditioned feature scaling is simple and physically meaningful, and rendering the robot prevents a major synthetic-vision shortcut. Still, multiple cameras increase calibration, bandwidth, and failure modes; learned feature scaling is not calibrated visibility. Decoupling the body during expert training may miss beneficial whole-body coordination, and the final student has no formal foothold guarantee.

Future work should estimate per-camera reliability, degrade gracefully when one view fails, and compare attention with geometric view fusion. Heavier/asymmetric moving payloads and tasks requiring arm support would probe the decoupling assumption. Onboard latency, power, and long-duration intervention rates should accompany obstacle success.

## SSR: Scaling Surefooted and Symmetric Humanoid Traversal to the Open World

### Three problems solved together

SSR combines safe sole placement, bilateral coordination, and terrain-appropriate motion style in one single-stage depth policy. The actor observes a short history of 72-dimensional proprioception—angular velocity, gravity, command, joints, and prior action—plus a `36 x 36` egocentric depth image, and outputs 21 joint targets. The critic additionally sees true base/foot velocities, contacts, limb positions, and foot/body height maps.

### Architecture and surefooted reward

A CNN encodes depth, an MLP encodes temporal proprioception, and a GRU produces latent context with heads for foot-centric terrain, base-centric terrain, and next-state prediction through a VAE-style estimator. A mixture-of-experts actor turns this estimated state into action.

Imagined foothold guidance evaluates a sole-sized `22.5 x 10 cm` patch sampled every 2.5 cm at candidate swing-foot locations. Unsupported fraction gives a dense pre-contact correction signal instead of waiting for a fall at touchdown. This specifically discourages edge slips and partial-sole landings, a better safety proxy than foot-center distance alone.

### Efficient symmetry and style

Mirroring a visual recurrent rollout is expensive. SSR constructs mirror-equivariant CNN/MLP/GRU layers, encodes the observation once, then mirrors the compact latent, privileged state, and action for augmentation. This preserves exploration flexibility while enforcing left–right data sharing. Terrain-specific discriminators judge five-frame motion histories, avoiding one adversarial prior that calls every terrain-adapted gait unnatural.

Deployment uses a waist RealSense D435i at 60 Hz, cropped/downsampled to `36 x 36`; Jetson AGX Orin inference/action runs at 50 Hz. Tests span stairs, platforms, gaps, and outdoor terrain beyond curriculum difficulty.

### Judgment

SSR’s highlight is aligning every inductive bias with a concrete failure: sole support for edge slips, latent equivariance for expensive symmetry, multiple priors for terrain-style conflict. However, imagined support depends on privileged training height and may not match noisy depth; strict bilateral symmetry is wrong with payloads, damage, or asymmetric terrain. “Open world” remains an empirical range, not open-set detection.

Next work should learn when to break symmetry, estimate foothold confidence from perception, and report failure calibration on unseen materials. Combining sole-support prediction with a runtime contact-safety filter would turn a strong reward into an explicit deployment check.

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

## STATE-NAV: Stability-Aware Traversability Estimation for Bipedal Navigation on Rough Terrain

### Traversability as allowable velocity

STATE-NAV replaces binary geometric traversability with a robot- and command-dependent stability prediction. A slope may be safe slowly but dangerous laterally or at speed. The supervision is body-to-stance-foot angle (BSFA) instability: the per-gait-cycle RMS of the body angle relative to the supporting foot, selected because it correlates strongly with imminent falls within the next two steps.

### TravFormer and uncertainty

TravFormer receives an egocentric elevation-map patch and candidate command velocity. A ResNet-18 extracts spatial features, deformable attention focuses on terrain regions relevant to stability, and a Transformer decoder treats command embeddings as queries over terrain keys/values. It predicts BSFA instability plus aleatoric uncertainty. The Transformer decoder improves instability RMSE by a reported 11.8% over an MLP decoder in the architecture ablation.

Predictions across position/velocity candidates form a stability-aware command-velocity map. TravRRT* uses it for global route search rather than assigning a hand-tuned slope cost. Locally, risk-sensitive MPC chooses speed under uncertainty and an Angular Linear Inverted Pendulum controller turns commands into footsteps/joint control.

### System evidence

Simulation and real Digit-style biped tests compare geometric/manual baselines, showing paths that slow or detour in unstable terrain and accelerate on favorable ground. A chest ZED 2i supplies point cloud and localization; global map/plan update every 5 s and MPC runs at 3 Hz on an external laptop.

### Assessment

The contribution is a meaningful traversability unit: predicted physical instability under a command, not an arbitrary cost. Self-supervised labels avoid hand annotation, and uncertainty explicitly affects planning. Yet BSFA is a proxy, not fall probability, and may miss slip, impact, actuator saturation, or upper-body load. Learned risk is embodiment/controller specific; slow update rates limit dynamic scenes.

Future work should calibrate predictions to failure probability, include multiple instability signals, and adapt online as wear/payload changes. Faster incremental maps and moving-object prediction are needed. A conformal or chance-constrained layer could turn empirical uncertainty into a stated safety level.

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

## T-GMP: Terrain-conditioned Generative Motion Priors for Versatile and Natural Humanoid Locomotion

### A motion prior that knows the terrain

T-GMP argues that an ordinary adversarial motion prior cannot know whether a gait occurs on a beam, stair, gap, or slope. It builds a conditional generative manifold from synchronized motion and local height maps, then uses a terrain-conditioned discriminator to guide a unified RL controller.

### Data and CVAE

The dataset combines privileged policies on difficult terrain with human mocap retargeted by General Motion Retargeting. Each sample contains joint position/velocity, root-relative end effectors, and a matching height map. A two-layer CNN encodes terrain; a conditional beta-VAE reconstructs short state sequences from a latent plus terrain. Although training has height sequences, the decoder uses only the current map to match deployment, with a short horizon limiting accumulation error.

Only about 29.6 minutes/88.8k frames span eight terrain categories. Privileged trajectories contribute physical adaptation; mocap contributes anthropomorphic coordination.

### RL policy and discriminator

The actor receives five frames of base angular velocity, gravity, joints, and previous 26-D action, plus a CNN map embedding, and outputs 26 joint targets. PPO optimizes task and regularization rewards. A five-layer terrain CNN conditions the adversarial discriminator, so style is compared with motion appropriate to the same geometry rather than a pooled flat-ground set.

### Judgment

Conditioning both generation and judgment on terrain avoids penalizing necessary crouches, arm balance, or altered steps as “unnatural.” But paired data are difficult: privileged experts are feasible yet less humanlike, while mocap is natural but geometrically narrow. A CVAE may average contacts, LiDAR maps are slow/noisy, and discriminator reward is not force feasibility.

Future work should use discrete/diffusion contact representations, propagate map uncertainty, and filter generated motion with tracker value. Evaluation should separate visual naturalness, energy, impact, and traversal success.

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

## TTT-Parkour: Rapid Test-Time Training for Perceptive Robot Parkour

### Adapt to the obstacle rather than pretrain every shape

TTT-Parkour accepts that procedural training cannot cover every wedge, stake, trapezoid, box, or narrow beam. A general humanoid depth policy is pretrained on synthetic platforms, then a novel obstacle is captured with RGB-D, converted into a simulation mesh, and used for rapid PPO fine-tuning. The adapted network transfers back without practicing failures on hardware.

### Policy and reconstruction

The actor uses a CNN depth encoder concatenated with projected gravity, angular velocity, command, joints, and previous action; an MLP outputs joint targets. Training combines progress, regularization, safety, and AMP under terrain/dynamics/depth randomization. Touching surrounding ground is failure, preventing bypass of sparse platforms.

A feed-forward RGB-D reconstruction avoids slow per-scene NeRF/3DGS optimization. Automatic metric-scale recovery and frame alignment produce a collision mesh, which is randomized during fine-tuning so the controller does not exploit one artifact. Capture, reconstruction, training, and deployment finish within ten minutes for most tests.

### Contribution and assessment

Hardware results turn base-policy failures into successful traversal. The important shift is simulation as a rapid *deployment-time practice environment*: safer than real trial-and-error and more targeted than generic generators.

This is not adaptation while moving; an operator scans and waits, and the checkpoint is specialized. Thin-edge/occlusion reconstruction errors become collision errors, full-weight fine-tuning can forget recovery, and moving terrain invalidates the mesh.

Future work should replay base terrain, use lightweight adapters, propagate mesh uncertainty into geometry randomization, and quantify retained general skill. A planner could decide when TTT is necessary and select informative scan viewpoints. Tests should deliberately alter the obstacle after capture.

## UniReLo: Learning a Unified Humanoid Policy from Fall Recovery to Locomotion across Diverse Terrains

### One controller across upright, fallen, and recovering states

UniReLo trains a single 29-DoF G1 policy to recover from prone, supine, and side states and transition directly into commanded walking across flat, gravel, grass, and slopes. It avoids the incompatible distributions and handoff of separate get-up and locomotion policies.

### Stable-region guidance and priors

Training samples broad postures and terrain. Terrain-conditioned stable-region guidance steers support/center-of-mass evolution toward configurations from which recovery and walking are feasible, supplying structure when command reward is meaningless on the ground. Continuously gated multi-scale motion priors regulate different time scales: recovery needs large nonperiodic whole-body motion, while upright travel benefits from gait style. Influence changes continuously with state rather than selecting a discrete controller.

The asymmetric PPO actor consumes proprioceptive history, command, gravity, joints, prior action, and terrain context and outputs all 29 targets; the critic uses privileged contacts/body/terrain. Dynamics, pushes, motor parameters, observation noise, and terrain are randomized.

### Evidence and assessment

The project shows recovery followed by walking on flat ground, gravel, 10°/15° slopes, and outdoor grass. The contribution is a common state–action distribution including the transition into locomotion. Stable-region guidance makes exploration feasible, while gated priors avoid imposing periodic style during recovery. See the [project page](https://vsislab.github.io/UniReLo/) and [paper](https://arxiv.org/abs/2606.08922).

Randomized starts cannot cover every self-collision, obstacle, payload, or damaged-joint fall, and physical wear matters. Future work should combine impact-aware falling, clutter perception, and confidence-based assistance. Recovery time, peak torque/contact force, energy, and repeated-cycle wear should accompany success rate; out-of-distribution detection could prevent unsafe get-up attempts.

## Unified Walking, Running, and Recovery for Humanoids via State-Dependent Adversarial Motion Priors

### Why one adversarial prior is counterproductive

This work trains one G1 policy for standing, commanded walking/running, falling, and prone/supine recovery. A conventional AMP discriminator trained only on gait clips would penalize every necessary get-up motion as “unnatural”; a recovery dataset alone would weaken locomotion. The solution is a simple state-dependent gate between two motion discriminators.

### Gate, data, and policy

Projected gravity provides the state test. When `|g_z + 1| > 0.6`—approximately more than 37° from upright—the recovery discriminator supplies style reward. Otherwise a velocity-conditioned locomotion discriminator evaluates transitions against walk/run examples. The recovery prior permits large nonperiodic body motions; the locomotion prior is conditioned on the commanded velocity so walking and running style changes with task rather than being averaged.

Only three LAFAN1 clips seed the adversarial references, showing data efficiency. PPO still receives task rewards for velocity/yaw tracking, uprightness, recovery progress, regularization, torque/action smoothness, contacts, and joint limits. Broad initial states include upright, perturbed, prone, and supine configurations. The actor observes command, projected gravity/angular velocity, joints, history/previous action and outputs whole-body joint targets.

### Deployment and achievement

The final policy is exported as one ONNX model and runs at 50 Hz. Hardware demonstrates smooth walk–run transitions, push/fall response, and recovery from both prone and supine states without an explicit state machine or policy switch. The conceptual highlight is that the *reward prior* switches, while the motor policy remains continuous. See the [paper](https://arxiv.org/abs/2605.18611).

### Critical assessment

The fixed tilt threshold is interpretable but crude. A crouch, roll, wall contact, or steep slope may cross it without being a fall; near the boundary, small noise can alternate style rewards. Three clips limit recovery diversity, and adversarial plausibility does not minimize impact or guarantee collision safety.

Future work should learn a hysteretic/probabilistic phase gate from contacts and motion context, add terrain/clutter awareness, and use impact-aware fall objectives. Evaluations should include threshold sensitivity, time-to-recover, peak torque/force, energy, and repeated fall wear. A continuous mixture of priors may preserve smoothness while recognizing more than two states.

## UEREBot: Learning Safe Quadrupedal Locomotion under Unstructured Environments and High-Speed Dynamic Obstacles

### Planning and reflex operate at different time scales

UEREBot addresses three competing needs: progress to a distant goal, passability around static/rough terrain, and split-second avoidance of a fast attacker. Replanning alone is too slow; a pure reflex can dodge indefinitely and abandon the route. The framework therefore runs a spatial–temporal planner, navigation policy, reflex policy, learned handoff, and control-barrier-function (CBF) shield.

### System architecture and rates

An offline 2.5-D map supplies static/terrain feasibility. LiDAR at 10 Hz, camera at 30 Hz, and state estimation at 50 Hz detect/localize obstacles. The planner updates reference path at 10–20 Hz while predicted obstacle state and threat score update at 50 Hz. Navigation and reflex policies both run at 50 Hz and propose velocity commands. A threat-aware handoff blends/selects them, after which the 50 Hz CBF shield projects the nominal command into a safe set; locomotion control executes at 200 Hz.

The planner generates several spatial–temporal candidate paths and scores progress, passability, and predicted dynamic clearance. It supplies intent and threat, not joints. The navigation network follows its path; the reflex network is trained on short-warning attacks to evade and regain stance. Separating these policies prevents fast avoidance data from erasing long-horizon navigation. The CBF is the last layer against candidate-command mistakes.

### Evidence and contribution

Isaac Lab and Unitree Go2 tests include rooms, narrow paths, grass, corridors/steps, and attacks by pokes, hits, kicks, a quadruped, and a humanoid. Comparisons report higher avoidance success/clearance while retaining goal progress. The strongest contribution is the timing-aware decomposition: deliberation supplies intent, reflex supplies reaction, and a mathematical shield constrains execution.

### Limits and next work

“Safe” is conditional on perception, obstacle prediction, barrier model, and low-level tracking. A fast object can appear inside sensor latency, and a velocity-level CBF cannot prevent body/limb collision if its robot geometry is oversimplified. Offline mapping limits unknown environments, and handoff oscillation is possible near threat thresholds.

Future work should propagate perception/prediction uncertainty into the barrier, prove or measure tracking-error margins, and build maps online. Multi-agent intent prediction and adversarial latency tests would strengthen claims. Reporting minimum clearance, false evasions, progress loss, and shield intervention frequency would expose the actual safety–efficiency trade.

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

## Visual Imitation Enables Contextual Humanoid Control

### VideoMimic learns behavior together with context

This paper turns casual monocular video into terrain-aware humanoid skills such as stair climbing and chair sitting. It reconstructs the person and surrounding 3-D scene, retargets motion inside that scene, trains physics trackers, then distills all demonstrations into one policy that receives only proprioception, an `11 x 11` local height map at 0.1 m spacing, and desired root direction. No target joint trajectory is required at deployment.

### Real-to-sim reconstruction

Grounded SAM2 tracks the person; VIMO supplies SMPL pose/shape/trajectory; ViTPose supplies 2-D joints; BSTRO estimates foot contact. MegaSaM or MonST3R reconstructs depth, cameras, and a scene cloud. Joint optimization aligns human and scene and uses a statistical human-height prior to recover metric scale. GeoCalib aligns gravity and NKSR converts the cloud into a simulator-ready mesh. Motion is kinematically retargeted while preserving scene contact/collision.

### Four policy stages

First, mocap pretraining establishes general tracking. Second, batched DeepMimic-style PPO tracks many reconstructed videos/scenes. Actor observations include five frames of joints, velocities, angular velocity, gravity, prior action, target joints/root orientation/direction, and height map; the critic has privileged state. Third, DAgger removes target joints/root roll-pitch and retains only body history, local geometry, and goal direction. Fourth, under-conditioned PPO fine-tuning lets the student discover context-driven behavior rather than merely average teacher actions.

### Achievement and judgment

The distilled policy selects stepping, climbing, or sitting based on local geometry and transfers to a humanoid. The breakthrough is that video supplies *why and where* a motion occurs, not only joint kinematics. A unified deployment interface converts many scene-specific demonstrations into contextual control.

Reconstruction errors compound across pose, camera, scale, contact, mesh, and retargeting; a height map loses chair backs/overhangs and semantics. Root-direction conditioning may not disambiguate two tasks with similar geometry. The generalist can also smooth rare behaviors during distillation.

Future work should retain volumetric/semantic context, propagate reconstruction uncertainty, and add language/task intent. Held-out object/scene evaluations and contact-force measurements would distinguish genuine contextual understanding from geometry matching. Online RGB conditioning could avoid a separate map but would reintroduce visual transfer risk.

## Walk the PLANC Physics-Guided RL for Agile Humanoid Locomotion on Constrained Footholds

### Reduced-order physics as a learnable guide

Walk the PLANC uses a Linear Inverted Pendulum/Hybrid LIP model to turn stepping-stone geometry into dynamically meaningful references, then lets PPO learn the full humanoid residual. Pure end-to-end RL often discovers standing still on sparse supports; pure model control is conservative and brittle. Physics supplies timing, momentum, and contact structure without dictating exact full-body torques.

### Planner and CLF reward

Consecutive stone centers define virtual slopes for varying height. Orbital energy selects forward-progressing center-of-mass trajectories; closed-form time-to-impact determines step timing; momentum transfer sets desired post-impact velocity. Sagittal cubic splines and the analytical lateral HLIP solution generate CoM/swing-foot references. A control-Lyapunov-function decrease condition becomes a dense RL reward instead of an online QP constraint.

The privileged teacher uses stance foot, phase, terrain, and CLF/reference terms and is trained with PPO. A deployable student receives command, base angular velocity, projected gravity, joints, and exteroceptive terrain input and outputs joint targets. It first distills the teacher, then PPO fine-tunes on the full curriculum because asymmetric PPO from scratch collapses. Ten difficulty levels expand gaps from 0.3 toward 0.7 m, height variation to ±0.2 m, and stair heights to 0.2 m.

### Contribution and assessment

Simulation and hardware demonstrate stairs and flat/height-varying constrained footholds. The strongest idea is converting approximate dynamics into shaping rather than a hard controller: incorrect model details can be corrected by RL, while the viability manifold prevents aimless exploration.

No formal CLF guarantee survives reward optimization or student distillation. Stone geometry/perception error can invalidate the reference, and the reduced model ignores swing-leg and upper-body dynamics. The training chain—planner, teacher, distillation, fine-tuning—is effective but complex.

Future work should expose viability margin to the student, add perception uncertainty to reference generation, and apply a runtime safety filter. Tests on irregular orientations, moving supports, and model mismatch would reveal whether the policy internalized physics or tracks a narrow curriculum.

## Walking with Terrain Reconstruction Learning to Traverse Risky Sparse Footholds

### Explicit reconstruction inside end-to-end RL

This predecessor to START shows that a low-cost quadruped can traverse stones, beams, stepping beams, and gaps from a single limited-view depth camera without global localization. A learned terrain reconstructor turns depth plus body history into a local height map extending beneath and behind the robot; the locomotion policy uses that map as an interpretable intermediate representation.

### Network and training

The reconstructor fuses proprioception and egocentric visual features, maintaining temporal information after terrain leaves view. It decodes the local height map under privileged simulation supervision and also produces compact estimated state/map latents. The actor receives latest proprioception, estimated base velocity, reconstructed-map features, and latent state and outputs target joint angles. Actor, critic, estimator, and reconstructor are trained jointly with PPO—no separate perception phase or global pose alignment.

Implicit–explicit learning is central: the explicit map forces depth features to preserve edges/support layout, while latent features can retain uncertainty/dynamics not captured by a single height value. Terrain curricula progressively reduce support area/increase gaps; rewards combine progress, command tracking, foot placement, stability, collisions, energy, and smoothness. Depth and dynamics randomization support zero-shot hardware transfer.

### Contribution and relationship to START

Real low-cost quadruped trials demonstrate agile sparse-terrain traversal and visualized map reconstruction. The paper establishes the core thesis later refined by START: task-trained robot-local reconstruction can give map precision without SLAM drift. START adds a more explicitly memory-augmented TR-Net and adaptive sampling for efficiency, so the two should be read as an evolution rather than independent identical methods.

### Limits and future work

Height reconstruction has no formal uncertainty and its loss may favor average surface error over edge topology. Recurrent alignment can drift, movable stones make memory stale, and a 2.5-D grid cannot model overhangs. The policy may also use hidden latents while the displayed map looks plausible, limiting interpretability.

Future work should measure edge/foothold error, not only height MAE; predict support probability/confidence; erase changed terrain; and test long occlusions. An ablation blocking latent bypass would determine how much control truly depends on reconstruction. Combining contact feedback with visual memory could correct a map after the foot discovers unexpected compliance.
