# Monash-Coles Grasp Dataset Inspection

This repository turns the extracted Monash-Coles 6-DoF robotic grasping data
into a read-only, indexed dataset with validation, visualization, statistics,
and explicit SE(3) transform conventions. It does not train a model or launch
Isaac Sim.

The completed audit found 1,500 trials across 20 objects, with every JSON, RGB,
depth, and point-cloud modality present. There are 834 successful grasps and
592 stable successes. The main Real2Sim blocker is missing per-trial object
pose, not missing trial data.

## Quick start

### Prerequisites

- Python 3.10 or newer
- NumPy and Pillow available in that Python environment
- The extracted dataset at
  `datasets/monash_coles_grasping_dataset/20174165`

Open3D is optional and only needed for an interactive 3D viewer. The default
viewer uses Pillow and writes a PNG.

### 1. Build and validate the index

```bash
python scripts/build_trial_index.py --strict --overwrite
```

This writes `outputs/trial_index.csv`. A trial ID repeats for every object, so
the unique key is `object_id/trial_id`, such as `1/0`.

### 2. Inspect a working trial

```bash
python scripts/inspect_trial.py --trial-id 1/0 \
  --output outputs/plots/trial_1_0_rgb_depth.png
```

The command checks modality correspondence, reports image and point-cloud
statistics, and writes an RGB/depth diagnostic. You now have a verified sample
with its source paths and outcome labels.

### 3. Load the same trial in Python

```python
from src.grasp_dataset import GraspDataset
from src.transforms import pose_mapping_to_transform

dataset = GraspDataset()
trial = dataset.get_trial("1/0")
T_base_grasp = pose_mapping_to_transform(trial.grasp_pose)

print(len(dataset), trial.success, trial.rgb.shape)
print(trial.point_cloud.points.shape, T_base_grasp)
```

Large modalities are loaded only when their properties are accessed. Raw files
are never rewritten.

## How to run the analysis workflows

### How to rebuild the trial index

```bash
python scripts/build_trial_index.py --strict --overwrite
```

Use `--strict` in automation: it returns a nonzero status for a missing,
duplicate, or mismatched modality. Verify that the summary reports 1,500
complete trials and zero issues.

### How to inspect or visualize a trial

```bash
python scripts/inspect_trial.py --trial-id 1/0
python scripts/visualize_trial.py --trial-id 1/0 \
  --output outputs/plots/trial_1_0_geometry.png
```

Select a deterministic random trial with `--random --seed 42`, or constrain a
bare trial number with `--trial-id 0 --object-id 1`. The Pillow visualization
draws the PCD and grasp axes in the verified `base_link` frame. It deliberately
does not name a gripper approach axis because that axis is undocumented.

For an interactive viewer, install Open3D and pass `--backend open3d`.

### How to validate all point clouds

```bash
python scripts/validate_point_clouds.py --strict
```

This scans all indexed PCDs for parse errors, empty clouds, NaN/Inf values,
implausible metre-scale extents, and layout changes. It does not filter or
repair source clouds.

### How to regenerate the audit and plots

```bash
python scripts/audit_dataset.py --overwrite
```

This rewrites the generated Markdown/JSON audit and aggregate PNG plots under
`outputs/`. It reads all 1,500 PCDs and can take several minutes.

### Verification and troubleshooting

Run the transform tests:

```bash
python -m unittest discover -s tests -v
```

Common failures:

- `Trial index does not exist`: run the index command first.
- `trial_id ... is ambiguous`: use `object_id/trial_id` or pass `object_id=`.
- Output already exists: add `--overwrite` only when you intend to replace a
  generated artifact. Raw dataset files are never valid output targets.
- Open3D import error: omit `--backend open3d`; the Pillow backend is the
  dependency-light default.
- Metric depth is needed: use the PCD. The 8-bit depth encoding and scale are
  unknown, so the PNG cannot safely be reprojected as metres.

## Python API reference

### `src.grasp_dataset`

- `GraspDataset(dataset_root=..., index_path=...)`: eagerly reads only the CSV
  index. Supports `len(dataset)`, iteration, integer indexing, and slicing.
- `get_trial(trial_id, object_id=None)`: accepts a composite ID such as `1/0`,
  or a trial ID plus object ID. A bare repeated trial ID raises `ValueError`.
- `get_object_trials(object_id)`: returns all indexed trials for one object.
- `GraspTrial`: exposes IDs, paths, `success`, and `stable_success` immediately;
  `metadata`, `rgb`, `depth`, and `point_cloud` are cached lazy properties.
- `GraspTrial.grasp_pose`: raw `PoseStamped`-like grasp mapping in `base_link`.
- `GraspTrial.transforms`: raw named pose mappings plus an explicitly
  unverified camera-transform candidate.
- `load_pcd(path)`: loads ASCII or uncompressed-binary PCD into
  `PointCloudData(points, colors, fields, header)`. Compressed PCD is rejected.
- `validate_point_cloud(cloud, ...)`: returns `PointCloudValidation`; structural
  and nonfinite problems are errors, while suspicious scale is a warning.

The package root exports `GraspDataset`, `GraspTrial`, `PointCloudData`,
`PointCloudValidation`, and `validate_point_cloud`.

### `src.transforms`

The project uses `T_A_B` for the pose of frame B expressed in frame A:

```text
p_A = T_A_B @ p_B
T_A_C = T_A_B @ T_B_C
```

Public functions are:

- `quaternion_xyzw_to_matrix(quaternion_xyzw)`
- `matrix_to_quaternion_xyzw(rotation)`
- `pose_to_transform(position_xyz, quaternion_xyzw)`
- `pose_mapping_to_transform(pose_stamped)`
- `invert_transform(T_A_B)`
- `compose_transforms(T_A_B, T_B_C)`
- `transform_points(T_A_B, points_B)`

Inputs are validated for shape, finiteness, orthonormal rotation, determinant,
and homogeneous bottom row. Quaternions use ROS `x,y,z,w` order.

## Frames, units, and known limits

- Grasp and robot poses are in `base_link`; positions and PCD coordinates are
  in metres.
- PCDs are unorganized binary float32 XYZRGB scene clouds already expressed in
  `base_link` with high confidence.
- The top-level JSON translation/quaternion is only a candidate camera/base
  transform. Its direction and array order are unverified; the loader never
  applies it implicitly.
- Depth PNGs are 640×480, 8-bit grayscale with no documented metric encoding.
- No per-trial object pose or object segmentation is present.
- Object-relative grasp replay requires
  `T_object_grasp = inverse(T_base_object) @ T_base_grasp`; `T_base_object` is
  currently unavailable.

Read [coordinate frames](outputs/coordinate_frames.md) before composing poses,
and [Real2Sim readiness](outputs/real2sim_readiness.md) before simulation work.

## Results and documentation

- [Registration history and correction guide](docs/monash_coles_registration_history.md):
  all material audit-to-grasp steps, failed candidates, accepted evidence,
  commands, limitations, correction procedure and a baseline reference snapshot.
- [Reusable dataset-understanding playbook](docs/dataset_understanding_playbook.md):
  the full 12-step workflow, completion gates, progress template, and a
  copy-paste prompt for future robotics datasets.
- [Data dictionary](outputs/data_dictionary.md): JSON, image, PCD, and index
  schemas.
- [Coordinate frames](outputs/coordinate_frames.md): evidence, units,
  quaternion ordering, and transform policy.
- [Dataset audit](outputs/dataset_audit.md): full counts, distributions, and
  integrity results; [machine-readable JSON](outputs/dataset_audit.json).
- [Object-relative pose readiness](outputs/object_relative_pose.md): why the
  missing object pose cannot be fabricated and what registration requires.
- [Real2Sim readiness](outputs/real2sim_readiness.md): capability matrix and
  ordered gates for safe Isaac Sim replay.
- [Trial index](outputs/trial_index.csv): one row per composite sample.
- [Plots](outputs/plots/): modality diagnostics, outcome counts, per-object
  rates, grasp-position distributions, and 3D geometry views.

## Project structure

```text
datasets/.../20174165/  immutable extracted dataset
docs/                    reusable dataset-understanding workflow
outputs/                generated index, reports, and plots
scripts/                index, inspect, visualize, validate, and audit CLIs
src/grasp_dataset.py    lazy read-only dataset and PCD loader
src/transforms.py       explicit SE(3) utilities
tests/                  transform unit tests
create_dataset.md       original task specification
```

The public dataset description is available from the
[Monash-Coles project page](https://lachlanchumbley.github.io/Monash-ColesGraspingDataset/),
and the separate mesh collection is described on the
[Coles object-set page](https://lachlanchumbley.github.io/ColesObjectSet/).

## Five-trial registration preparation

Object **3 (Bathroom Cleaner)** now has a visually verified mesh match. The
five selected trials have deterministic table removal, clustering and saved
three-view preprocessing diagnostics. **Mesh units and scale remain unverified;
registration and `T_object_grasp` were stopped at the requested scale gate.**

```bash
uv pip install --python .venv/bin/python -r requirements-registration.txt
.venv/bin/python scripts/fetch_selected_mesh.py
.venv/bin/python scripts/prepare_registration_subset.py
```

The fetch command retrieves only the pinned mesh member and verifies its bytes.
Preparation saves derived files to `outputs/registration_preparation/`; pass
`--overwrite` to regenerate that directory. Original trial data and mesh bytes
are preserved. Configuration is in `configs/registration_subset.json`; the
single verified identity mapping is in `configs/object_mesh_mapping.json`.

See [registration readiness](outputs/registration_readiness.md) for the scale
evidence, stop condition and remaining implementation; [object/mesh candidates](outputs/object_mesh_candidates.md)
for the complete ID and public filename inventory; and
[subset manifest](outputs/registration_preparation/subset_preparation.json)
for labels, observed geometry metrics and known base-frame grasp transforms.

## Object 16 follow-up

The next selected object, ID **16**, is visually verified as **Soap Box**. The
earlier provisional Water Crackers name was corrected after inspecting readable
Pure Soap labels, author references and the actual colored mesh.

Five trial clouds and fixed-scale geometric/color-orientation experiments are
in [the Soap Box scale report](outputs/soap_box/mesh_scale_analysis.md), with
configuration in `configs/registration_subset_soap_box.json` and
`configs/object_mesh_scale_soap_box.yaml`. Scale consistency improved, but the
all-five physical and visual gate remains unmet. No scale or object-relative
grasps are accepted. The Bathroom Cleaner failure evidence remains preserved.

## Dishwasher Powder screening

Object **12 (Dishwasher Powder)** passed the fast geometric gate with one
uniform scale of **0.068 m/native unit**. Trials **12/6, 12/57 and 12/69** have
visually validated GOOD object poses and exported object-relative grasps;
12/0 and 12/15 remain QUESTIONABLE and are excluded. No Isaac launch or USD
generation was performed.

See the [candidate decision](outputs/dishwasher_powder/candidate_decision.md),
[scale analysis](outputs/dishwasher_powder/mesh_scale_analysis.md),
[accepted grasps](outputs/dishwasher_powder/object_relative_grasps.json), and
[Isaac handoff](outputs/dishwasher_powder/isaac_replay_inputs.md). The handoff
separates verified geometry from unknown physics and robot trajectories.
