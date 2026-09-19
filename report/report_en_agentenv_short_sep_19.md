# MitoBox on Kimi AgentENV: key results

*spark00: 20-core aarch64 DGX Spark, 119 GB unified memory*

## 1. AgentENV features and SWE-bench results

- [AgentENV](https://github.com/kvcache-ai/AgentENV) is the open-source sandbox service for Kimi K3's agentic RL. It provides one Firecracker microVM per sandbox through an E2B-compatible API and stores disk and memory images as copy-on-write layers.
- **Both restore methods use a pre-spawned VM** that loads memory pages on demand from one shared host copy.
    - **AgentENV warm start (S) restores a snapshot**, a checkpoint of a sandbox's memory and disk captured once in advance. Each rollout restores a new sandbox from it, replacing boot + task setup for states known beforehand, such as a GRPO group at step 0.
    - **AgentENV fork (F) captures a live sandbox** and restores N children while the source runs. It replaces prefix replay for states available only at runtime, such as a branch point in the middle of a trajectory (tree rollouts).
- **AgentENV cold start (C) boots a new VM** from the container image without a snapshot. Pause/resume releases RAM and CPU during LLM waits, but was not evaluated.

**Experiment**

- Each SWE-bench Verified django 10880 rollout has 5 steps. Each step is a 3 s LLM wait followed by one shell tool call. Step 4 applies the fix. B rollouts start together on 20 cores, with B = 1 to 256 and 3 rounds per point.
- **Docker (D) creates a new container per rollout.** After tests run once, `docker commit` preserves build products and caches. We snapshot AgentENV at the same point. Rollouts use `docker run`, one `docker exec` per tool call, then `docker rm`. This is the strongest container baseline, but images store files, not running processes.
- Tree rollouts run a trunk of 3 steps, with tests in the third, then B branches of 3 steps each. D and S first replay the trunk in every branch. F forks it, with 1 round at B = 256.
- LLM wait and reward time are fixed at 15 s and 3 s per rollout. As in most RL systems, LLM and reward computation run elsewhere: judges use GPU clusters, and test-based rewards use separate or forked evaluation sandboxes.
- We verify every rollout by its final diff hash. The official grader marked that diff resolved. Among 8,817 rollouts, only 9 cold starts failed to start their sandboxes.

![](agentenv_sep_19/figures/swe_scale.png)

*Figure 1. The left shows median rollout time by phase and totals for truncated bars. The right shows throughput relative to D for S at step 0 and F in tree rollouts.*

**Results**

- **For rollouts that start at step 0, D matches AgentENV within 4% up to B = 16.** Command-line (CLI) startup is short relative to LLM wait (Figure 1).
- **S benefits large batches only**, increasing throughput up to +19% over D. At large batches Docker's container creation and removal take longer than AgentENV's restore and deletion (Figure 1, left).
- **F matches S at step 0** up to B = 128, within 2 points. At B = 256, it gains less over D (+10% against +19%). We issued 4 sequential fork calls, each limited to 100 children. Tool calls were also slower in forked sandboxes (2.8 s against 1.8 s per rollout).
- **F improves tree rollout throughput by up to +139% over D.** Branches become ready sooner than with replay (Figure 1).
- **C is slower than Docker** and needs 80 GB at B = 256. It also uses the original, unprepared image (Figure 1).
- **Tool calls are slower on AgentENV.** At B = 128, latency is 0.12 s (p95 1.0 s), against Docker's 0.07 s (p95 0.13 s).
- In one probe, 511 of 512 CLI sandboxes ran in 22 GB after we raised the kernel's `ublk` device limit. The default limit of 64 allows about 60 sandboxes.

## 2. Blender computer-use agents (CUAs) on AgentENV

<img class="shot" align="right" width="33%" src="agentenv_sep_19/figures/blender_gui_firecracker.png">

- **AgentENV requires three changes on this host to run Blender's GUI.**
    - A 20-line server patch (`AENV_STATIC_CPU_CONFIG_PATH`) enables SVE/PAC features absent from AgentENV's default guest CPU. Without them, Blender's GUI crashes with an illegal instruction.
    - Since AgentENV never runs image entrypoints, a GUI template launches Xvfb, Blender and our screenshot/action server. The snapshot preserves these processes. A task snapshot with the scene loaded is the reset point.
    - A proxy client connects to the guest server through AgentENV's HTTP proxy.
- Firecracker has no display device, so the guest uses Xvfb and software OpenGL. The screenshot shows the 1280×800 PNG an agent sees 1 s after restore.
- **AgentENV warm start (S) and AgentENV fork (F) work unchanged** because they capture the whole VM without application participation. Readiness requires *Blender to answer*, not merely *the API to return*.

**Experiment**

- Each BlenderGym `placement1` rollout has 4 steps. Each step is screenshot → 3 s LLM wait → action → state check → 0.3 s settle. Steps 1 and 3 use mouse and keyboard input (click, A, G, X, "0.05", Enter). Steps 2 and 4 use `blender_python`.
- **Docker (D) creates a new container per rollout.** Its image stores Blender and task files, but no running Blender process. Each container boots Xvfb + Blender and loads the task.
- We tested B = 1 to 128, one round per point, and verified all 639 rollouts against expected object positions. Cold start was omitted because it adds VM boot to D's GUI boot.

![](agentenv_sep_19/figures/cua_scale.png)

*Figure 2. The layout and methods match Figure 1, without cold start.*

**Results**

- **S removes GUI boot time.** Throughput gains over D grow with batch size, reaching +125%. F performs the same (Figure 2).
    - Concurrent Docker GUI boots queue on 20 cores, delaying readiness, because each costs about 10 core-seconds (Figure 2).
- **Shared snapshot pages increase GUI sandbox density.** For 128 sandboxes, AgentENV needs 25 GB against Docker's 66 GB.

## 3. Discussion of the Blender results

- **CPU queueing lengthens the green phase in Figure 2** (screenshot + action + settle) as batches grow.
    - Without contention, in one sandbox with 4 vCPUs, a screenshot costs 0.05 core-seconds, a `blender_python` edit plus redraw costs 0.5 core-seconds, and a click plus 5 key events costs 2.2 core-seconds in 0.8 s of wall time. After every event, Blender redraws its whole 1280×800 interface using software OpenGL on all 4 vCPUs.
    - Rollouts alternate actions, averaging about 1.4 core-seconds per step and 5.6 core-seconds per rollout of 4 steps.
    - The fixed 3 s LLM wait synchronizes actions. At B = 16 the host CPU is briefly fully used each step and mostly idle between steps. From B = 64 it is fully used throughout the batch (Figure 3, right).
    - A simple queue model reproduces measured growth in step time: step time = B × 1.4 core-seconds / 20 cores (Figure 3, left).
    - Time until the action call's effect appears in Blender's state grows most (Figure 3, left). This is a GUI cost, not sandbox management. Docker's green phase also grows.
- **Any sandbox system's computed upper limit on 20 cores is about 210 rollouts/min**, given 5.6 core-seconds per rollout of 4 steps. Warm start reached 140 rollouts/min at B = 128. Restore costs about 1 core-second per sandbox, against about 10 for Docker GUI boot. The restore cost determines how much CPU remains for steps.
- **Warm start and fork help all GUI batches, but only large CLI batches.** The cause is the work that Docker must repeat for every rollout.
    - A CLI task's prepared state is files, which a Docker image stores. Containers are ready in 0.2 s, so Docker matches AgentENV up to B = 16 (Figure 1). Warm start and fork increase CLI throughput only from B ≥ 32.
    - A GUI task's prepared state is a running application and display server, which an image cannot store. Each container boots Xvfb + Blender and loads the task in 4.5 s and about 10 core-seconds. Restore takes 0.7 s and about 1 core-second. Docker is slower at every batch size. The difference grows with B because concurrent boots queue on the 20 cores (Figure 2).

![](agentenv_sep_19/figures/gui_step_cost.png)

*Figure 3. The left shows mean GUI step times by component for AgentENV warm start (S) and the CPU queue model. The right shows host CPU use for S at B = 16 and B = 64.*
