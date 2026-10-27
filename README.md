# Keepsake

**What should a robot remember?**

Keepsake tests whether a world model's own surprise is a good guide.

Paper working title: *What Should a Robot Remember? Surprise as a Guide to Lean Memory in Real-Time World-Model Planning*

| | |
|---|---|
| Status | Step 01 in progress. Project started 27 Oct 2026. Revised 27 Oct 2026 (surprise treated as a hypothesis, not a definition; two claims kept separate). |
| Author | Vinod Anbalagan |
| Target | Complete preprint on arXiv by end of December 2026; real-robot demo in early 2027 |
| Progress | Tracked step by step on the [Lab page](https://vinodanbalagan.vercel.app/lab) |
| Convention | Anything marked `[TBD: ...]` is unconfirmed. It is a placeholder, not a value. |

---

## 1. The idea in one paragraph

Most state-of-the-art robot foundation models have no memory. They act on the current camera frame only, which works for short or repetitive tasks but fails on long tasks where the scene does not show how far along you are (Finn, YC talk, 2026). Memory is expensive when done naively: 10 seconds of video from four cameras at 50 Hz is about half a million tokens, and even 1 frame per second is about 10,000 tokens (same talk). Physical Intelligence's answer, MEM, uses compressed video for short-term memory and LLM-written text summaries for long-term memory (Torne et al., 2026).

**Our question:** is a world model's own surprise an effective way to decide what goes into a small, fixed memory, so that a fast latent world-model planner stays reliable on tasks where the current frame does not reveal what to do next?

**Our candidate mechanism:** the planner runs memoryless by default (fast, "Type 1"). It writes to a small, fixed-size memory when its own action-conditioned prediction of the next state is wrong, and reads that memory when planning (slower, "Type 2"). No video buffer, no LLM, constant memory size, constant cost.

**What we do not assume:** that surprise equals importance. A visually unexpected but irrelevant change can produce a large prediction error. A familiar-looking moment that marks real task progress can produce almost none. Whether surprise is a good proxy for what is worth remembering is something this project tests, not something it takes for granted.

---

## 2. Research question and claims

**RQ.** Is surprise from an action-conditioned world model an effective way to allocate a limited memory budget for reliable, low-latency planning?

The project makes two separate claims, which can come out differently and are reported separately throughout. Success on one must never be presented as success on the other.

| ID | Claim | Tested by | What would falsify it |
|---|---|---|---|
| H1 | A memoryless latent world-model planner fails on our memory-requiring tasks. | No-memory baseline vs oracle | It succeeds anyway, meaning the tasks do not need memory (redesign the tasks) |
| H2 | **Efficiency.** A compact, fixed-size memory recovers most of the reliability of full history at a fraction of the planning cost, whatever rule decides what is written. | Compact memory with fixed or random writes vs full history vs recurrent state | Compact memory is clearly worse than full history at every budget, or no faster |
| H3 | **Selection.** At the same memory budget, writing on the world model's own surprise beats writing at fixed intervals or at random. | Surprise vs fixed vs random writes, same K | Fixed or random writes do as well |
| H4 | **Event types.** Surprise catches unexpected outside changes, misses self-caused progress, and fires on irrelevant novelty. | The Surprise Audit (Step 02), per event type | Surprise behaves the same across all three event types |

**Event types (the core of the analysis):**

| Event type | Example | Expected surprise behaviour |
|---|---|---|
| Unexpected outside change | A box moved by something else; a cue whose identity cannot be predicted | Fires (good) |
| Self-caused progress | "I just finished box 3" | Probably silent, because the robot caused it (miss) |
| Irrelevant novelty | A flicker or a moving distractor | Fires (false alarm, the "noisy TV" problem from the curiosity literature) |

If H4 holds as expected, the natural method extension is a second write trigger for self-caused task progress, alongside surprise. That extension is only added if Step 02 shows the miss.

**The first decisive question (Step 02):** after training, the model has learned when cues appear and disappear. Does surprise still fire for the cue? Our expectation: it fires for the cue's *identity* (which goal it shows), which the model cannot predict, not for its *appearance*, which it can. If the gate misses the cue entirely, that is the precise limitation the paper must address, not a reason to abandon the direction.

---

## 3. Why this is a gap: prior work and positioning

| Work | Memory | Memory used for | Speed reported | How we differ |
|---|---|---|---|---|
| **MEM** (Torne et al., 2026, Physical Intelligence) | Compressed video (short-term) plus LLM-written text summary (long-term) | VLA control, tasks up to 15 min | Not the focus | No video, no LLM; a world-model planner, not a VLA |
| **Worth Remembering** (Gorlo, Wise, Speranzon, Carlone, 2026, MIT and Lockheed Martin) | Surprise-gated episodes (8 frames each), surprise from a Gaussian fit over V-JEPA-2 features | Question answering by an LLM agent | **No** latency reported | Surprise from the planner's *own action-conditioned* prediction error (they list an action-conditioned predictor as future work); memory used for **control**; speed is a primary metric; surprise is tested as a proxy, not assumed |
| **Mem-World** (Zheng et al., 2026) | 4D surfel memory of wrist views | Video world model for policy evaluation and data generation, not planning | No latency in main text; trained ~2 days on 8 H100s | Lean latent memory, used inside a planner |
| **ERA** (Fan, Zhang, Han, 2026) | Case bank of (event code, maneuver, reliability) | Retrieval controller for UAV navigation; ~130 Hz on Jetson Orin Nano | Yes | Events are hand-defined (distance threshold); ours are triggered by learned surprise |
| **EV-WM** (Wang et al., 2026) | No dedicated memory module | Verifying imagined plans via structured event states | No | Complementary: verification, not memory |
| **LeWorldModel** (Maes, Le Lidec, Scieur, LeCun, Balestriero, 2026) | None | Latent planning from pixels; ~15M params; trains on one GPU in a few hours; planning up to 48x faster than DINO-WM | Yes | **Our base model.** Its benchmark tasks do not require memory |
| **Fast-LeWM** (Gao, Xu, 2026) | None | Parallel latent rollout; ~2.1x faster CEM solve than LeWM | Yes | Possible faster base; code is open |

**Positioning sentence (draft):** *Prior work gates memory on surprise for offline question answering, or adds heavy memory to video world models for evaluation. We test whether an action-conditioned world model's own surprise is a good way to fill a constant-size memory inside a real-time planner, separate the value of compact memory from the value of surprise-based selection, and measure the trade-off between reliability and planning speed directly.*

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
4. **Write rule (the variable under test):** surprise (the world model's own one-step prediction error, with an adaptive threshold), fixed interval, or random, all at the same budget. For the surprise threshold we start from the robust median-plus-MAD style threshold used by Gorlo et al. and add habituation, because they report that recurring surprising subjects keep triggering writes.
5. **Progress trigger (conditional):** a second write trigger for self-caused task progress, added only if Step 02 shows surprise misses it.

**Type 1 / Type 2 reading:** most steps never touch memory beyond a cheap attention read over a few slots. Memory is written only at selected moments, so it stays small and does not bloat.

---

## 5. Experiments

### 5.1 Tasks (memory-requiring variants of LeWM's environments)

LeWM evaluates on Push-T, OGBench-Cube, Two-Room and Reacher, none of which need memory. We modify them so that the current frame does not reveal what to do next. Each task is chosen to produce one event type from Section 2.

| ID | Task | Why the current frame is not enough | Event type it tests |
|---|---|---|---|
| T1 | **Transient cue** (Two-Room): a goal cue is visible only for the first steps, then disappears | The agent must remember which goal was shown | Unexpected outside change (cue identity) |
| T2 | **Identical items** (OGBench-Cube style): push each of N identical cubes to a zone exactly once | Processed and unprocessed cubes look the same | Self-caused progress |
| T3 | **Consequential perturbation**: an object the task depends on is displaced mid-task | Memory must be *updated* | Unexpected outside change |
| T4 | **Distractor**: an irrelevant object moves or flickers during T1 or T3 | Tests false alarms | Irrelevant novelty |
| C | **Original LeWM tasks** (control) | Checks that memory does not hurt tasks that do not need it | None |

First implementation: T1 and T3 only, plus control. T2 and T4 are added in Step 02 for the audit, and kept if the audit says they matter. `[TBD: final task list after Step 02]`

### 5.2 Baselines

- No memory (plain LeWM)
- Frame stacking (last k latents)
- Recurrent state in the predictor (the learned-memory competitor, as in Dreamer-style models)
- Full history (all past latents, attention)
- Compact memory with fixed-interval writes, same K
- Compact memory with random writes, same K
- Oracle memory (ground-truth task state), as an upper bound

### 5.3 Metrics

- Task success rate (mean and spread over `[TBD: seeds, at least 3]` seeds)
- Planning latency per decision (ms) and rate (Hz), on stated hardware
- Memory footprint (slots, bytes) and writes per episode
- **Surprise audit:** for each event type, how often surprise fires on labelled important moments (recall) and how often it fires on unimportant ones (false-alarm rate)
- Recovery after perturbation (T3)

### 5.4 Ablations

- Memory size K
- Threshold and habituation settings
- Surprise signal: own action-conditioned prediction error vs Gaussian-surrogate surprise (Gorlo et al. style)
- Read mechanism: attention in predictor vs memory used only in the cost function

---

## 6. Compute and hardware

- **Simulation:** the LeWM environments `[TBD: confirm the simulator behind each environment from the LeWM repo]`.
- **Code and checkpoints:** LeWM's paper links public code `[TBD: confirm whether pretrained checkpoints and datasets are released, in Step 01]`.
- **GPU:** rented cloud GPU, budget not a constraint. LeWM reports training on a single GPU in a few hours `[TBD: confirm once Step 01 reproduction is timed]`.
- **Real-robot demo (early 2027):** arm not yet chosen `[TBD]`. Requirements: a wrist or overhead camera, good repeatability (the world model must predict where the arm actually goes), and an open software stack. Candidates: SO-101 (lowest cost), Koch v1.1 (Dynamixel servos, more precise, higher cost).

---

## 7. The Lab: six steps to the paper (target: preprint by end of December 2026)

Each step is one entry on the website's Lab page. Status values match the site: `building` (in progress), `next` (up next), `planned`, `shipped` (done and written up). The first two steps are deliberately about checking assumptions, not building the memory system: if either assumption fails, we want to know in the first two weeks.

| Step | Title | Dates | Goal | Done when | Status |
|---|---|---|---|---|---|
| 01 | The Baseline | Oct 28 to Nov 3 | Literature comparison finalized; reproduce one LeWM task end to end | Success rate and planning time per decision measured on our hardware | building |
| 02 | The Surprise Audit | Nov 4 to 10 | No memory yet. Log the trained model's prediction error on T1 and T3 (plus T2, T4 if time) against hand-labelled important moments | **Checkpoint 1 (go/no-go):** recall and false-alarm rate per event type. Does surprise still fire for the cue after training? | next |
| 03 | Tasks That Need Memory | Nov 11 to 17 | Finalize T1 and T3; run memoryless LeWM and the oracle | **Checkpoint 2:** memoryless fails, oracle succeeds (H1). If not, redesign tasks | planned |
| 04 | Remembering Everything | Nov 18 to 24 | Baselines: frame stack, recurrent state, full history | Baseline table filled | planned |
| 05 | The Surprise Gate | Nov 25 to Dec 8 | Compact memory with surprise, fixed and random writes at the same K; progress trigger only if Step 02 calls for it | **Checkpoint 3:** H2 (efficiency) and H3 (selection) answered separately | planned |
| 06 | The Price of Memory | Dec 9 to 15 | Full runs over seeds; success vs planning time for every variant; ablations | Main results table, trade-off plot, ablation table | planned |
| Paper | Keepsake | Dec 16 to 29 | Write, make figures, internal review, release code, submit to arXiv | Preprint live | planned |

After the paper: the same planner and memory on a real arm with a camera (early 2027).

---

## 8. Risks and fallbacks

| Risk | Fallback |
|---|---|
| Closest work (Gorlo et al., 2026) is very similar in spirit | Cite it prominently; our contribution is control plus speed plus action-conditioned surprise tested as a proxy. Re-check arXiv every 2 weeks for new overlaps |
| Surprise misses the trained cue entirely (Step 02) | That is the limitation the paper addresses: analyse why, then test a progress or identity-based trigger |
| Surprise fires on irrelevant novelty (noisy TV) | Measure the false-alarm rate (T4); test habituation and thresholds; report honestly |
| World models plan reliably only over short horizons (Alrasheed et al., 2026) | Keep tasks short, or plan to subgoals |
| Surprise gating does not beat fixed or random writes (H3 fails) | H2 can still hold on its own. Together with the event-type analysis, this is potentially a useful negative result, *if* the analysis explains why. Publication is not guaranteed by completeness |
| Timeline slips | Keep T1, T3, control and H1 to H3; drop T2/T4 extensions and ablations first |

---

## 9. Open decisions

- [ ] Base model: LeWM or Fast-LeWM
- [ ] Final task list (after Step 02)
- [ ] Memory size K and eviction rule
- [ ] Whether a progress trigger is needed (after Step 02)
- [ ] GPU provider
- [ ] Robot arm for the demo
- [ ] Venue after arXiv (workshop or conference) `[TBD]`

---

## 10. Later directions (not in this paper)

- **Another body:** the same memory on a simulated drone or quadrotor, unchanged.
- **Typed fast heads:** small heads that answer atomic questions in one pass ("is this subtask done?", "how confident am I?"), inspired by "System One" models such as TypeSafe's Jev.
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
