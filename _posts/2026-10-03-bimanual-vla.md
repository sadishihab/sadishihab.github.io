---
title: "Teaching Two Robot Arms to Set a Table: MuJoCo, ACT, and OpenVINO, Measured End to End"
date: 2026-10-03
layout: post
permalink: /bimanual-vla/
categories: [Robotics, AI, Machine Learning]
tags: [Robotics, Imitation Learning, ACT, LeRobot, MuJoCo, OpenVINO, Quantization, Vision-Language-Action, SO-101, PyTorch, Edge AI]
description: "How I built a bimanual table-setting pipeline with two SO-101 arms in MuJoCo: a scripted expert, a 47,000-frame LeRobot dataset, an ACT policy with language conditioning, and OpenVINO INT8 deployment. Every number measured, every failure explained."
author: "Md. Shihabuddin Sadi"
---

*Software Engineer · DevOps & Cloud Native Engineer · AI / RAG Application Developer*  
*October 03, 2026*  

<br>

### **Summary**

Here is the headline number from this project: **my trained robot policy places 3 of 40 props.**

That is not the number most people would put at the top of a blog post. I am putting it there on purpose, because the most valuable thing this project produced is not a demo video. It is a pipeline where every stage is measured, every failure has a mechanism attached to it, and nothing is tuned away until it is understood.

The task: **two SO-101 robot arms set a table in MuJoCo.** A scripted expert picks four props (plate, fork, spoon, mug) out of a randomized layout and arranges them into a place setting, handing a prop from one arm to the other when no single arm can reach both the prop and its slot. Successful runs are recorded as a **LeRobot v3.0 dataset**, an **ACT (Action Chunking Transformer)** policy is trained on it, evaluated back in the same scene, extended with **language conditioning**, and finally converted to **OpenVINO IR** and quantized to **INT8** for Intel inference hardware.

Along the way:

- Pick success went from **57.5% to 92.5%**, each fix driven by a measurement
- An **in-air handover was proven geometrically impossible** in this scene, so it was replaced, not faked
- A placed prop lands a median **1.6 mm** from its target slot
- A **controlled experiment** isolated one of three causes behind the policy's weak performance
- A quantization parity check that **looked perfect on synthetic data was off by three orders of magnitude on real frames**, and was caught before it shipped

🔗 **GitHub Repository:** [sadishihab/bimanual-vla](https://github.com/sadishihab/bimanual-vla)

<br>

### Table of Contents

- [Summary](#summary)
- [Key Technologies Used](#key-technologies-used)
- [1. Why Table Setting Is Hard](#1-why-table-setting-is-hard)
- [Architecture Diagram](#architecture-diagram)
- [2. Folder Structure](#2-folder-structure)
- [3. The Gripper Was Lying](#3-the-gripper-was-lying)
- [4. Reach Is Not Where the Arm Can Touch](#4-reach-is-not-where-the-arm-can-touch)
- [5. Pick Success: 57.5% to 92.5%](#5-pick-success-575-to-925)
- [6. Proving a Handover Impossible](#6-proving-a-handover-impossible)
- [7. The Full Task, and a Fix That Made Things Worse](#7-the-full-task-and-a-fix-that-made-things-worse)
- [8. The Dataset](#8-the-dataset)
- [9. Training ACT on a Kaggle T4](#9-training-act-on-a-kaggle-t4)
- [10. Closed Loop: 3 out of 40, and Why](#10-closed-loop-3-out-of-40-and-why)
- [11. Language Conditioning as a Controlled Experiment](#11-language-conditioning-as-a-controlled-experiment)
- [12. OpenVINO: The Parity Check That Nearly Shipped](#12-openvino-the-parity-check-that-nearly-shipped)
- [13. Latency, and Being Honest About Hardware](#13-latency-and-being-honest-about-hardware)
- [14. Engineering Rules That Kept It Sane](#14-engineering-rules-that-kept-it-sane)
- [15. Learning Outcomes](#15-learning-outcomes)
- [References](#references)
<br>

### **Key Technologies Used**

`Python` · `MuJoCo` · `SO-101 arms` · `Damped least-squares IK` · `LeRobot 0.4.4` · `ACT (Action Chunking Transformer)` · `PyTorch` · `ResNet18` · `MiniLM sentence embeddings` · `Kaggle T4 GPU` · `OpenVINO 2026.3` · `NNCF INT8 quantization`  
<br>

### **1. Why Table Setting Is Hard**

Setting a table sounds like a toy problem. It is not, for three reasons.

**It is bimanual.** Two arms share one workspace. They split the table at the midline, so a fork lying on the left side whose slot is on the right must be passed from one arm to the other.

**It is multi-object and multi-step.** Four props, each with a different shape and grasp, picked and placed in sequence. Errors compound across every pick and every place.

**It is physical.** Contact, friction, stiffness, and torque all matter. A grasp that looks fine in a diagram can drive a fork into the table, or flick a spoon across the room.

The project runs the full modern robot-learning loop: **simulate, script an expert, record demonstrations, train an imitation policy, evaluate in closed loop, and deploy to edge hardware.** The rule throughout: where something fails, record the mechanism and the number. Do not tune the failure away.

The pipeline has five stages:

- **Scene and expert** – a MuJoCo scene with two SO-101 arms, seeded domain randomization, IK, and pick/place/handover primitives  
- **Recorder** – captures successful episodes as a LeRobot v3.0 dataset without modifying the expert  
- **Policy** – ACT trained on two camera views plus joint state, later extended with a language token  
- **Evaluator** – runs the policy closed loop in the same scene and scores it with the same test as the expert  
- **Deployment** – OpenVINO IR conversion, INT8 quantization, and a device-agnostic latency benchmark  
<br>

#### **Architecture Diagram**
```pgsql
 MuJoCo scene ─► scripted expert ─► record_demos.py ─► pack_lerobot.py ─► LeRobotDataset v3.0
                       │                                                        │
                       │                             ACT (lerobot 0.4.4) ◄──────┘
                       │                                    │
                       └──────────► eval_policy.py ◄────────┤   closed loop, same scene
                                                            │
                         convert.py ─► FP32 / FP16 IR ─► quantize.py ─► INT8 IR
                                                            │
                                                     benchmark.py ─► CPU / GPU / NPU
```
<br>

### **2. Folder Structure**
```bash
├── scenes/
│   └── bimanual_table.xml          # Dual SO-101, table, overhead + front cameras
├── envs/
│   ├── scene.py                    # Loads the scene, converts the gripper to force control
│   ├── randomize.py                # Seeded domain randomization and the reach map
│   └── task.py                     # Place-setting goal; which props cross the midline
├── control/
│   ├── ik.py                       # Damped least-squares IK for one arm
│   ├── gripper.py                  # Gripper force control and the measured jaw pads
│   ├── primitives.py               # Pick, place, handover, and their planners
│   └── language.py                 # Task-string conditioning as an extra ACT token
├── scripts/                        # Per-seed runners, sweeps, recorder, evaluator
├── notebooks/
│   └── train_act_kaggle.ipynb      # ACT training on Kaggle
└── openvino/                       # IR conversion, INT8 quantization, device benchmark
```
<br>

### **3. The Gripper Was Lying**

Almost every design decision in this project traces back to four facts about the SO-101 gripper that I only learned by measuring it.

**❌ The collision hull is not the finger**

MuJoCo collides a mesh as its **convex hull**, and both SO-101 jaws are concave. On the fixed finger, the hull bridges from the wrist mount to the fingertip as a single **1428 mm² facet tilted 21.7°**. Of its 606 facets facing the opening, **not one is within 5° of parallel.**

So every grasp was really a wedge. It touched only the top edge of the object and pushed it down: **2.07 N into the table against a 0.30 N fork.** And because the ramp moves with the gripper, no choice of grasp height could fix it.

The fix was not to invent geometry. The real inner faces underneath the hull *are* flat. Fitting planes through the mesh vertices gave faces **0.23° and 0.24°** off the opening axis with a **0.33 mm residual**, **15.80 mm apart**. Each jaw got an explicit collision pad placed exactly on the face that was already there.

**❌ The tool site is not the middle of the jaws**

The tool site sits on the *fixed* finger. Centre it on an object and you drop a finger straight onto it. Grasps are offset by half the object's width plus 4 mm of clearance: 10 mm for flatware, **22 mm for the mug.** That offset came back as a bug three more times.

**❌ Position control cannot hold a grasp**

A commanded jaw angle relaxes as the servo settles, and the prop falls out mid-lift. So the gripper is **torque-controlled** with three values for three jobs:

```python
OPEN  = +1.0   # N·m, run the jaw to its stop
GRIP  = -0.8   # N·m, the closing sweep
HOLD  = -0.20  # N·m, what stays on during the carry
```

Getting `HOLD` wrong fails in both directions. Measured on an 11 g spoon: **0.35 N·m ejects it after one lift sub-step out of twelve. 0.20 N·m carries it all the way.** Too much grip is as fatal as too little, because the jaws close past parallel and flick light objects out.

**❌ The props had to be redesigned for the gripper**

The graspable band turned out to be **12 to 88 mm above the table**, with a top-down reach ceiling of **93 mm.** A full-size plate cannot be grasped by a 46 mm gripper at all. So the plate became a 34 mm footed dish, the mug a 36 mm barrel, and the flatware was shortened from 136 mm to **65 mm**.

That last one is a nice piece of physics. At 136 mm, lifting the handle just pivoted the fork about its far end: measured at **−39° to −42° with the tines still on the table while the handle was 42 to 46 mm up**, which is all the lift the arm has. Clearing a piece held near one end takes a lift on the order of its own length. The ceiling is 93 mm, so the fork came down to the lift, rather than the lift going up to the fork.  
<br>

### **4. Reach Is Not Where the Arm Can Touch**

Reach is modelled as a **per-arm, orientation-aware occupancy grid** on a 10 mm cell. A cell counts as reachable only if the arm can put its gripper there **pointing straight down** *and* **hold that pose statically**: inverse dynamics at zero velocity, every joint torque inside its actuator's limits.

```python
# envs/randomize.py (conceptually)
reachable = (
    ik_converges(cell, approach="down")       # position AND orientation
    and static_torques_within_limits(pose)    # can it actually hold it?
)
```

Position-only reach is far too generous. A pose the arm can touch but not hold is not a pose you can grasp from.

IK is **damped least squares** over five joints. Five joints cannot generally hit a six-DOF target, so orientation is down-weighted and **the residual is reported, not assumed away.**

One bug here is worth calling out. The reach map was indexed by where the *prop* sits, but what must be reachable is where the *tool site* goes, up to 22 mm away for the mug. Ignoring that made the goal layout promise slots the arms could not release at: on seed 0 it gave the mug a slot its receiving arm missed by **7.5 mm.** The fix was a **translation, not a safety margin.** Eroding the map by a disk instead, which is what "stay clear of the edge" suggests, left **all ten seeds with no legal setting at all.**  
<br>

### **5. Pick Success: 57.5% to 92.5%**

Four props, ten seeds, one process per case. A pass requires more than 30 mm of rise, no table contact, and the prop still held.

|        | plate | fork  | spoon | mug  | overall           |
|--------|-------|-------|-------|------|-------------------|
| before | 10/10 | 3/10  | 2/10  | 8/10 | **23/40 = 57.5%** |
| after  | 10/10 | 10/10 | 9/10  | 8/10 | **37/40 = 92.5%** |

Three fixes, each from a measurement:

1. **Clamp every waypoint into the reach envelope.** Commanding a point outside it does not just miss. The servos saturate and the arm flies a different path than the one requested. Measured on the spoon, sub-step spacing collapsed from 4.2 mm to 3.0 mm as the arm ran out of reach, and the sideways motion sheared the spoon out of the jaws.
2. **Route the transit over the props and subdivide it.** The tool bows off a straight commanded line, and at 12 sub-steps it still dipped onto the prop it was about to pick. 36 sub-steps fixed it. Demanding the final grasp yaw too early swung the tool **39 mm below the line, straight through the fork.**
3. **Shorten the flatware and ease the squeeze once the jaws are loaded.**

The three remaining failures are honest ones: two mug transit knocks and one dropped spoon.  
<br>

### **6. Proving a Handover Impossible**

The arms split the table at y = 0, so some props must change hands. The natural design is an **in-air handover**: one arm holds the prop, the other grabs it, the first lets go.

I searched for one. It does not exist in this scene, and the proof is simple geometry:

- Across its lateral axis, the gripper's collision hull spans **52.0 mm.** Two grippers at the same yaw must be at least that far apart to clear each other. Flipping one end-for-end only reduces it to **48.4 mm**.
- The longest grasp feature in the scene is the mug's **36 mm.**

**52 mm of gripper against 36 mm of object.** Two arms cannot both hold the same prop without overlapping. Searching over both arms' yaws independently, with grasps stacked up the mug's barrel, the best case still interpenetrates by **14.8 mm** for flatware and **27.5 mm** for the mug. The only collision-free pose that turned up places both jaws tangent to the mug, with **0.0 mm** of material between them to close on.

So the handover is **mediated by the table**: the giving arm sets the prop down where both arms can reach, and the receiving arm picks it up. The impossibility search is **kept in the code as a reported measurement**, not deleted. Every call to `handover()` still reports the shallowest clash it found, so anyone who doubts the decision can see the evidence.  
<br>

### **7. The Full Task, and a Fix That Made Things Worse**

Full task, four props × ten seeds. A prop passes only if it ends within 20 mm of its slot **and resting on the table.** A prop still in the jaws does not count, wherever it happens to be.

|                    | props placed      | complete settings | handovers |
|--------------------|-------------------|-------------------|-----------|
| scripted expert    | **24/40 (60.0%)** | 0/10              | 2/8       |

The interesting part is *where* it fails. **Placement accuracy is not the problem.** A placed prop lands a median **1.6 mm** from its slot, worst case 5.8 mm, against a 20 mm tolerance. All 16 failures are upstream: props set down short at the transfer spot, flatware shed mid-carry, picks that never lift, and release poses the arm correctly refuses rather than dropping the prop.

**The tolerance budget.** The fork originally finished its handover **17.9 mm** off target, dangerously close to the 20 mm limit, because error compounds across the extra pick/place cycle. I traced it to four separate accumulators: grasping 7.23 mm off the centre of mass, a stale tool-to-prop offset measured before the carry, release height computed from the body origin instead of the lowest tine, and a bounding sphere that understated the load's hang by up to 9.2 mm. Fixing all four brought it to **4.1 mm.**

**The intervention that hurt.** I had added a "re-grip" that fired mid-carry when grip force faded. It sounds sensible. Measured over the sub-steps where the spoon's grip had faded:

| re-grip strategy             | spoon put down  |
|------------------------------|-----------------|
| re-squeeze at closing torque | **84.3 mm** off |
| re-seat at holding torque    | **96.0 mm** off |
| do nothing                   | **6.0 mm** off  |

Doing nothing was **14× better.** The re-grip is now a counter, not an intervention. This is the kind of result you only get if you measure the "obvious improvement" instead of assuming it.

The largest remaining failure, flatware shed on long carries, is **recorded as unresolved**: holding torque cannot go up without ejecting the spoon, and re-gripping makes it worse. It is a genuine limit of a parallel jaw on a 12 mm bar, documented rather than hidden.  
<br>

### **8. The Dataset**

| | |
|---|---|
| episodes | **134** |
| frames | **47,092** at 25 Hz (31.4 minutes) |
| seeds | 49 of 50 contributed |
| with a handover | 20 |
| placement error | median 1.6 mm |
| format | **LeRobotDataset v3.0**, loads with no conversion |

Observations are two cameras (`overhead`, `front`) at 256×256 plus 12 joint positions. The action is the 12 actuator commands.

Two design decisions matter here.

**Recording without touching the expert.** Capture is a hook on `mujoco.mj_step`, the one call every primitive already steps through. The expert code was never modified to record, so the recorder cannot change the behaviour it records. State and action are sampled *before* each step, so every pair is a command and the state it was applied to.

**The action vector mixes units.** Ten channels are joint targets in **radians**; the two gripper channels are torque in **N·m**, a direct consequence of the force-controlled gripper. LeRobot normalizes per channel, so the units are never pooled, but the training notebook **asserts this twice** (on the stored statistics and on a real batch) rather than assuming it.  
<br>

### **9. Training ACT on a Kaggle T4**

| | |
|---|---|
| policy | ACT, lerobot 0.4.4, **51.6 M** parameters |
| backbone | ResNet18 ×2, one per camera |
| transformer | d_model 512, 8 heads, 4 encoder + 1 decoder layers, VAE latent 32 |
| action chunk | 100 actions = **4.0 s** at 25 Hz |
| hardware | one 16 GB T4 on Kaggle, batch 16, fp16 with `GradScaler` |
| result | **44,000 steps, final loss 0.09** |

The notebook is written to survive Kaggle's session limits instead of fighting them. Checkpoints go to a temp directory and are **atomically renamed**, so a killed session can never leave a half-written checkpoint. Training resumes from the newest one, and a wall-clock budget stops and checkpoints cleanly rather than being cut off mid-step. The same habits I use for production jobs on Kubernetes apply to a free GPU notebook.

One detail in that result turned out to matter a lot: **total loss 0.09 with L1 0.09 means the KL term had collapsed to roughly zero.**  
<br>

### **10. Closed Loop: 3 out of 40, and Why**

The evaluator runs the policy in the same scene, its 12 outputs driving the actuators directly, scored by exactly the same test as the expert.

|                 | props placed                         |
|-----------------|--------------------------------------|
| scripted expert | **24/40 (60.0%)**                    |
| ACT, 44k steps  | **3/40 (7.5%)**, of which **2 earned** |

Why "2 earned"? On one seed, the spoon's target slot happened to land 9.8 mm from where the spoon already lay. The policy never touched it, but it technically passes. That is **reported separately and not counted**, because an untouched pass is not evidence the policy can place anything.

The policy is not random. Its arms move purposefully, it contacted props 11 times and lifted 5. But **every meaningful interaction was with the plate.** The fork and the mug were never touched in any of ten seeds.

I diagnosed three causes, all visible in how the data was built:

- **No task conditioning.** ACT sees images and joint positions. It cannot be told which prop to move.
- **VAE collapse.** With KL ≈ 0, the latent carries nothing, so ACT regresses the *average* of the demonstrations. Faced with four props, two arms, and handover-or-not, the average is a single behaviour.
- **Plate class imbalance.** The expert always takes the plate first, and the plate is **47 of 134** episodes. From the opening observation, the demonstrated action is nearly always "go for the plate."

Add a 4-second open-loop action chunk, and success-only demonstrations that never show a recovery, and nothing recovers once the plate is fumbled. **The policy learned the opening move and little else.** That is a data and conditioning problem, not a "train it longer" problem, and knowing the difference saves weeks of GPU time.  
<br>

### **11. Language Conditioning as a Controlled Experiment**

The first diagnosed cause is the most direct to attack: give the policy a sentence like *"put the fork in its place."*

**Adding language without forking LeRobot.** LeRobot 0.4.4's ACT has no language path. But ACT already accepts one extra 1-D observation vector, `observation.environment_state`, with its own linear projection and positional embedding. A sentence embedding is exactly that shape. So the language token goes in through that existing slot and **the library is untouched.**

```python
# control/language.py (conceptually)
# Frozen MiniLM-L6 encoder → mean-pool → L2-normalize → 384-dim vector
emb = TASK_EMBEDDINGS[task_string]            # 7 strings, embedded once
batch["observation.environment_state"] = emb  # injected, not stored in the dataset
```

- The encoder (**22.7 M parameters**) is frozen. Only the **384→512 projection is learned: 197,120 parameters** out of 51.8 M.
- The embedding is injected into the batch at load time, so the **dataset did not need repacking.**
- The result is an ordinary ACT checkpoint that loads with `from_pretrained`, so the **OpenVINO pipeline still works.** A conditioned IR simply gains a fourth input of shape `(1, 384)`.

**v1: the language was wired in, and ignored.** Directed evaluation gives the same scene with a different instruction per episode, so any behaviour change must come from the sentence. Result: the same 3/40, and in all 30 episodes asking for the fork, spoon, or mug, the policy went for the plate.

The reason was a **confound**. The expert always worked in a fixed order, so the table state in the image already told the policy which prop was next. It never needed the sentence.

**v2: remove the confound and re-test.** I changed the recorder to work the props in a **seeded random order** and re-recorded 132 episodes. Measured on the data itself, predicting the target from the opening table state alone fell from **100.0% to 54.5%**. Same architecture, same schedule, only the data changed.

|                                    | props placed | non-plate target contact | lifted |
|------------------------------------|--------------|--------------------------|--------|
| unconditioned                      | 3/40         | n/a                      | 5/40   |
| conditioned v1, fixed-order data   | 3/40         | **0/30**                 | 5/40   |
| conditioned v2, shuffled-order data| 2/40         | **7/30**                 | 1/40   |

**Language now measurably redirects attention**, from 0/30 to 7/30, with fork, spoon, and mug each contacted on their own instruction. **But task success did not improve.** Most new contacts are shallow touches, not grasps.

That is exactly what the diagnosis predicted. De-confounding fixes *one* of three causes. The plate is still the dominant class (48 of 132 episodes) and the VAE latent is still collapsed. A policy that now knows *which* prop it was asked for, but was never taught a reliable grasp for three of the four, should look exactly like this: broader, shallower engagement.

I report this as **one cause isolated and confirmed through a controlled before/after experiment**, not as a feature that "worked" or "failed." That framing is the difference between a research result and a demo.  
<br>

### **12. OpenVINO: The Parity Check That Nearly Shipped**

The trained checkpoint was converted to OpenVINO IR at three precisions and compared against PyTorch on **real dataset frames**:

| precision | size          | mean abs error | gripper sign flips |
|-----------|---------------|----------------|--------------------|
| FP32      | 131.3 MiB     | 0.00000        | **0.00%**          |
| FP16      | 66.1 MiB      | 0.14223        | **14.94%**         |
| INT8      | **38.0 MiB**  | 0.0161         | **0.92%**          |

The key metric is **gripper sign flips**. A sign flip on a gripper channel turns *close* into *open*, which drops the prop mid-carry. A mean error hides it completely.

The verdict: **ship FP32, INT8 is defensible with a closed-loop check, and FP16 must not be deployed on this model.**

**This nearly went the other way.** The first parity check used one **synthetic uniform-noise frame**, where FP16 looked excellent at **1.2e-03** error. On real frames the same IR was off by **1.9e+00**, three orders of magnitude worse. Random noise sits far outside the training distribution, and the network answers it from a flat part of its response, so a badly broken precision can look exact. `convert.py` now validates against real frames, reports error per unit group (radians vs N·m), and warns loudly.

I also ruled out the obvious explanations for FP16's error: rounding the weights to fp16 in PyTorch costs only **0.0056**, execution precision and overflow were checked and excluded, and the converter itself was verified. What remains is inside OpenVINO's compression pass, documented as an open item rather than guessed at.

A few deployment details that matter on real edge hardware:

- **INT8 calibration** uses 300 real frames sampled evenly across all 47,092 (`np.linspace`, not the first N, which would all come from one episode).
- The IR is **self-contained**: normalization in, un-normalization out. It takes raw observations and returns actions in real units, so per-channel statistics cannot drift at the edge.
- **Every shape is static**, because the NPU plugin rejects dynamic shapes.  
<br>

### **13. Latency, and Being Honest About Hardware**

Benchmarks ran on the only Intel hardware I had: a **2015 Core i7-5500U (Broadwell)**, CPU only. I could not get access to Core Ultra hardware, and Intel's cloud service did not accept personal accounts.

| device | precision | size MiB | mean ms | p95 ms     |
|--------|-----------|----------|---------|------------|
| CPU    | fp32      | 131.3    | 156.38  | **182.69** |
| CPU    | fp16      | 66.1     | 163.67  | **182.51** |
| CPU    | int8      | 38.0     | 172.75  | **199.26** |
| GPU    | —         | —        | —       | not present |
| NPU    | —         | —        | —       | not present |

Two honest findings:

**INT8 is 3.46× smaller but slightly slower here.** That is correct, not a bug. Broadwell has no VNNI instructions, so INT8 math has no hardware fast path and the quantize/dequantize nodes are pure overhead.

**None of them meet the control budget.** Demonstrations were recorded at 25 Hz, which allows 40 ms per step. The best p95 is 182 ms. That is a statement about a 2015 laptop CPU, not about the architecture.

What I could do was make the benchmark ready for the right hardware. `benchmark.py` reads `Core().available_devices` instead of assuming any device, benchmarks whatever it finds, and reports missing devices as missing. It imports only `openvino` and `numpy`, so it **runs unmodified on a Core Ultra machine** and fills in the GPU and NPU rows. Expectations for that hardware are written down in the repo, clearly labelled as expectations and not measurements.

Latency is reported as mean, p50 **and p95**, because the tail is what breaks a control loop.  
<br>

### **14. Engineering Rules That Kept It Sane**

Robotics simulation is unforgiving about state and memory. Two rules run through the entire repo:

1. **One seed per process.** Sweep loops live in shell scripts, so every case gets a fresh interpreter and a fresh model. No state leaks between seeds, and a crash in one case cannot poison the rest.
2. **No renderer unless the output genuinely needs pixels.** Memory was the recurring failure, so only the recorder and the closed-loop evaluator ever build one, and both free it.

```bash
# One seed of the whole task
python scripts/run_task.py --seed 0

# Full-task sweep: one process per case, parallelised in the shell
JOBS=4 scripts/sweep_task.sh && python scripts/sweep_task_report.py

# Deployment pipeline
python openvino/convert.py  --checkpoint checkpoints/step_0044000
python openvino/quantize.py --ir openvino/ir/act_fp32.xml
python openvino/benchmark.py
```

Environments are separated deliberately: the simulation venv holds only MuJoCo and NumPy, training and conversion use a LeRobot environment, and the benchmark needs nothing but the OpenVINO runtime. Dependency pins are documented with reasons. For example, `transformers` is pinned below 5 because version 5.17 broke `import lerobot.policies` outright.

And the most important rule of all: **every claim in the README is labelled as measured, built, run, or not measured.** The one thing the repo explicitly refuses to claim anywhere is NPU performance, because it was never measured.  
<br>

### **15. Learning Outcomes**

- Built a **full robot-learning pipeline**: simulation, scripted expert, dataset, imitation policy, closed-loop evaluation, and edge deployment  
- Diagnosed **simulation physics issues** (convex hulls, contact stiffness, force control) through measurement instead of trial and error  
- Designed an **orientation- and torque-aware reach envelope** that reflects what an arm can actually hold  
- Used **geometric analysis to rule out an in-air handover**, and kept the proof in code  
- Recorded a **47,092-frame LeRobot v3.0 dataset** with a non-invasive capture hook  
- Trained a **51.6 M parameter ACT policy** on a free Kaggle T4 with crash-safe checkpointing  
- Diagnosed weak policy performance into **three concrete causes** rather than "needs more training"  
- Added **language conditioning without forking LeRobot**, and validated it with a **controlled de-confounding experiment**  
- Converted and **quantized to OpenVINO INT8** with real-data calibration and per-unit error reporting  
- Caught a **parity check that was three orders of magnitude optimistic** before it shipped  
- Wrote a **device-agnostic benchmark** ready for CPU, GPU, and NPU, with honest reporting of what was not measured  

<br>

### **Final Thoughts**

A 3/40 success rate with a clear, evidence-backed explanation is more useful than a cherry-picked video of the one run that worked. In robotics, as in production AI systems, the expensive failures are the ones nobody measured. This project is a record of measuring them.

If you are working on robot learning, edge AI deployment, or any AI system where "it looks like it works" is not good enough, I would love to talk: [book a 30 minute call](https://calendly.com/sadi-shihab/30min) or find me on [LinkedIn](https://www.linkedin.com/in/md-shihabuddin-sadi).

<br>

### **References**
- [bimanual-vla on GitHub](https://github.com/sadishihab/bimanual-vla)
- [Learning Fine-Grained Bimanual Manipulation with Low-Cost Hardware (ACT paper)](https://arxiv.org/abs/2304.13705)
- [LeRobot by Hugging Face](https://github.com/huggingface/lerobot)
- [SO-ARM100 / SO-101 Robot Arm](https://github.com/TheRobotStudio/SO-ARM100)
- [MuJoCo Physics Engine](https://mujoco.org)
- [OpenVINO Documentation](https://docs.openvino.ai)
- [NNCF: Neural Network Compression Framework](https://github.com/openvinotoolkit/nncf)
- [all-MiniLM-L6-v2 Sentence Transformer](https://huggingface.co/sentence-transformers/all-MiniLM-L6-v2)
