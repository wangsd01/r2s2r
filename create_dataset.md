You are helping me build a robotics manipulation research pipeline that will eventually connect real-world grasp data, Isaac Sim, Sim2Real evaluation, and learned world models.

For now, DO NOT build a world model, train a large neural network, or modify the production grasp pipeline.

The first milestone is to understand, audit, and standardize our EXISTING real-world grasp dataset.

## Current dataset

The dataset contains some or all of the following:

- Part identifier
- Part CAD model
- Real-world RGB image(s)
- Real-world point cloud
- Grasp pose
- Grasp success/failure label
- Possibly robot pose/state, gripper information, camera calibration, timestamps, or other metadata

The exact current schema may differ, so inspect the repository and dataset structure first instead of assuming field names.

The eventual research pipeline will be:

Real grasp dataset
→ standardized grasp benchmark
→ grasp outcome prediction
→ replay the same grasps in Isaac Sim
→ compare Real vs Sim outcomes
→ identify Sim2Real mismatch
→ collect targeted trajectory data
→ learned dynamics / world model
→ model-based grasp evaluation and planning

However, this task ONLY covers the first step: understanding and organizing the existing dataset.

# Goal

Create a clean, reproducible dataset interface where one grasp trial can conceptually be represented as:

D_i = {
    part_id,
    image,
    point_cloud,
    grasp_pose,
    success,
    metadata
}

If coordinate information allows it, eventually I also want the grasp represented relative to the object:

T_object_grasp

instead of relying only on camera-frame or robot-base-frame coordinates.

Do not fabricate transforms if calibration or object pose information is unavailable.

# Step 1 — Inspect before changing anything

First inspect:

1. Repository structure
2. Dataset directories
3. Existing dataset loaders
4. File naming conventions
5. Existing grasp-pose representation
6. Success/failure label representation
7. CAD-to-part mapping
8. Image-to-point-cloud mapping
9. Coordinate-frame definitions
10. Camera calibration data, if any
11. Robot/gripper metadata, if any
12. Existing train/test split logic, if any

Before writing substantial code, give me a concise summary of what currently exists.

Explicitly identify anything that is ambiguous.

For coordinate frames, determine whether each pose is expressed in:

- camera frame
- robot base frame
- end-effector frame
- gripper frame
- object frame
- another frame

Do not guess if this cannot be determined from the code/data.

# Step 2 — Produce a dataset data dictionary

Create a clear table or Markdown document describing each available field.

For example:

| Field | Example | Shape/type | Coordinate frame | Units | Required? | Notes |
|---|---|---|---|---|---|---|
| part_id | ... | string | N/A | N/A | yes | |
| rgb | ... | H×W×3 | camera | pixels | optional | |
| point_cloud | ... | N×3 | camera? | meters? | yes | verify |
| grasp_pose | ... | 4×4 / xyz+quat | ? | meters/radians | yes | verify |
| success | 0/1 | bool | N/A | N/A | yes | |
| cad_path | ... | path | object frame | ? | yes | |

Include confidence/uncertainty where needed.

# Step 3 — Build a non-destructive dataset audit script

Create a script such as:

scripts/audit_grasp_dataset.py

It must NOT modify the original dataset.

It should report at least:

- total number of grasp trials
- number of unique parts
- number of grasps per part
- success rate overall
- success rate by part
- missing files
- invalid labels
- missing CAD models
- missing images
- missing point clouds
- malformed grasp poses
- NaN / Inf values
- point-cloud size distribution
- grasp-position distribution
- grasp-orientation distribution if practical
- duplicated sample IDs if they exist
- obvious class imbalance
- obvious dataset imbalance across parts

Save machine-readable results where appropriate, for example:

outputs/dataset_audit.json
outputs/dataset_summary.csv

and a human-readable report:

outputs/dataset_audit.md

Do not overwrite existing output files without warning.

# Step 4 — Build a clean dataset abstraction

Create or propose a lightweight dataset interface, for example:

GraspSample:
    sample_id
    part_id
    cad_path
    rgb
    point_cloud
    grasp_pose
    success
    metadata

and:

GraspDataset:
    __len__()
    __getitem__()
    get_part_samples(part_id)
    validate_sample(sample)
    summary()

Prefer adapting existing code rather than duplicating loaders.

Do not force all large image or point-cloud data into memory.

Use lazy loading if appropriate.

# Step 5 — Normalize pose representation

Inspect how grasp poses are currently stored.

Prefer a canonical internal representation such as:

4×4 SE(3) transform

while preserving access to the original representation.

Clearly document:

T_A_B means "pose of frame B expressed in frame A"

and use this convention consistently.

Do not silently transpose/invert transforms.

Add explicit utility functions for:

- xyz + quaternion → SE(3)
- SE(3) → xyz + quaternion
- pose inversion
- pose composition
- transforming point clouds

Add small unit tests for pose operations.

# Step 6 — Investigate object-relative grasp pose

The desired representation is eventually:

T_object_grasp

Investigate whether this can be computed from the available data.

For example, if:

T_camera_object

and

T_camera_grasp

are known, then:

T_object_grasp =
inverse(T_camera_object) @ T_camera_grasp

But ONLY implement this if the frame definitions and calibration information are verified.

If object pose is unavailable, clearly state what additional information is needed.

Do not estimate CAD alignment yet unless an existing pipeline already does so.

# Step 7 — Dataset visualization

Create a simple visualization/debug tool that lets me inspect random samples.

For a selected grasp trial, show as much as available:

- RGB image
- point cloud
- CAD model if easy to load
- grasp coordinate frame
- gripper pose / grasp pose
- success/failure label
- part ID
- relevant metadata

If possible, visualize the grasp frame over the point cloud.

The purpose is DATA VALIDATION, not presentation quality.

I should be able to run something like:

python scripts/visualize_grasp_sample.py --sample-id XXX

or:

python scripts/visualize_grasp_sample.py --random

# Step 8 — Design appropriate dataset splits

Do NOT only use a random grasp-level train/test split.

Propose at least two evaluation schemes:

1. Within-part held-out grasp split
   - tests new grasps on known parts

2. Part-held-out split
   - tests generalization to completely unseen parts

Avoid leakage where nearly identical grasps from the same trial/session appear in both train and test.

If session IDs, timestamps, or repeated captures exist, investigate whether they should also define split boundaries.

Create deterministic split generation with a fixed seed.

Save split manifests rather than copying data.

For example:

splits/
    within_part_train.txt
    within_part_val.txt
    within_part_test.txt

    part_holdout_train.txt
    part_holdout_val.txt
    part_holdout_test.txt

# Step 9 — Produce useful dataset statistics

Create plots/tables for:

- grasps per part
- success/failure ratio
- success rate per part
- grasp XYZ distribution
- grasp orientation distribution if interpretable
- point-cloud point-count distribution
- missing-data statistics

If possible, investigate whether grasp poses are highly biased, e.g.:

- mostly top-down grasps
- narrow orientation distribution
- narrow object-size range

Flag these because they will affect model generalization.

# Step 10 — Prepare for the NEXT milestone, but do not implement it yet

At the end, tell me whether the dataset is ready for:

A. PointCloud + GraspPose → P(success)

B. Replaying the same real grasp poses in Isaac Sim

C. Computing T_object_grasp

D. Future trajectory/world-model work

For each item, classify it as:

READY
PARTIALLY READY
NOT READY

and explain what is missing.

# Engineering constraints

- Do not modify original raw data.
- Do not rename or move raw dataset files.
- Do not make destructive changes to existing production code.
- Prefer adding isolated utilities/scripts/configs.
- Reuse existing project infrastructure where reasonable.
- Keep code modular and readable.
- Avoid introducing heavy dependencies unless necessary.
- Use existing project conventions.
- Add unit tests for coordinate-transform utilities.
- Make every analysis reproducible with a fixed random seed.
- Never assume meters vs millimeters; verify.
- Never assume quaternion convention; verify xyzw vs wxyz.
- Never assume camera convention; inspect documentation/code.
- Never silently convert coordinate systems.
- Surface uncertainty explicitly.

# How I want you to work

Do not try to complete the entire project in one large rewrite.

Work iteratively.

Start by:

1. inspecting the repository and dataset,
2. summarizing what exists,
3. identifying coordinate-frame and schema ambiguities,
4. proposing the smallest safe first implementation.

Then implement the audit and dataset abstraction incrementally.

After each meaningful step, tell me:

- what you found
- what you changed
- what remains uncertain
- what command I should run to verify it
- what the result means for the Real2Sim/world-model project

The immediate definition of success is:

"I can select any grasp trial and reliably know which part it belongs to, load its RGB/point cloud/grasp pose/label, understand the coordinate frames and units, visualize the grasp relative to the observed geometry, and create leakage-safe benchmark splits."

Do not start Isaac Sim integration until this milestone is complete.