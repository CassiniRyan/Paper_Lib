# Human Motion Reconstruction — Paper Notes

The two papers in this folder are discussed according to the problems they actually solve. TRAM is best understood as a decomposition of world-space human reconstruction into scene motion, metric scale, and body motion; GVHMR is best understood through its gravity-aligned representation and treatment of global trajectory. The notes therefore spend space on those ideas and their evidence instead of forcing both papers into a fixed checklist. Both entries were analyzed from the local full-text PDFs.

## TRAM: Global Trajectory and Motion of 3D Humans from in-the-wild Videos

### Why scene-relative motion is hard

TRAM reconstructs a person's full 3D motion—including local articulated pose and the global trajectory through the scene—from an ordinary monocular video recorded by a potentially moving camera. The key difficulty is separating camera motion from human motion while also recovering metric scale. Conventional human-mesh recovery often produces poses only in camera coordinates; monocular SLAM provides camera motion only up to scale and can fail when a large moving person violates its static-scene assumption.

### What the separation buys

The paper demonstrates that a deliberately separated two-stage pipeline can recover substantially better world-space trajectories than methods that infer global displacement mainly from human-motion priors. On EMDB-2, TRAM reports a root translation error of 1.4%, versus 4.6% for the reported WHAM-with-DROID baseline, and improves the reported 100-frame world-aligned and world-frame joint errors. Its VIMO component also improves pose/shape reconstruction and temporal smoothness on 3DPW and EMDB compared with the paper's HMR2.0 fine-tuning baseline and prior video methods.

### Scene scale, masking, and two kinds of temporal reasoning

- **Scene-centric scale estimation:** rather than estimating scale from a learned human locomotion prior, TRAM aligns SLAM depth to metric monocular depth predicted from the static scene. This is particularly helpful for unusual motions—stairs, parkour, or long trajectories—that may fall outside motion-capture training distributions.
- **Dual masking for dynamic-human SLAM:** detected human regions are removed both from DROID-SLAM's input images and from the confidence weights used by dense bundle adjustment. Image masking protects the learned global image features; confidence masking removes dynamic pixels from geometric optimization.
- **Robust sequence-level scale aggregation:** it estimates a scale independently at multiple keyframes using robust Geman–McClure least squares, excludes unreliable far-depth regions, and takes the median across the sequence.
- **VIMO:** an image HMR model is converted into a video model by inserting one transformer in the image-feature domain and another directly in the SMPL-motion domain.

### From moving-camera video to a metric human trajectory

1. YOLOv7 detects the person and Segment Anything produces a human mask.
2. Masked DROID-SLAM estimates relative camera poses and scene depth using only static-background evidence.
3. ZoeDepth predicts metric depth. Robust alignment between ZoeDepth and DROID depth supplies a metric scale for the camera trajectory.
4. VIMO estimates the person's SMPL pose, shape, root orientation, and camera-relative translation for every frame.
5. The metric camera motion is composed with camera-relative human motion to obtain the world-space root trajectory and articulated motion.

The decomposition is easy to reason about and diagnose: camera/scale errors belong to the scene branch, while pose errors belong to VIMO. Its weakness is that the two branches cannot correct one another until an optional future joint-refinement stage.

### VIMO and the supporting visual models

- **Human model:** SMPL, with 23 relative joint rotations, 10 shape coefficients, root orientation, and camera-relative translation.
- **Backbone:** the pretrained HMR2.0 ViT-Huge image encoder. TRAM freezes this backbone during video fine-tuning to preserve its broad image-level recognition ability.
- **Image temporal transformer:** attention is applied across time independently for corresponding ViT patch locations. It improves appearance and motion features using neighboring frames.
- **Motion temporal transformer:** an encoder/decoder operates directly on sequences of SMPL pose variables. It acts as a learned motion-space denoiser and smoothness prior without requiring a separately pretrained motion model.
- **SLAM/depth models:** DROID-SLAM supplies learned dense visual odometry and scene structure; ZoeDepth supplies metric monocular depth; YOLOv7 and SAM isolate dynamic humans.

VIMO is trained end-to-end on video supervision with losses for reprojected 2D joints, 3D joints, SMPL parameters, and mesh vertices. The design's strongest conceptual choice is to put temporal attention in both the visual evidence and the final motion representation rather than only in a generic latent feature.

### What TRAM needs from the video

The only required raw sensor is monocular RGB video, but the pipeline assumes known camera focal length. It derives human masks, dense visual correspondences, monocular metric depth, camera poses, and human image features. It does not require an IMU, depth camera, calibrated scene object, known floor plane, or prebuilt scene model.

### Evidence for each part of the decomposition

VIMO is trained on 3DPW, Human3.6M, and BEDLAM for 100,000 AdamW iterations using 16-frame windows and batches of 24 sequences. Evaluation separates three concerns:

- pose/shape: MPJPE, PA-MPJPE, and per-vertex error;
- temporal quality: acceleration error;
- global trajectory: W-MPJPE100, WA-MPJPE100, normalized root translation error, and egocentric root-velocity error;
- camera estimation: absolute trajectory error with either aligned or estimated scale.

The ablations are useful because they independently test input masking, dense-bundle-adjustment masking, scale recovery, and each temporal transformer. In the reported EMDB camera experiment, dual masking reduces average DROID trajectory error from 2.42 m to 0.32 m after ground-truth scale alignment.

### Where the modular pipeline still breaks

- Known focal length is a meaningful restriction for arbitrary internet video.
- Metric monocular depth becomes unreliable for unusual focal lengths, distant backgrounds, sky, or visually ambiguous scenes.
- Human and camera motion are estimated separately, so physical constraints such as contact, penetration, and foot sliding are not jointly enforced.
- A large multi-model perception stack offers modularity and strong performance but also increases computation, memory, and failure modes.

TRAM is best viewed as a strong scene-referenced reconstruction system rather than a physics-aware motion estimator. Its most reusable insight is that the static environment can be a better metric reference than a learned locomotion prior. For robotics, its outputs are useful for building world-frame imitation data, but retargeting, contact cleanup, and embodiment-specific feasibility checks are still required.

## World-Grounded Human Motion Recovery via Gravity-View Coordinates

### Removing ambiguity from the world frame

This paper presents GVHMR, a feed-forward method for recovering continuous 3D human pose, shape, and global trajectory from monocular video in a gravity-aware world frame. It targets two weaknesses of previous global-motion methods: ambiguity in the chosen world coordinate system and accumulated drift from autoregressive prediction, particularly along the gravity axis and over long sequences.

### Accuracy and speed of direct sequence prediction

GVHMR reports state-of-the-art camera-space and world-grounded reconstruction on RICH, EMDB, and 3DPW under the paper's evaluation protocol. On EMDB with estimated DPVO camera rotations, it reports 111.0 mm WA-MPJPE100, 276.5 mm W-MPJPE100, and 2.0% root translation error; the corresponding reported WHAM results are 135.6 mm, 354.8 mm, and 6.0%. The core network processes a 1,430-frame video in about 0.28 seconds on an RTX 4090 after preprocessing, compared with 2.0 seconds for WHAM's core and more than six hours for the optimization-based SLAHMR pipeline in the paper's timing test.

### Gravity-View coordinates and bounded parallel inference

- **Gravity-View coordinates:** every frame receives a coordinate system whose y-axis follows gravity and whose remaining axes are fixed using camera viewing direction. This makes a person's gravity-aware orientation much less ambiguous than learning an arbitrary global orientation.
- **One-degree-of-freedom global orientation recovery:** adjacent Gravity-View frames differ only by rotation around the gravity axis. Relative camera rotation is therefore used to align all frames to the first frame's GV coordinate system without accumulating roll/pitch error.
- **Parallel rather than autoregressive inference:** the complete motion sequence is regressed with a transformer, avoiding the initialization sensitivity and sequential drift of recurrent rollout.
- **RoPE and bounded receptive field:** rotary positional embeddings represent relative frame relationships, while an inference-time attention mask limits each frame to the training-length neighborhood. The model can process sequences longer than those seen during training without a sliding window.
- **Contact-style cleanup:** predicted stationary probabilities for hands, toes, and heels guide root-translation correction and a CCD inverse-kinematics pass, reducing foot sliding and implausible contact motion.

### From image evidence to a world trajectory

1. Preprocessing detects and tracks the person, obtains bounding boxes and 2D joints, extracts per-frame image features, and estimates relative camera rotation.
2. Individual MLPs project bounding-box features, 2D keypoints, HMR2.0 image features, and relative camera rotations to a common size; early fusion adds them into a 512-dimensional token per frame.
3. A 12-layer, 8-head temporal transformer with RoPE processes the full sequence.
4. Multi-task MLP heads predict weak-perspective camera parameters, camera-frame orientation, SMPL-X pose and shape, joint-stationary probabilities, Gravity-View orientation, and local root velocity.
5. Weak camera predictions are converted to full camera translation. Relative camera rotations align GV orientations to a shared world frame, and transformed root velocities are accumulated to form global translation.
6. Stationary-joint constraints refine root motion, followed by CCD inverse kinematics for local pose cleanup.

### The temporal model and its prediction heads

- **Body model:** SMPL-X with 21 local body-joint rotations and 10 shape coefficients; the method predicts camera- and world-frame root orientation/translation in addition to local pose.
- **Image representation:** fixed HMR2.0/ViT features rather than end-to-end raw-image encoding inside the temporal model.
- **2D pose and tracking:** YOLOv8-based detection and ViTPose are part of the reported preprocessing stack.
- **Camera motion:** DPVO supplies relative camera rotations; ground-truth gyroscope rotations are used only as an alternative evaluation input.
- **Temporal model:** 12 transformer encoder layers, eight attention heads, hidden dimension 512, and two-layer GELU MLPs. RoPE encodes relative temporal position.
- **Output heads:** multi-task MLPs jointly regress pose, shape, camera, GV orientation, root velocity, and contact/stationarity signals.

Training combines MSE losses on continuous targets, BCE for stationary probabilities, and L2 supervision for 2D/3D joints, vertices, camera translation, and world translation.

### A light world model, but a substantial front end

The method consumes monocular RGB video after a substantial learned preprocessing pipeline: human boxes, 2D keypoints, ViT image features, and relative camera rotation. Gravity is inferred through the human/image representation and the GV construction; no physical IMU or calibrated floor plane is required. Unlike TRAM, it does not estimate a dense scene model or use monocular depth to establish the trajectory.

### Long-sequence evidence and ablations

GVHMR is trained from scratch on a mixture of AMASS, BEDLAM, Human3.6M, and 3DPW. AMASS sequences receive simulated static/dynamic cameras; video datasets use fixed image features. The training sequence length is 120 frames, batch size 256, and training runs for 500 epochs (reported as about 13 hours on two RTX 4090 GPUs).

Evaluation uses RICH and EMDB-2 for world-space motion and 3DPW, RICH, and EMDB-1 for camera-space reconstruction. Metrics cover global alignment error, root translation, jitter, foot sliding, joint/vertex accuracy, and acceleration. Ablations support the importance of GV coordinates, direct GV-orientation prediction, the transformer, RoPE, and post-processing. A notable robustness result is that replacing ground-truth gyroscope rotation with DPVO produces only a small reported degradation, suggesting that the gravity-constrained representation absorbs some camera-rotation noise.

### Drift, preprocessing cost, and physical plausibility

- Most of the impressive core-network speed excludes detection, 2D pose estimation, feature extraction, and visual odometry; preprocessing takes roughly as long as the example video itself in the reported test.
- Global translation is still produced by integrating predicted local root velocity, so yaw/velocity error can accumulate even though gravity-axis orientation drift is structurally controlled.
- Stationary-contact correction is a kinematic heuristic, not full physical reasoning; it does not guarantee force feasibility, collision avoidance, or consistent scene contact.
- Results depend on multiple pretrained front-end models and their behavior under occlusion, truncation, unusual camera motion, or failed person tracking.
- The method predicts a body motion representation suitable for data generation, but robot imitation still needs retargeting and dynamics-aware filtering.

GVHMR's strongest contribution is representational: it removes unnecessary degrees of freedom before asking a network to learn the problem. Compared with TRAM's scene-centric geometry, GVHMR favors fast direct regression with gravity constraints. The approaches are complementary—TRAM has an explicit metric scene/camera reference, while GVHMR is substantially faster and directly optimized for long-sequence human motion.
