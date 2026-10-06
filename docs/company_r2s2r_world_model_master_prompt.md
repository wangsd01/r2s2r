# Company Master Prompt — Grasp Dataset → Real2Sim → R2S2R → World Model → Planning

You are helping me build a robotics manipulation research and engineering pipeline inside a company environment.

The project should connect:

**existing real-world grasp data**
→ **dataset audit and standardization**
→ **static grasp outcome prediction**
→ **CAD / USD / frame alignment**
→ **Real-to-Sim grasp replay**
→ **Real-vs-Sim mismatch analysis**
→ **physical calibration and domain randomization**
→ **targeted real trajectory collection**
→ **simulation trajectory generation**
→ **learned dynamics / world model**
→ **uncertainty-aware prediction**
→ **grasp ranking / counterfactual evaluation**
→ **model-based planning / MPC**
→ **optional visual latent world model**
→ **optional world-action model research direction**

This is an incremental engineering/research program.

Do NOT try to implement everything at once.

Do NOT skip validation gates.

Do NOT upload proprietary data, CAD, images, logs, code, model weights, screenshots, metadata, or configs to any external service unless explicitly approved by company policy.

Do NOT copy company data or code to a personal machine.

Do NOT change production pipelines destructively.

Prefer isolated experiment packages, configs, tests, and reproducible reports.

---

# 0. Operating principles

## 0.1 Data and IP safety

Treat all company data, code, CAD, robot information, internal file paths, calibration values, model outputs, and simulation parameters as confidential unless explicitly marked otherwise.

Rules:

- Never upload internal files to public websites.
- Never send internal data to external APIs unless approved.
- Never publish internal screenshots.
- Never copy internal datasets to personal devices.
- Never push proprietary assets to public repositories.
- Keep experimental outputs in approved internal storage.
- If public datasets or public models are used, keep them clearly separated from proprietary assets.

Maintain a short provenance note for every artifact:

- PUBLIC
- COMPANY INTERNAL
- DERIVED FROM COMPANY INTERNAL
- MIXED

---

# 1. Overall project objective

The long-term technical question is:

> Can simulation and learned world models predict real-world grasp outcomes well enough to support reliable grasp evaluation and model-based robot planning?

Break this into progressively harder questions:

### Q1 — Static outcome prediction

Can we predict:

\[
P(success \mid observation, grasp)
\]

from the existing real-world dataset?

### Q2 — Real-to-Sim fidelity

If we replay the same real grasp in simulation:

\[
y_{sim}
\]

how well does it agree with:

\[
y_{real}
\]

?

### Q3 — Physical sensitivity

Which simulation assumptions most affect Real-vs-Sim agreement?

Examples:

- object pose
- collision geometry
- friction
- mass
- inertia
- gripper force
- controller parameters
- contact parameters
- sensor noise

### Q4 — Dynamic prediction

Can we learn:

\[
p(s_{t+1}\mid s_t,a_t)
\]

or:

\[
p(z_{t+1}\mid z_t,a_t)
\]

well enough to predict manipulation dynamics?

### Q5 — Planning

Can the learned model rank or plan candidate grasps/actions better than a static score alone?

---

# 2. Phase A — Inspect and standardize the existing real grasp dataset

## Goal

Create a reproducible, non-destructive dataset interface where a single grasp trial can conceptually be represented as:

```text
GraspTrial:
    trial_id
    part_id
    cad_id / cad_path
    image(s)
    point_cloud
    grasp_pose
    robot_state
    camera_calibration
    success
    stable_success if available
    metadata
```

The exact schema must come from the real repository/data.

Do NOT assume field names.

---

## A1. Inspect before modifying

Inspect:

- repository structure
- dataset directories
- existing loaders
- data formats
- trial naming
- part identifiers
- CAD mappings
- image mappings
- point-cloud mappings
- grasp pose format
- success/failure labels
- robot states
- gripper states
- camera intrinsics
- camera extrinsics
- timestamps
- train/test splits
- existing simulation links if any

Produce:

```text
outputs/dataset_audit/
    repository_summary.md
    data_dictionary.md
    unresolved_questions.md
```

---

## A2. Coordinate-frame audit

This is a critical gate.

Identify all relevant frames:

- world
- robot base
- camera
- part/object
- end-effector
- tool/TCP
- grasp
- gripper
- fingers

Adopt one explicit notation:

\[
T_{A,B}
\]

means:

> pose of frame B expressed in frame A.

Verify:

\[
p_A = T_{A,B} p_B
\]

Do not silently transpose or invert transforms.

Verify:

- meters vs millimeters
- radians vs degrees
- quaternion ordering
- xyzw vs wxyz
- handedness
- camera coordinate convention
- grasp approach axis
- gripper closing axis

If any item is ambiguous, mark it UNKNOWN.

Do not guess.

---

## A3. Build a trial index

Create a non-destructive index mapping every trial to available modalities.

Suggested fields:

```text
trial_id
part_id
cad_id
rgb_path
depth_path
point_cloud_path
metadata_path
grasp_pose
success
stable_success
session_id
timestamp
```

Save:

```text
outputs/dataset_audit/trial_index.csv
```

Report:

- missing files
- duplicate IDs
- inconsistent mappings
- malformed labels
- missing CAD
- missing calibration
- missing point clouds
- missing images

---

## A4. Build a lightweight dataset abstraction

Prefer lazy loading.

Example:

```python
class GraspTrial:
    ...

class GraspDataset:
    def __len__(self): ...
    def __getitem__(self, idx): ...
    def get_trial(self, trial_id): ...
    def get_part_trials(self, part_id): ...
```

Do not load all images/point clouds into RAM.

---

## A5. Visualization gate

For an arbitrary trial, visualize:

- RGB
- depth
- point cloud
- grasp frame
- object frame if available
- EE/TCP frame
- success/failure
- part ID

Only overlay frames after confirming they are in the same coordinate system.

Definition of completion:

> I can select any trial and understand exactly what every major field means.

STOP here if frame semantics remain fundamentally unresolved.

---

# 3. Phase B — Dataset statistics and leakage-safe splits

## B1. Compute statistics

Report:

- total trials
- unique parts
- grasps per part
- success rate
- stable-success rate if available
- success rate by part
- session distribution
- grasp position distribution
- grasp orientation distribution
- point-count distribution
- missing-data distribution
- class imbalance
- object imbalance

Flag biases such as:

- mostly top-down grasps
- narrow object families
- narrow grasp widths
- repeated near-identical trials

---

## B2. Define multiple splits

Do NOT rely only on random grasp-level splitting.

Create:

### Split 1 — Within-part held-out

Tests new grasps on known parts.

### Split 2 — Part-held-out

Tests generalization to unseen parts.

### Split 3 — Session-held-out if sessions exist

Tests generalization across collection sessions.

### Split 4 — Hard-condition held-out if useful

Examples:

- unseen friction regime
- unseen orientation range
- difficult geometry class

Save manifests.

Do not copy data.

---

# 4. Phase C — Static grasp outcome baseline

## Goal

Predict:

\[
P(success \mid observation, grasp)
\]

before doing full world-model work.

---

## C1. Start with simple engineered features

Do not begin with a large neural network.

Possible features:

- grasp height
- approach angle
- gripper width
- point count near grasp center
- points between fingers
- local occupancy
- local bounding-box dimensions
- clearance
- local surface-normal statistics
- symmetry indicators

Train simple models:

- logistic regression
- random forest
- gradient boosting if already available internally

Metrics:

- AUROC
- AUPRC
- calibration
- precision/recall
- top-K success rate
- per-part performance

---

## C2. Point-cloud model

Then evaluate a small point-cloud model.

Conceptual structure:

```text
local point cloud in grasp frame
        ↓
   PointNet-like encoder
        ↓
geometry embedding
        +
grasp features
        ↓
       MLP
        ↓
P(success)
```

Do not optimize architecture excessively.

The purpose is representation learning and baseline comparison.

---

## C3. Failure analysis

Visualize:

- high-confidence false positives
- high-confidence false negatives
- part-specific failures
- geometric failure patterns

Save a failure taxonomy.

This taxonomy will later inform Real-vs-Sim analysis.

---

# 5. Phase D — CAD / object model / USD readiness

## D1. Verify part-ID-to-CAD mapping

Create an explicit mapping:

```text
part_id → CAD / mesh / USD
```

Do not infer silently.

Verify using:

- metadata
- file naming
- geometry
- existing internal mappings
- visual inspection

---

## D2. Metric scale

For every asset intended for simulation, verify:

- physical units
- scale
- bounding-box dimensions
- origin
- orientation
- part coordinate frame

Do not proceed if asset scale is uncertain.

---

## D3. Visual vs collision geometry

Keep separate:

- visual mesh
- collision mesh

Document whether collision uses:

- convex hull
- convex decomposition
- primitive approximation
- signed-distance field if supported
- triangle mesh where valid

Do not assume the visual mesh is a valid dynamic collider.

---

## D4. Simulation-ready asset report

For each selected part classify:

```text
READY
PARTIALLY READY
NOT READY
```

for:

- visual geometry
- collision geometry
- metric scale
- object origin
- object frame
- mass
- inertia
- friction
- material
- grasp compatibility

---

# 6. Phase E — Recover object-relative grasp pose

## Goal

Compute:

\[
T_{object,grasp}
\]

for replay.

If object pose is available:

\[
T_{object,grasp}
=
T_{world,object}^{-1}
T_{world,grasp}
\]

or the equivalent verified frame chain.

If object pose is unavailable, estimate it using approved internal perception or classical registration.

---

## E1. Registration

Possible pipeline:

```text
real point cloud
    ↓
object segmentation/crop
    ↓
CAD/mesh sample
    ↓
coarse registration
    ↓
ICP refinement
    ↓
T_camera_object or T_base_object
```

Do not accept registration only because ICP fitness is high.

Visual validation is mandatory.

---

## E2. Cross-trial consistency

If the same part appears in multiple trials:

- asset scale must remain fixed
- geometry must remain fixed
- only rigid pose should vary

Reject trial-specific hacks.

---

# 7. Phase F — Recover grasp-frame / TCP semantics

## Goal

Understand how stored grasp pose relates to the actual robot/gripper.

Use:

- paired grasp and EE poses
- TF tree if available
- robot model
- gripper CAD/URDF/USD
- execution code
- trajectory logs
- visualization

Estimate fixed transforms such as:

\[
T_{grasp,EE}
\]

or:

\[
T_{grasp,TCP}
\]

Validate across many trials.

Do not guess approach or closing axes from screenshots.

---

## F1. Canonical grasp frame

Define a simulator-independent canonical grasp frame:

- origin = intended grasp center
- one axis = approach
- one axis = gripper closing direction
- third axis = right-handed completion

Explicitly map:

```text
dataset grasp frame
    ↓
canonical grasp frame
    ↓
simulation robot TCP
```

Do not conflate the dataset frame with a specific robot EE link.

---

# 8. Phase G — Deterministic Real-to-Sim grasp replay

## Goal

Replay a real grasp in simulation without physics tuning.

---

## G1. Canonical object placement

Prefer:

\[
T_{world,object}=I
\]

for replay.

Then:

\[
T_{world,grasp}
=
T_{world,object}
T_{object,grasp}
\]

This preserves object-relative grasp geometry while removing irrelevant global scene placement.

---

## G2. Execution state machine

Implement:

```text
RESET
  ↓
PREGRASP
  ↓
APPROACH
  ↓
CLOSE
  ↓
LIFT
  ↓
HOLD
  ↓
SUCCESS / FAILURE
```

Record the stage explicitly.

---

## G3. Deterministic baseline physics

Use one fixed baseline config.

Unknown parameters must be labeled ASSUMED, not VERIFIED.

Examples:

- object mass
- object inertia
- object friction
- gripper friction
- gripper force
- contact offset
- solver iterations
- physics timestep

Do not tune them yet.

---

## G4. Define simulated outcome

Examples:

```text
sim_success =
    object lifted above threshold
    AND object retained during hold

sim_stable_success =
    object retained through perturbation/stability phase
```

Also record continuous metrics:

- max lift height
- slip distance
- object-gripper relative motion
- drop time
- contact duration
- contact force if available
- penetration / interpenetration proxy

---

# 9. Phase H — Real-vs-Sim baseline

Replay a small set first.

Then expand.

Create:

```text
trial_id
part_id
real_success
real_stable_success
sim_success
sim_stable_success
slip
max_lift
registration_quality
notes
```

Compute:

- confusion matrix
- overall agreement
- per-part agreement
- false-positive rate
- false-negative rate

Pay particular attention to:

\[
Sim=Success,\ Real=Failure
\]

These are simulator false positives.

They matter greatly for downstream policy evaluation.

---

# 10. Phase I — Physical sensitivity analysis

## Goal

Identify which simulation assumptions materially change task-level outcomes.

Start with one-factor-at-a-time studies.

Possible variables:

- object mass
- inertia
- object friction
- gripper friction
- gripper force
- collision geometry
- contact offset
- timestep
- solver iterations
- initial pose error
- gripper pose error

Measure:

\[
\Delta TaskPerformance
\]

for each perturbation.

Create a task-relevant sensitivity ranking.

---

## Important principle

Do NOT optimize for visual realism alone.

Focus on:

\[
\text{task-relevant fidelity}
\]

Ask:

> Which errors actually change grasp success, slip, stability, or policy ranking?

---

# 11. Phase J — Calibration vs domain randomization

Use this principle:

### Calibrate

Parameters that are:

- measurable
- stable
- identifiable
- system-specific

Examples:

- link geometry
- camera extrinsics
- object scale
- TCP offset

### Randomize

Parameters that are:

- uncertain
- variable
- deployment-dependent
- difficult to identify precisely

Examples:

- friction
- contact variation
- lighting
- clutter
- small pose perturbations
- object material variation

---

## J1. Physics calibration

Where appropriate, fit parameters:

\[
\theta^*
=
\arg\max_\theta
Agreement(Sim(G,\theta), Real(G))
\]

Do not overfit to one part.

Use calibration / validation / test partitions across parts if possible.

---

## J2. Probabilistic simulation

For uncertain parameters:

\[
\theta_k \sim p(\theta)
\]

run multiple simulations per grasp.

Estimate:

\[
P_{sim}(success \mid grasp)
\]

instead of returning only deterministic success/failure.

This is important for uncertainty-aware evaluation.

---

# 12. Phase K — Decide when to collect new real trajectory data

Existing static data may contain:

```text
image / point cloud
grasp pose
success/failure
```

This is enough for:

- static grasp prediction
- Real-vs-Sim replay
- physical calibration

It is NOT enough for a full temporal world model.

---

## Collect multi-frame/video trajectory data when the question becomes:

\[
p(s_{t+1}\mid s_t,a_t)
\]

or when we need to explain:

- approach failure
- first-contact failure
- closing dynamics
- slip
- lift instability
- delayed drop
- controller correction

---

## K1. Prefer structured trajectory data over raw video alone

At synchronized timestamps save:

- RGB
- depth
- point cloud if practical
- robot joint position
- robot joint velocity
- EE/TCP pose
- gripper width
- gripper command
- object pose if available
- force/torque if available
- contact sensor if available
- execution stage
- timestamp

Conceptually:

\[
(o_t,s_t,a_t,o_{t+1},s_{t+1})
\]

---

## K2. Key stages

At minimum capture:

```text
PRE-APPROACH
PRE-CONTACT
FIRST CONTACT
GRIPPER CLOSED
LIFT
HOLD
```

If high-rate continuous capture is available, keep it.

If not, synchronized stage-based observations are still valuable.

---

## K3. Targeted collection

Do not immediately recollect the entire dataset.

Prioritize:

- real fail / sim success
- real success / sim fail
- borderline grasps
- high-uncertainty grasps
- high-confidence model mistakes
- slip-heavy cases
- contact-rich cases

This gives more information per real robot trial.

---

# 13. Phase L — Build a simulation trajectory dataset

Use Isaac Sim as a controlled data engine.

For each episode record:

```text
episode_id
part_id
initial object pose
physics parameters
state trajectory
action trajectory
contact signals
RGB/depth if enabled
final outcome
```

Recommended conceptual schema:

```text
episode/
    observations/
        q
        dq
        ee_pose
        object_pose
        object_velocity
        gripper_width
        contact
        rgb
        depth

    actions/
        ee_delta
        gripper_command

    parameters/
        mass
        friction
        collider
        controller
        randomization_seed

    outcome/
        success
        stable_success
        slip
```

---

# 14. Phase M — Train the first learned dynamics model

Do NOT start with video diffusion.

Start with low-dimensional state.

Example:

\[
s_t =
[
q_t,
\dot q_t,
T^{ee}_t,
T^{obj}_t,
v^{obj}_t,
g_t,
contact_t
]
\]

and:

\[
a_t =
[
\Delta x,\Delta y,\Delta z,
\Delta r,
gripper\ command
]
\]

Predict:

\[
\Delta s_t
=
s_{t+1}-s_t
\]

with:

\[
\hat s_{t+1}
=
s_t+f_\theta(s_t,a_t)
\]

Start with a small MLP.

---

## M1. Evaluate one-step prediction

Metrics:

- object translation error
- object rotation error
- velocity error
- gripper/object relative pose error
- contact-state accuracy
- slip prediction
- outcome prediction

---

## M2. Evaluate rollout

Test:

\[
H=1,5,10,20,50
\]

Track error growth.

This exposes:

\[
\text{compounding model error}
\]

which is central to world-model planning.

---

# 15. Phase N — World-model uncertainty

Do not assume the model always knows.

Simple first approach:

Train an ensemble:

\[
f_1,\ldots,f_K
\]

Estimate predictive disagreement:

\[
Var(\hat s_{t+1})
\]

Use uncertainty to identify:

- OOD states
- risky grasps
- long-horizon unreliable rollouts
- candidate real-data collection cases

---

# 16. Phase O — Use the world model for grasp evaluation

Given candidate grasps:

\[
G_1,\ldots,G_N
\]

use the world model to predict future interaction/outcome.

Compare:

```text
world-model ranking
vs
Isaac simulation ranking
vs
real-hardware ranking
```

Important metric:

\[
Spearman(\text{ranking}_{model},
         \text{ranking}_{real})
\]

Policy/grasp ranking may be more important than predicting the exact success percentage.

---

# 17. Phase P — Model-based planning / MPC

Once short-horizon dynamics are usable:

Sample candidate action sequences:

\[
A^1,\ldots,A^N
\]

Roll them out:

\[
\hat s_{t+1:t+H}
\]

Score with:

\[
R =
w_1 goal
-
w_2 slip
-
w_3 collision
+
w_4 grasp\ stability
\]

Choose:

\[
A^*
=
\arg\max_A R
\]

Start with:

- random shooting

Then:

- CEM

Then:

- receding-horizon MPC

Important:

execute only the first action,
observe again,
then replan.

This limits compounding world-model error.

---

# 18. Phase Q — Visual / latent world model

Only after the state-based world model works.

Use:

```text
RGB / RGB-D
    ↓
encoder
    ↓
z_t
```

Then learn:

\[
(z_t,a_t)\rightarrow z_{t+1}
\]

Do NOT necessarily train the encoder from scratch.

Start with a pretrained visual encoder if approved internally.

Evaluate whether the latent representation preserves:

- geometry
- object pose
- relative pose
- contact state
- motion
- affordance

while ignoring irrelevant variation such as minor texture/lighting changes.

---

# 19. Phase R — Optional world-action model direction

Do not start here.

After the world model and planning pipeline work, consider:

\[
(o_t, goal)
\rightarrow
future\ representation + action
\]

Possible formulations:

- imagine-then-execute
- world-model features conditioning an action model
- joint future/action model
- world prediction as auxiliary objective for policy learning

Treat this as a research extension, not the initial milestone.

---

# 20. Phase S — Evaluation framework

Build evaluation around robot usefulness, not only prediction metrics.

## Static evaluation

- AUROC
- AUPRC
- calibration
- top-K success
- held-out-part generalization

## Simulation evaluation

- real/sim agreement
- false-positive rate
- false-negative rate
- slip error
- stability agreement

## World-model evaluation

- one-step error
- multi-step rollout error
- uncertainty calibration
- outcome prediction
- grasp ranking correlation

## Planning evaluation

- task success
- collision rate
- slip rate
- intervention/replan count
- robustness under parameter variation

## Sim2Real utility

Most important question:

> Does simulation/model ranking predict real-hardware ranking?

---

# 21. Phase T — Experiment management

Every experiment must record:

- git commit
- config
- dataset version
- split version
- random seed
- simulator version
- physics config
- robot/gripper asset version
- part asset version
- model checkpoint
- evaluation code version

Make experiments reproducible.

---

# 22. Daily engineering log

At the end of each work session, update a concise log:

```text
Question:
What did we test?

Hypothesis:
What did we expect?

Experiment:
What changed?

Result:
What happened?

Failure mode:
What broke or remained uncertain?

R2S2R / world-model implication:
Why does this matter?

Next gate:
What must be verified next?
```

This log should become future interview material.

---

# 23. Recommended implementation order

Do the work in this order:

## Milestone 1
Dataset audit + trial visualization

## Milestone 2
Static grasp outcome baseline

## Milestone 3
Part/CAD mapping + metric scale

## Milestone 4
Object-relative grasp recovery

## Milestone 5
Grasp/TCP frame semantics

## Milestone 6
Deterministic Isaac replay

## Milestone 7
Real-vs-Sim baseline

## Milestone 8
Physics sensitivity

## Milestone 9
Calibration + domain randomization

## Milestone 10
Targeted real trajectory collection

## Milestone 11
Simulation trajectory dataset

## Milestone 12
State-based world model

## Milestone 13
Uncertainty

## Milestone 14
Grasp ranking / counterfactual evaluation

## Milestone 15
MPC

## Milestone 16
Visual latent world model

## Milestone 17
Optional WAM exploration

Do not reverse this order unless the existing company infrastructure already solves earlier milestones.

---

# 24. Stop gates

The agent must STOP rather than guess when any of the following are unresolved:

- coordinate-frame convention
- quaternion convention
- units
- object-ID-to-CAD mapping
- CAD metric scale
- object pose
- grasp/TCP relation
- approach axis
- closing axis
- collision representation
- simulator asset scale
- data timestamp alignment

Record the blocker and the evidence needed to resolve it.

A failed gate is a valid scientific outcome.

---

# 25. What NOT to do early

Do NOT start with:

- video diffusion
- huge transformer
- VLA
- WAM
- RL
- end-to-end policy training
- full synthetic data generation
- massive domain randomization
- thousands of simulation episodes

until the geometry and deterministic replay gates are correct.

---

# 26. Interview-oriented outputs

Throughout the project, maintain internal summaries that can later be converted into non-confidential interview explanations.

Do NOT include company secrets in interview materials.

Focus on generalizable lessons such as:

- visual fidelity vs physical fidelity
- task-relevant fidelity
- calibration vs randomization
- Real-to-Sim frame recovery
- TCP alignment
- simulation false positives
- grasp ranking
- model uncertainty
- rollout compounding error
- why closed-loop MPC matters

Suggested interview questions to be able to answer:

1. What is a world model?
2. How is a world model different from a VLA?
3. Why predict in latent space?
4. What is Real2Sim2Real?
5. What causes the Sim2Real gap?
6. Calibration vs domain randomization?
7. How do you validate a simulator against hardware?
8. How do you evaluate a world model for robotics?
9. Why is long-horizon rollout difficult?
10. Why use MPC with a learned world model?
11. How would you determine which simulation fidelity matters?
12. How would you use real failures to improve simulation?
13. How would you use a world model to rank grasps?
14. How would you quantify world-model uncertainty?
15. How would you design targeted real-data collection?

---

# 27. Immediate starting task

Start NOW from the existing company repository and current grasp dataset.

Do not implement later milestones yet.

First:

1. inspect existing code and data,
2. identify what parts of Milestones 1–6 already exist,
3. produce a gap analysis,
4. reuse all existing infrastructure,
5. identify the smallest next validation gate.

Create:

```text
outputs/master_plan/current_state.md
outputs/master_plan/gap_analysis.md
outputs/master_plan/next_gate.md
```

The `current_state.md` should answer:

```text
Dataset loading: READY / PARTIAL / NOT READY
Frame semantics: READY / PARTIAL / NOT READY
Part-to-CAD mapping: READY / PARTIAL / NOT READY
Metric scale: READY / PARTIAL / NOT READY
Object pose: READY / PARTIAL / NOT READY
T_object_grasp: READY / PARTIAL / NOT READY
TCP alignment: READY / PARTIAL / NOT READY
Isaac replay: READY / PARTIAL / NOT READY
Real-vs-Sim evaluation: READY / PARTIAL / NOT READY
Trajectory data: READY / PARTIAL / NOT READY
World-model dataset: READY / PARTIAL / NOT READY
```

Then propose exactly ONE next engineering task.

Do not jump ahead.

---

# 28. Definition of overall success

The full program is successful when we can demonstrate the following chain:

\[
\text{real observation + real grasp}
\]

→

\[
\text{verified object-relative grasp}
\]

→

\[
\text{deterministic simulation replay}
\]

→

\[
\text{real/sim outcome comparison}
\]

→

\[
\text{physics sensitivity + uncertainty}
\]

→

\[
\text{trajectory dataset}
\]

→

\[
\text{learned dynamics / world model}
\]

→

\[
\text{counterfactual grasp evaluation}
\]

→

\[
\text{closed-loop model-based planning}
\]

while maintaining:

- reproducibility
- clear provenance
- frame correctness
- no data leakage
- no proprietary data exfiltration
- explicit uncertainty
- task-level validation against real hardware

The first priority is correctness, not model complexity.
