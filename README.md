# Surprise-Gated Event Memory for Fast World-Model Planning

**Working title.** Alternative: *Remember What Surprises You: Lean Memory for Real-Time World-Model Planning*

| | |
|---|---|
| Status | Planning stage. Project started 27 Oct 2026. |
| Author | Vinod Anbalagan |
| Target | Complete arXiv preprint before mid-January 2027 application deadlines `[TBD: exact earliest deadline date]`; real-robot demo in Feb to Mar 2027 |
| Convention | Anything marked `[TBD: ...]` is unconfirmed. It is a placeholder, not a value. |

---

## 1. The idea in one paragraph

Most state-of-the-art robot foundation models have no memory. They act on the current camera frame only, which works for short or repetitive tasks but fails on long tasks where the scene does not show how far along you are (Finn, YC talk, 2026). Memory is expensive when done naively: 10 seconds of video from four cameras at 50 Hz is about half a million tokens, and even 1 frame per second is about 10,000 tokens (same talk). Physical Intelligence's answer, MEM, uses compressed video for short-term memory and LLM-written text summaries for long-term memory (Torne et al., 2026).

**Our question:** what is the *cheapest* memory that makes a fast latent world-model planner reliable on tasks where the current frame does not reveal task progress?

**Our proposal:** the planner runs memoryless by default (fast, "Type 1"). It writes to a small, fixed-size memory only when its own world model is surprised, meaning its prediction of the next state was wrong. It reads that memory when planning (slower, "Type 2", used only when needed). No video buffer, no LLM, constant memory size, constant cost.

---

## 2. Research question and hypotheses

**RQ.** Can a world-model planner with surprise-gated event memory match the reliability of full-history memory while keeping planning speed close to a memoryless planner?

| ID | Hypothesis | What would falsify it |
|---|---|---|
| H1 | A memoryless latent world-model planner fails on memory-requiring tasks. | It succeeds anyway, meaning our tasks do not actually need memory (then the tasks must be redesigned). |
| H2 | Surprise-gated event memory reaches success within `[TBD: margin, e.g. 5 points]` of full-history memory, at planning latency within `[TBD: e.g. 1.5x]` of memoryless. | Gated memory is clearly worse than full history, or no faster. |
| H3 | Writing on the world model's *own* action-conditioned prediction error beats writing at fixed intervals or at random, at the same memory budget. | Fixed-interval or random writes do as well. |
| H4 (stretch) | The same mechanism transfers to a second embodiment (e.g. simulated quadrotor) without redesign. | It needs embodiment-specific tuning to work. |

H1 is a sanity check. H2 and H3 are the paper. H4 is a bonus.

---

## 3. Why this is a gap: prior work and positioning

| Work | Memory | Memory used for | Speed reported | How we differ |
|---|---|---|---|---|
| **MEM** (Torne et al., 2026, Physical Intelligence) | Compressed video (short-term) plus LLM-written text summary (long-term) | VLA control, tasks up to 15 min | Not the focus | No video, no LLM; a world-model planner, not a VLA |
| **Worth Remembering** (Gorlo, Wise, Speranzon, Carlone, 2026, MIT and Lockheed Martin) | Surprise-gated episodes (8 frames each), surprise from a Gaussian fit over V-JEPA-2 features | Question answering by an LLM agent | **No** latency reported | Surprise from the planner's *own action-conditioned* prediction error (they list an action-conditioned predictor as future work); memory used for **control**; speed is a primary metric |
| **Mem-World** (Zheng et al., 2026) | 4D surfel memory of wrist views | Video world model for policy evaluation and data generation, not planning | No latency in main text; trained ~2 days on 8 H100s | Lean latent memory, used inside a planner |
| **ERA** (Fan, Zhang, Han, 2026) | Case bank of (event code, maneuver, reliability) | Retrieval controller for UAV navigation; ~130 Hz on Jetson Orin Nano | Yes | Events are hand-defined (distance threshold); ours are triggered by learned surprise |
| **EV-WM** (Wang et al., 2026) | No dedicated memory module | Verifying imagined plans via structured event states | No | Complementary: verification, not memory |
| **LeWorldModel** (Maes, Le Lidec, Scieur, LeCun, Balestriero, 2026) | None | Latent planning from pixels; ~15M params; trains on one GPU in a few hours; planning up to 48x faster than DINO-WM | Yes | **Our base model.** Its benchmark tasks do not require memory |
| **Fast-LeWM** (Gao, Xu, 2026) | None | Parallel latent rollout; ~2.1x faster CEM solve than LeWM | Yes | Possible faster base; code is open |

**Positioning sentence (draft):** *Prior work gates memory on surprise for offline question answering, or adds heavy memory to video world models for evaluation. We put a constant-size, surprise-gated memory inside a real-time latent world-model planner and measure the trade-off between reliability and planning speed directly.*

Also relevant: **The Planning Limits of Latent World Models** (Alrasheed, Azam, Akhtar, 2026) finds a world model guides action selection reliably only when the goal lies within or slightly beyond its imagined trajectory. Long memory tasks will need subgoals or a short planning horizon. See Risks.

---

## 4. Method sketch

```
frame o_t ──► encoder ──► z_t ──┐
                                 ├──► predictor(z, a, memory) ──► ẑ_{t+1}
memory M (K slots) ─────────────┘
                                         │
planner (CEM / MPPI) ◄── cost uses M ────┘

surprise s_t = || ẑ_t − z_t ||   (prediction made one step ago vs what actually happened)
if s_t > threshold  →  write z_t (plus small context) into M
```

1. **Encoder:** LeWM's encoder (ViT-Tiny, ~5M params), trained end to end as in LeWM.
2. **Predictor:** LeWM's action-conditioned predictor, extended to cross-attend to the memory slots.
3. **Event memory:** K fixed slots `[TBD: K, start with 4 to 16]`. Eviction rule `[TBD: FIFO vs lowest surprise]`.
4. **Surprise gate:** the world model's own one-step prediction error, with an adaptive threshold. We start from the robust median-plus-MAD style threshold used by Gorlo et al. and add habituation, because they report that recurring surprising subjects keep triggering writes.
5. **Typed fast heads (stretch, inspired by "System One" models such as TypeSafe's Jev):** small heads that answer atomic questions in one pass, e.g. "is this subtask done?" and "how confident am I?". They could replace or refine the surprise gate. Not needed for the core paper.

**Type 1 / Type 2 reading:** most steps never touch memory beyond a cheap attention read over a few slots. Memory is written only at surprising moments, so it stays small and does not bloat.

---

## 5. Experiments

### 5.1 Tasks (memory-requiring variants of LeWM's environments)

LeWM evaluates on Push-T, OGBench-Cube, Two-Room and Reacher, none of which need memory. We modify them so that the current frame does not reveal what to do next:

| ID | Task | Why the current frame is not enough |
|---|---|---|
| T1 | **Cue then act** (Two-Room): a goal cue is visible only for the first steps, then disappears | The agent must remember which goal was shown |
| T2 | **Identical items** (OGBench-Cube style): push each of N identical cubes to a zone exactly once | Processed and unprocessed cubes look the same |
| T3 | **Hidden object**: object moves behind an occluder | Its location must be remembered |
| T4 | **Perturbation**: an object is displaced mid-task | Memory must be *updated*; tests whether surprise catches the change |
| C | **Original LeWM tasks** (control) | Checks that memory does not hurt tasks that do not need it |

`[TBD: final task list after Week 2; pick at least 2 of T1 to T4 plus control]`

### 5.2 Baselines

- No memory (plain LeWM)
- Frame stacking (last k latents)
- Full history (all past latents, attention)
- Fixed-interval writes, same K
- Random writes, same K
- Oracle memory (ground-truth task state), as an upper bound

### 5.3 Metrics

- Task success rate (mean and spread over `[TBD: seeds, at least 3]` seeds)
- Planning latency per decision (ms) and rate (Hz), on stated hardware
- Memory footprint (slots, bytes) and writes per episode
- Recovery after perturbation (T4)

### 5.4 Ablations

- Memory size K
- Threshold and habituation settings
- Surprise signal: own prediction error vs Gaussian-surrogate surprise (Gorlo et al. style) vs fixed vs random
- Read mechanism: attention in predictor vs memory used only in the cost function

### 5.5 Stretch: second embodiment

A simulated quadrotor or point-mass navigation task with a transient cue `[TBD: simulator choice]`.

---

## 6. Compute and hardware

- **Simulation:** the LeWM environments `[TBD: confirm the simulator behind each environment from the LeWM repo]`.
- **GPU:** rented cloud GPU `[TBD: provider and monthly budget]`. LeWM reports training on a single GPU in a few hours, so the core experiments should fit a modest budget `[TBD: confirm once Week 1 reproduction is timed]`.
- **Real-robot demo (Feb to Mar 2027):** arm not yet chosen `[TBD]`. Requirements: a wrist or overhead camera, good repeatability (the world model must predict where the arm actually goes), and an open software stack. Candidates: SO-101 (lowest cost), Koch v1.1 (Dynamixel servos, more precise, higher cost).

---

## 7. Timeline (11 weeks to a complete preprint)

| Week | Dates | Goal | Done when |
|---|---|---|---|
| 1 | Oct 28 to Nov 3 | Read the 5 core papers; clone LeWM; reproduce one task end to end | Success rate and planning latency measured on our hardware |
| 2 | Nov 4 to 10 | Build T1 and T2; run memoryless LeWM | **Checkpoint 1:** memoryless fails on T1/T2 (H1). If not, redesign tasks |
| 3 | Nov 11 to 17 | Baselines: frame stack, full history, oracle | Baseline table filled |
| 4 | Nov 18 to 24 | Event memory and surprise gate, v1 | Runs end to end on T1 |
| 5 | Nov 25 to Dec 1 | Tune; add T3/T4 | **Checkpoint 2:** gated beats fixed and random at the same K (H3). If not, re-scope (see Risks) |
| 6 | Dec 2 to 8 | Full runs over seeds; latency benchmarking | Main results table |
| 7 | Dec 9 to 15 | Ablations | Ablation table |
| 8 | Dec 16 to 22 | Stretch embodiment *or* extra seeds; start writing method and setup | Sections drafted |
| 9 | Dec 23 to 29 | Writing (holiday buffer) | Intro, related work, results drafted |
| 10 | Dec 30 to Jan 5 | Figures; full draft | Complete draft |
| 11 | Jan 6 to 12 | Review, code release, arXiv submission | Preprint live |

After January: real-arm demo, then cold outreach with paper plus video.

---

## 8. Risks and fallbacks

| Risk | Fallback |
|---|---|
| Closest work (Gorlo et al., 2026) is very similar in spirit | Cite it prominently; our contribution is control plus speed plus action-conditioned surprise. Re-check arXiv every 2 weeks for new overlaps |
| World models plan reliably only over short horizons (Alrasheed et al., 2026) | Keep tasks short, or plan to subgoals |
| Surprise threshold is brittle | Adaptive threshold and habituation; report sensitivity honestly |
| Gated memory does not beat fixed or random writes (H3 fails) | Re-frame as a careful study of memory cost vs reliability in world-model planners; still a complete, publishable result |
| Timeline slips | Drop H4 and T3/T4 first; the core is T1, T2, control, H1 to H3 |

---

## 9. Open decisions

- [ ] Exact earliest application deadline (sets the hard stop)
- [ ] Base model: LeWM or Fast-LeWM
- [ ] Final task list
- [ ] Memory size K and eviction rule
- [ ] GPU provider and budget
- [ ] Robot arm for the demo
- [ ] Venue after arXiv (workshop or conference) `[TBD]`

---

## 10. Later directions (not in this paper)

- **Cross-episode memory:** distilled lessons stored in a compact graph and retrieved only when surprised.
- **Environmental memory:** the agent externalizes task state into the scene (see *Artifacts as Memory Beyond the Agent Boundary*, Martin et al., 2026).
- **Swarm / leader-follower control** with learned world models.

---

## 11. Repository layout (planned)

```
surprise-memory-wm/
├── README.md        this file
├── notes/           one reading note per paper
├── envs/            memory-requiring task variants
├── src/             memory module, surprise gate, planner glue
├── experiments/     configs and run scripts
├── results/         logs, tables, figures
└── paper/           LaTeX draft
```

---

## 12. References

1. Torne, Pertsch, et al. *MEM: Multi-Scale Embodied Memory for Vision Language Action Models.* 2026. https://arxiv.org/abs/2603.03596
2. Gorlo, Wise, Speranzon, Carlone. *Worth Remembering: Surprise-Gated Robot Episodic Memory.* 2026. https://arxiv.org/abs/2606.03787
3. Zheng et al. *Mem-World: Memory-Augmented Action-Conditioned World Models for Persistent Robot Manipulation.* 2026. https://arxiv.org/abs/2606.18960
4. Fan, Zhang, Han. *Event-Centric World Modeling with Memory-Augmented Retrieval for Embodied Decision-Making.* 2026. https://arxiv.org/abs/2604.07392
5. Wang et al. *EV-WM: Event-Verified World Models for Long-Horizon Robotic Manipulation.* 2026. https://arxiv.org/abs/2606.13053
6. Maes, Le Lidec, Scieur, LeCun, Balestriero. *LeWorldModel: Stable End-to-End Joint-Embedding Predictive Architecture from Pixels.* 2026. https://arxiv.org/abs/2603.19312
7. Gao, Xu. *Fast LeWorldModel.* 2026. https://arxiv.org/abs/2606.26217 (code: https://github.com/Yuntian-Gao/Fast-LeWorldModel)
8. Alrasheed, Azam, Akhtar. *The Planning Limits of Latent World Models.* 2026. https://arxiv.org/abs/2609.39235
9. Martin et al. *Artifacts as Memory Beyond the Agent Boundary.* 2026. https://arxiv.org/abs/2604.08756
10. Finn, C. Talk on the state of physical intelligence, Y Combinator, 2026 (transcript in project notes).
11. TypeSafe AI. *Jev introduction.* https://docs.typesafe.ai/introduction
