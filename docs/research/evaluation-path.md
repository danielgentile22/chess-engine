# Evaluation path: classical first, or NNUE from the start?

Researched 2026-09-16.

Scope: how to get from no evaluation function to an NNUE (Efficiently Updatable Neural Network, a small neural network evaluation whose first layer is updated incrementally as moves are made; introduced by Yu Nasu in 2018 for shogi and ported to chess in 2020 [12]) in a hobby engine, given a target of roughly 3000 Elo on the CCRL 40/15 rating list on one CPU core, a Rust engine with a Python and PyTorch or Rust trainer, an MIT licence, a Mac Studio (M4 Max), and a few hours a week. The question is issue #4. Search is covered separately in `docs/research/search-survey.md` and is assumed to be built in parallel. Every Elo number below is quoted with the conditions it was measured under, because self-play Elo and rating-list Elo are different scales and neither is additive.

## Answer

**Take route (a), but build a deliberately cheap hand-crafted evaluation, not a good one.**

The sequence is: material and piece-square tables, then a texel-tuning pass (fitting the evaluation's weights to game outcomes by gradient descent), then a handful of extra evaluation terms, then use that engine to generate self-play data, then replace the whole thing with a `768 -> 128x2 -> 1` network trained in `bullet`. Target roughly 2600 to 2800 CCRL for the hand-crafted stage and stop there. Do not chase the 3400+ that a mature hand-crafted evaluation can reach.

Four findings drive this, and two of them cut against the obvious reasoning.

1. **Both routes are proven, so strength is not the deciding axis.** Stormphrax and Reckless each bootstrapped a first network from randomly initialised weights, with no evaluation knowledge at all, and both beat their own hand-crafted predecessors by more than 300 self-play Elo [16][21]. Viridithas, akimbo, Berserk and Altair each went the other way and trained a first network on data their hand-crafted evaluation had labelled [13][22][26][29]. Nobody is blocked by this choice.

2. **3000 CCRL 40/15 on one core does not need a network at all.** On the list computed 2026-09-10, Stockfish 11, the last purely classical Stockfish, sits at 3495 on a single core, and Berserk 4.7.0, which deliberately pairs a 2026 search with a restored hand-crafted evaluation, sits at 3449 [31][32]. Ordinary hand-crafted engines cross 3000 routinely: Igel 2.3.0 at 3015, RubiChess 1.3 at 3006, Texel 1.04 at 3001 [31]. The stated target is reachable classically, so "NNUE is required to hit 3000" is false and should not be an argument for either route.

3. **The learning value sits in the texel-tuning step, and it is the same mathematics as NNUE training.** Texel tuning minimises `E = (1/N) * sum((result_i - sigmoid(qScore_i))^2)` over positions labelled with game results [35][36]. `bullet`'s reference example minimises `output.sigmoid().squared_error(target)` where `target = wdl * game_result + (1 - wdl) * sigmoid(score / 400)` [10]. Same loss, same sigmoid, same data shape (position, score, result). Only the model changes, from a linear function of hand-chosen features to a 768-input two-perspective network. Doing texel tuning first means the neural step is a model swap into machinery already understood, not a new discipline.

4. **The thing route (a) is usually sold on, that the classical evaluation stays useful afterwards, is false.** Stockfish kept its world-class classical evaluation frozen alongside NNUE for five releases and then deleted it, stating that "the idea that this code could lead to further inputs to the NN or search did not materialize" and that its remaining use was worth "roughly 2Elo" [3]. Plan to delete the hand-crafted evaluation. Budget it as tuition, not as infrastructure.

The cost of this recommendation is honest and worth stating: a few weeks of a limited hobby budget go into code that gets deleted. The mitigation is the word "cheap" in the first line. The failure mode of route (a) is not choosing it, it is over-investing in it.

### Route comparison

| Axis | (a) Classical first | (b) Bootstrap a network directly |
|---|---|---|
| Learning value | High. Chess domain knowledge, plus texel tuning as a gentle first machine-learning step on the same loss NNUE uses [35][10] | Low on domain, moderate on tooling. The first net teaches data plumbing, not chess |
| Time to a playable engine | Fast. Material and piece-square tables is an afternoon and the engine plays immediately | Slow. Stormphrax needed five successive networks before the release-quality one [16] |
| Time to 3000 | Reachable without any network [31], then roughly +100 to +155 more from a network [32][33] | Similar endpoint, but the early months produce nothing rateable |
| Data-generation quality | Good from the start. Labels are search scores from a real evaluation | Poor at first. A random net gives noise, so early targets must be game results only [30] |
| Milestone granularity | Every step is separately testable by self-play. Matches the project constraint | Long stretches where the engine is not stronger than the previous version |
| Precedent | Viridithas, akimbo, Berserk, Altair, Stockfish [13][22][26][29][1] | Stormphrax, Reckless, Hobbes [16][21][30] |
| Risk | Sunk cost in deleted code | Compute grinding with no learning payoff; hard to debug a bad net with no reference eval |

### What each engine's first network actually trained on

This is the core of the historical question. All from release notes, READMEs or in-repo network logs.

| Engine | Had a hand-crafted eval first? | First network trained on | Category |
|---|---|---|---|
| Stockfish (2020) | Yes, world class | "the evalutions of millions of positions at moderate search depth", from its own classical eval [1] | Own classical eval |
| Viridithas 3.0.0 | Yes, texel-tuned | Games from Viridithas 2.5.0 to 2.7.0, "all rescored with a development version of Viridithas 2.7.0 at low depth" [13] | Own classical eval |
| akimbo 0.6.0 | Yes | Own HCE dataset, where the HCE itself was bootstrapped "starting from material values" and iterated [22] | Own classical eval, seeded from material |
| Berserk 6 | Yes | "350 million FENs from Berserk 5 self play games" [26] | Own classical eval |
| Altair 6.0.0 | Yes, then zeroed | HCE weights "completely set to zero", then eight rounds of iterative datagen; first net on 30M positions [29] | Own eval, zero-knowledge variant |
| Stormphrax 1.0.0 | Yes (as Polaris) | Self-play from a chain of five nets, "the first being seeded with random weights and biases" [16] | Random bootstrap |
| Reckless 0.5.0 | Yes, through v0.4.0 [20] | "generated through self-play, initially using a randomly initialized network" [21] | Random bootstrap |
| Alexandria 2.0.0 | Yes | Training data given to the author, generated by Koivisto [27] | Someone else's data |
| Obsidian | No public one | "Positions: 50M (from little_guinea_pig)", then Stockfish binpacks [28] | Someone else's data |

Two observations. First, the majority had a hand-crafted evaluation first, and most used it as the labeller. Second, the two clean random bootstraps both came from authors who had already built and shipped a hand-crafted engine, so neither is evidence that a first-time engine author should skip that stage.

## The historical record: Stockfish 2020 to 2024

Worth getting right because it is usually told wrong.

The NNUE port landed in commit `84f3e86790` on 2020-08-05, closing PR #2912 [1]. The commit message claims "> 80 Elo on fishtest" [19], with two cited runs: +92.77 ±2.1 Elo over 60,000 games at 10s+0.1s on one thread, and +89.47 ±2.0 over 40,000 games at 20s+0.2s on eight threads [1]. The network "is optimized and trained on the evalutions of millions of positions at moderate search depth" [1], the typo being Stockfish's own. The labels came from the classical evaluation Stockfish already had. This is route (a) executed at the largest possible scale.

At the merge itself there was no hybrid: `Eval::evaluate` simply branched on a `Use NNUE` option [1]. The hybrid arrived the next day, commit `3dca13a958`, 2020-08-06, which used NNUE only on materially balanced positions because "NNUE eval is slower than classical eval for most of the hardwares" [2]. That condition was narrowed over three years to `!useNNUE || abs(psq) > 2048`, annotated in the source as worth "~4 Elo at STC, 1 Elo at LTC" [3].

The classical evaluation was deleted in commit `af110e02ec` (committed 2023-07-11, PR #4674, by Joost VandeVondele), removing 2,745 lines across 16 files including `pawns.cpp`, `material.cpp` and `endgame.cpp` [3]. The commit says the classical code was "roughly 25% of SF", that its remaining value was "roughly 2Elo", and, most tellingly for this decision, that "the idea that this code could lead to further inputs to the NN or search did not materialize" [3]. All three cited tests are small regressions accepted on maintenance grounds: -2.35 ±1.1, -1.74 ±1.0 and -1.70 ±0.9 Elo, each over 100,000 games [3]. Classical piece-square tables went separately in `0ad9b51dea` on 2023-07-24 [4].

Two corrections to the common account: the removal shipped in **Stockfish 16.1** (2024-02-24), not Stockfish 16, because the `sf_16` tag predates the commit [6]; and the Stockfish 12 release announcement contains **no Elo figure**, only "will typically win at least ten times more game pairs than it loses" [5].

The span from first NNUE to deleting the classical evaluation was about three years, and the verdict was that the classical evaluation had no second life.

## Can a hand-crafted evaluation reach 3000?

Yes, comfortably, and this materially changes the framing of the question.

CCRL 40/15 is "equivalent to 40 moves in 15 minutes on an Intel i7-4770k", ponder off, general book to 12 moves [31]. Entries without a `4CPU` or `8CPU` suffix are single-core; CCRL does not state this in those words, but its hash-size rule refers to "single-CPU engines" in contrast to 4-CPU entries, so this is a strong inference rather than a quotation [31]. Figures below are from the list computed 2026-09-10.

| Engine (1 core) | CCRL 40/15 | Notes |
|---|---|---|
| Stockfish 11 | 3495 ±11 | Last purely classical Stockfish, Jan 2020 [31] |
| Berserk 4.7.0 | 3449 ±18 | 2026 search with the v4.6.0 hand-crafted eval restored [32] |
| Ethereal 12.75 | 3377 ±9 | Last pre-NNUE Ethereal [31] |
| Weiss 2.0 | 3264 ±10 | Hand-crafted throughout [31] |
| Xiphos 0.6 | 3303 ±6 | Hand-crafted [31] |
| RubiChess 1.8 | 3240 ±14 | Last pre-NNUE [31] |
| Igel 2.6.0 | 3157 ±18 | Last pre-NNUE [31] |
| Polaris 1.7.0 | 2927 | Texel-tuned hobby engine, Stormphrax's predecessor [17] |
| akimbo 0.5.0 | 3026 | "Better HCE", author's own table [22] |

So the hand-crafted ceiling on one core is somewhere around 3450 to 3500, and 3000 is where an ordinary hand-crafted evaluation lands when paired with a competent search. Reaching 3000 is mostly a search and engineering problem, which is consistent with the search survey's finding that a handful of search techniques are worth hundreds of Elo each.

What a network buys, holding search constant, is smaller than hobby folklore suggests. The cleanest controlled comparison available is Berserk against itself: 4.7.0 with the hand-crafted evaluation at 3449 versus Berserk 14 with NNUE at 3604, a gap of 155 Elo attributable to evaluation [32][31]. Stockfish 11 to 12 is +49 on the same list [31]. Berserk 6's own release test against its hand-crafted predecessor was +137.82 ±5.35 over 10,000 games at 40s+0.4s, one thread [26].

The much larger figures quoted by hobby engines are self-play against a weak predecessor, where Elo inflates: akimbo 0.6.0 measured +387.57 ±19.94 over 2,000 games against akimbo 0.5.0 [23], Stormphrax 1.0.0 measured +357.65 ±16.33 against Polaris 1.8.1 [16], Reckless 0.5.0 measured +350.3 ±27.2 against 0.4.0 [21], and Viridithas 3.0.0 measured +249.1 ±10.6 against 2.7.0 [14]. akimbo's own rating-list movement over the same change was 3026 to 3335, i.e. +309 [22], so in that case the self-play figure overstated by about 80 Elo. Treat self-play deltas as directionally right and numerically generous.

## Texel tuning: the bridge

Peter Österlund described the method on TalkChess in January 2014; Miguel Ballicora had proposed an equivalent for Gaviota in 2009 [35][36]. Minimise

```
E = (1/N) * sum_i (result_i - sigmoid(qScore_i))^2
sigmoid(s) = 1 / (1 + 10^(-K*s/400))
```

where `result_i` is the game result from White's point of view (0, 0.5 or 1), `qScore_i` is the **quiescence search score** of the position rather than the static evaluation, and `K` is a single scaling constant fitted once and then frozen [35][36]. Österlund found K = 1.13 for Texel [35].

Data requirement, in his numbers: about 64,000 games at a fast time control giving roughly 8.8 million positions, excluding book positions and positions with mate scores [35]. That is roughly 100 to 150 positions per game.

Payoff, from the method's author: across 2014, evaluation weight tuning alone contributed "24.6 + 4.0 + 5.8 + 2.8 + 12.8 + 39.4 + 10.2 = 99.6 elo" to Texel [35]. So the first tuning pass on an untuned evaluation is worth a lot. Andrew Grant's paper, the best modern treatment, uses full AdaGrad gradient descent over the whole evaluation rather than coordinate descent, and reports much smaller gains when retuning an already-tuned Ethereal: +10.0 Elo for linear terms, +3.4 for king safety [37].

Why this matters for the recommendation: the texel step produces the data pipeline (self-play games, position extraction, result labelling), the loss function, and the habit of measuring evaluation changes by fitted error rather than intuition. All three carry directly into NNUE training. Viridithas, akimbo and Altair all describe exactly this progression in their own words [13][22][29].

## Tooling: can the Mac Studio train?

**Yes, and this was the largest open risk going in.** `bullet` (jw1912), the de facto NNUE trainer for hobby engines, is Rust, MIT-licensed, and has had a Metal backend since 2026-06-15 [8][9].

The backend is real, not a stub. `crates/gpu/src/runtime/metal.rs` implements the full GPU binding trait against `objc2-metal`, with runtime Metal Shading Language compilation and matrix multiplication through `MPSMatrixMultiplication`; `crates/gpu/Cargo.toml` gates the Apple dependencies under `cfg(target_os = "macos")` and links the `MetalPerformanceShaders` framework [9]. It arrived in PR #525 [9]. The documented requirement is one line: "For users on macOS. Enable the `metal` feature" [8].

Three caveats, all confirmed from source and all worth a spike before committing:

- **There is no CPU backend any more.** It was removed in the 2026 backend rewrite. With no backend feature enabled you get a mock device that errors out [9]. So Metal is the only option on a Mac, not a fallback.
- **Metal is not wired into the low-level `DefaultDevice` alias** in `crates/trainer/src/run.rs`, which has arms only for `cuda` and `rocm`. The high-level `ValueTrainerBuilder` path used by `examples/simple.rs` and `examples/progression/*` does have a Metal arm and is fine; some other examples would silently hit the mock [9].
- **Metal is not covered by continuous integration.** The workflow runs on Linux with `cuda,rocm` only, and there is no macOS runner [9]. The backend is three months old.

Fallback if Metal disappoints: `nnue-pytorch` (official-stockfish) is GPL-3.0 but is a trainer, not linked code, and it supports Apple's Metal Performance Shaders directly. Its `config.py` offers `accelerator: "auto" | "cuda" | "mps" | "cpu"` and the README has a dedicated "Setup for Apple Silicon" section [11]. Marlinflow, the older alternative, is effectively dead: two commits in two years, both from an outside contributor, and no licence file at the repository root [9]. Do not plan around it.

Data preparation in `bullet` is done by `bullet-utils` (convert, interleave, shuffle, validate), which the docs state does not require any GPU backend [8]. So the whole data pipeline is Mac-native regardless.

On data format, `bullet` recommends `viriformat` binpacks because they are "the most commonly used amongst people who generate their own data (and thus there are reference implementations in many programming languages)" [8]. The simpler option for a first net is bulletformat's `ChessBoard`, a 32-byte record holding occupancy, packed piece codes, score, result and both king squares, stored relative to the side to move [42]. `bullet-utils convert --from text` will build it from lines of `<FEN> | <score> | <result>` [8], which is the least effort path from a fresh datagen module. Note there is no offline Stockfish binpack converter; Stockfish binpacks are converted during training instead [8].

`bullet` contains no datagen guidance at all. The string "datagen" does not appear in its source or documentation [8][9]. Every engine writes its own, inside the engine, which is why step 5 below is engine work rather than trainer work.

## What a first network needs

The canonical first architecture is `768 -> N x2 -> 1`: 768 inputs (2 colours x 6 piece types x 64 squares), a hidden layer of N computed once per side ("two perspectives", hence x2), and a single output. `bullet`'s reference example uses **N = 128** [10].

Reference hyperparameters, taken verbatim from `examples/progression/1_simple.rs` [10]:

- Activation: SCReLU (squared clipped rectified linear unit)
- Loss: `output.sigmoid().squared_error(target)`
- `eval_scale: 400.0`, `wdl_proportion: 0.75`, so the target is 75 percent game result and 25 percent `sigmoid(score / 400)`
- Quantisation on save: input layer to `i16` at 255, output layer at 64, output bias at 255 * 64
- Optimiser AdamW, cosine learning-rate decay from 0.001
- `batch_size: 16_384`, `batches_per_superbatch: 6104`, so one superbatch is 100,007,936 positions, the conventional "100 million"
- 40 superbatches for the simple net

The quantisation constants are not arbitrary: akimbo's inference code hard-codes exactly `QA = 255`, `QB = 64`, `SCALE = 400` [25], so the engine-side integer inference matches the trainer's output format directly.

Sizing evidence, and two cautionary data points:

- akimbo's first net was `768 -> 256x2 -> 1` and crushed its hand-crafted evaluation by +387 self-play Elo [23]. Viridithas and Carp also shipped 256 first [43].
- Reckless's first committed network was `768 -> 32`, not even two-perspective, purely as a bootstrap vehicle; it went 32 to 64 (+59.7 ±27.2), to 128 (+112.4 ±28.5), then added SCReLU (+35.2 ±20.1) over about three months [43]. Motor's published ladder shows the same shape: 32 to 64 was +278.3 ±54.9, 256 to 512 was +142.2 ±8.4, 512 to 1024 was +62.9 ±8.6, and 1024 to 1536 was +67.2 ±11.9 [43].
- Altair's first net was `768 -> 64x2 -> 1` on 30 million positions and measured **-46.02 ±19.68 against its own master**, i.e. still weaker than the hand-crafted evaluation it was meant to replace [29]. It took three more generations (128M, 360M, then roughly 800M positions) to win.

So 32 is enough to bootstrap from nothing, but 64 is not enough to beat a decent hand-crafted evaluation, and 256 comfortably is. Start at 128 or 256.

**Do not widen ahead of the dataset.** Viridithas's network log records the identical change (widening to 512 neurons) losing 31.8 ±11.4 Elo on a small dataset and gaining 19.1 ±8.3 on a larger one, with the author's note that big nets "appear to work now ... almost certainly due to the much larger training set" [43]. Width is a function of data volume, not ambition.

**Shuffle the data.** Viridithas's first network, `viri0`, is logged as "much weaker than the HCE". `viri1` was "the same data as viri0, but data was shuffled, which fixed problems", and became the net that "crushes HCE" and shipped in v3.0.0 [43]. The same failure recurred twice later in the same log. If the first net comes out weak, suspect the data ordering before the architecture.

`bullet`'s own documentation warns against the opposite error, copying a Stockfish architecture. From `docs/1-basics.md`, "Beginner Traps": "Just start with basic 768 inputs. You won't have enough data for things like HalfKA/HalfKP at first", and "Many aspects of the SF architectures require **significant** effort, amounts of data, and/or training time/complexity to actually gain elo. As a result, an engine may (and likely will for a beginner) actually *lose* elo with an SF architecture vs a much simpler one" [8]. For scale, Stockfish's current network has roughly 86,896 input features feeding 1024 neurons per perspective across 8 buckets [10].

One practical gift: `bullet`'s `examples/simple.rs` ships a complete quantised inference implementation in Rust, including the accumulator and its `add_feature` and `remove_feature` methods [10]. It is a working reference for the engine-side code in step 8, which is worth reading before writing that code by hand.

King buckets (separate weight sets by king position) and output buckets (separate output layers by material count) are later refinements, not first-net features. `bullet`'s own examples are ordered this way: `1_simple` is plain `768 -> 128x2 -> 1`, `2_output_buckets` adds output buckets, `3_input_buckets` adds input buckets [10]. akimbo's history matches: output buckets at 0.7.0 (+96 Elo), a wider net at 0.8.0 (+73), king buckets and mirroring at 1.0.0 (+35) [24].

Isolated in their own test patches, buckets are worth single digits to low double digits, against roughly 350 for the first net: Viridithas measured +5.47 ±3.57 for four king buckets with mirroring, +23.64 ±8.91 for going from 4 to 9 king buckets, and +19.76 ±6.80 for eight output buckets; Reckless measured +3.03 ±2.46 at short time control for four output buckets [43]. Viridithas also has a failed first attempt on record, a four-bucket net at -14.37 ±8.14 against its unbucketed predecessor [43]. Buckets are not on the critical path, and the calendar gaps confirm it: Viridithas went NNUE in October 2022 and added king buckets in March 2024 [43].

## Data generation: what it costs

Real settings from engines with public datagen modules. The pattern is remarkably consistent: play from a randomised opening, search every position to a fixed node budget, record position, score and eventual game result.

| Engine | Random opening plies | Search budget per position | Filtering |
|---|---|---|---|
| Viridithas | 8 or 9 | 25,000 soft nodes, 200,000 hard | Verification search, reject if abs(eval) > 1000 [15] |
| Stormphrax | 8 or 9 | 24,000 soft nodes, 1,000,000 hard | Verification at depth 10, score limit 500; win/draw adjudication [18] |
| akimbo 0.6.0 | 8 or 9 | 5,000 soft nodes, 1,000,000 hard | Quiet-position filter; drop last 8 positions of drawn games |
| Altair 5.0.0 | 8 | 5,000 nodes, 100 ms cap | Opening discarded beyond 400 centipawns [29] |
| Alexandria 3.5.0 | 6 | 2,500 nodes | Depth-10 sanity search, discard abs(score) > 1000 |

Two details worth copying. A 5,000 soft-node budget is the historically validated starting value: Stormphrax used it from July 2023 until May 2025, and Reckless used it in its first datagen [43]. And Viridithas's run filenames record about 80 written positions per game after filtering (1,000,000 games yielding 80 million positions) [43], which is the number to plan dataset size against.

Dataset sizes actually used for a first network: Altair 30 million [29], Obsidian 50 million [28], Carp "100M d8 self-play fens", Berserk 350 million [26]. Mature networks are far larger: akimbo used 1 billion at 0.7.0 and 1.5 billion at 0.8.0 [24], Carp 3.2 billion, Renegade over 8.4 billion [43]. The most useful calibration point is Viridithas's own log: its first counted runs were 60 and 80 million positions, and the 80 million set alone measured *worse* than the older mixed set at -87.6 ±19.1 [43]. A few hundred million is where a dataset starts being clearly productive.

**Cost estimate, derived rather than sourced.** No engine publishes positions-per-second for datagen, and `bullet` publishes no training throughput figures on any hardware [8][9]. Mark this whole paragraph as an estimate. At 5,000 nodes per position and a plausible 2 million nodes per second for an early Rust engine on one M4 Max core, one core yields on the order of 400 positions per second. Using 12 of 16 cores, that is roughly 5,000 positions per second, so 50 million positions in about three hours of wall clock and 100 million in about six. That is comfortably within a hobby budget, and it is the reason to prefer a 5,000-node budget over Viridithas's 25,000 for a first dataset. This estimate should be replaced with a measurement as soon as the engine can run a fixed-node search.

## Licensing

The project is MIT and public, so this matters. Findings, all from the licence files and project statements themselves.

| Asset | Licence | Usable in an MIT repo? |
|---|---|---|
| Stockfish engine source | GPL-3.0 [3] | No, do not copy code |
| Stockfish `nn-*.nnue` networks | **CC0-1.0** [7] | Yes. The `networks` repo carries a CC0 LICENSE and states "All networks in this repository were uploaded by the authors under the CC0 license"; fishtest's upload form requires agreeing to CC0 [7] |
| Leela Chess Zero training data | ODbL-1.0 on the database, DbCL-1.0 on contents | With attribution. `storage.lczero.org/files/training_data/LICENSE.txt` states this explicitly [38] |
| Leela Chess Zero networks | **No licence stated anywhere** [38] | Treat as all rights reserved |
| Linrock NNUE datasets (robotmoon, Hugging Face) | **No licence stated anywhere** [39] | Avoid without clearing |
| `official-stockfish/fishtest_pgns` | LGPL-3.0 | Avoid |
| Lichess eval database | **CC0-1.0** [34] | Yes. "Use them for research, commercial purpose, publication, anything you like" [34] |
| `bullet` trainer | MIT [8] | Yes |
| `nnue-pytorch` trainer | GPL-3.0 [11] | Yes as a tool; it is not linked into the engine |

Note the asymmetry Stockfish itself operates: it trains on ODbL Leela data, discharges the obligation with an attribution line in its README [40], and releases the resulting networks under CC0 [7]. Whether a trained network is a "Produced Work" under ODbL (attribution only, section 4.3) or a "Derivative Database" (share-alike, section 4.4) [41] is **genuinely unsettled**, no chess project states an answer, and the GNU GPL FAQ does not mention neural networks or training data at all. Stockfish's conduct implies Produced Work. That is an inference from behaviour and not legal advice.

Practical consequence: reusing a Stockfish network is legally clean but defeats the purpose of the project. Several reference engines treat data originality as a point of pride, and say so: Stormphrax "prides itself on using self-generated training data ... and will never use data generated by a third party" [17], Viridithas "prides itself on using original training data" [13]. Notably, **none of Viridithas, Stormphrax, monty, bullet, Ethereal, Leorik or akimbo has released its training data.** Self-generated data is the norm, not the exception.

If a permissively licensed external dataset is ever wanted, the Lichess eval dump is the defensible option: CC0, 409,710,113 positions, 22.1 GB compressed, though the evaluations come from heterogeneous Stockfish versions running in browsers at varying depth, so quality is uneven [34].

## Recommended sequence for this project

Each step ends with a playable engine and is gated by self-play against the previous version. "Hand-written" means Daniel writes it; "agent" means tooling an agent builds.

1. **Material plus piece-square tables** (hand-written, one session). Tapered between middlegame and endgame. The engine plays real chess and is rateable. Do not use borrowed values; the point is to tune them yourself in step 3.
2. **Self-play harness and SPRT gating** (agent). `fastchess` or `cutechess-cli` driving a match against the previous binary, with a sequential probability ratio test as the stopping rule. This is needed before any evaluation work can be measured, and it is reused for the rest of the project's life.
3. **Texel tuning** (hand-written maths, agent-built data pipeline). Extract positions from self-play games with results, fit `K`, then gradient-descend the evaluation weights against `E = mean((result - sigmoid(qScore))^2)` [35]. Budget roughly 8 million positions from tens of thousands of fast games [35]. Expect a large gain: this is the step Österlund measured at about 100 Elo over a year [35]. It is also the step that teaches the loss function used in step 7.
4. **A modest set of evaluation terms** (hand-written). Mobility, pawn structure, king safety, rook on open file, bishop pair. Retune after each. **Stop when the engine is around 2600 to 2800.** Resist going further; Stockfish's own verdict is that this code has no second life [3].
5. **Fixed-node datagen mode** (agent, with the search hand-written already). A `datagen` subcommand: 8 or 9 random opening plies, a verification search that rejects openings beyond about 1000 centipawns, 5,000 soft nodes per position, win and draw adjudication, output in bulletformat's 32-byte `ChessBoard` records or `viriformat`. Copy the settings shape from the table above, not the code. Measure actual positions per second here and replace the estimate in this note.
6. **Generate 50 to 100 million positions** (agent-run, overnight). Altair's 30 million was not enough for a small net [29]; Berserk used 350 million [26]. 100 million is one `bullet` superbatch and a sensible first target.
7. **Train `768 -> 128x2 -> 1` in `bullet` with the Metal backend** (agent tooling, hand-written understanding). Start from `examples/progression/1_simple.rs` verbatim, unchanged; 128 is its default `hl_size` [10]. Train a 256 net as the second run and let self-play pick between them (see Open questions). **Spike this first**, before step 5, because the Metal backend is three months old and has no continuous integration [9]. If Metal fails, fall back to `nnue-pytorch` on Metal Performance Shaders [11].
8. **Hand-write the inference code** (hand-written, and this is the real learning). The accumulator, incremental update on make and unmake, SCReLU, and integer dequantisation with `QA = 255`, `QB = 64`, `SCALE = 400` [10][25]. This is the part that makes NNUE "efficiently updatable" and it belongs to the core, alongside move generation and search. akimbo's entire `network.rs` is 5.5 KB, so this is a small file, not a subsystem.
9. **Gate the swap by self-play.** Expect a large win if the net is sized right, and expect it to be smaller than the +250 to +390 that other engines report against weaker predecessors [23][16][21][14].
10. **Iterate.** Regenerate data with the NNUE engine, retrain, repeat. Only then consider output buckets, then king buckets, then a wider layer, in that order [10][24].

Steps 1 to 4 are perhaps a third of the calendar time and nearly all of the chess-domain learning. Steps 5 to 8 are where the engine gets strong.

## Open questions

These are candidates for tickets.

- **Does `bullet --features metal` actually build and train on an M4 Max?** Read from source only, never executed [9]. The `DefaultDevice` gap and the absence of macOS continuous integration make this the single highest-value spike in this note. Resolve before step 5.
- **What is this engine's real datagen throughput?** The cost figures above are derived from assumed nodes per second, not measured. No engine and no `bullet` document publishes positions per second [8][9].
- **How many positions does a first net actually need at hidden size 128 versus 256?** The evidence is a scatter: 30M failed at size 64 [29], 50M sufficed for Obsidian at L1 64 [28], 350M for Berserk [26]. No source isolates the size-versus-data tradeoff.
- **Is the `768 -> 128` or `768 -> 256` choice worth an arena?** Both are cheap to train once the pipeline exists. Two runs would settle it empirically for this engine rather than by analogy.
- **How did Stormphrax's random-weight bootstrap produce usable labels?** Ciekce never states it. The strong hint is Hobbes's network log, whose first row records `wdl: constant(1)`, meaning the training target was 100 percent game result and 0 percent evaluation score [30]. That is a cross-engine inference, not a Stormphrax source. It matters only if route (b) is ever revisited.
- **Where does the static evaluation get cached during search once it is an NNUE call?** Flagged in the search survey as a Tier 2 decision; NNUE makes evaluation calls much more expensive, which changes the answer.
- **Is a network a "Produced Work" or a "Derivative Database" under ODbL?** Unsettled, unaddressed by every project examined, and not addressed by the GNU GPL FAQ. Only relevant if external data is ever used; self-generated data avoids the question entirely.
- **Does texel tuning stay useful after NNUE?** The search survey notes hundreds of search constants tuned by SPSA. Whether the texel machinery built in step 3 can be repointed at search parameters is untested here.

## Sources

1. Stockfish commit `84f3e867903f62480c33243dd0ecbffd342796fc` (2020-08-05), "Add NNUE evaluation", PR #2912. Claims "> 80 Elo on fishtest"; +92.77 ±2.1 over 60,000 games at 10+0.1, one thread. https://github.com/official-stockfish/Stockfish/commit/84f3e867903f62480c33243dd0ecbffd342796fc
2. Stockfish commit `3dca13a958cd0dfea1cdea91da230c5aac9e322f` (2020-08-06), "NNUE evaluation threshold", PR #2916. Introduces the material-balance hybrid condition. https://github.com/official-stockfish/Stockfish/commit/3dca13a958cd0dfea1cdea91da230c5aac9e322f
3. Stockfish commit `af110e02ec96cdb46cf84c68252a1da15a902395` (committed 2023-07-11), "Remove classical evaluation", PR #4674. "roughly 25% of SF"; "roughly 2Elo"; three regression tests at 100,000 games each. https://github.com/official-stockfish/Stockfish/commit/af110e02ec96cdb46cf84c68252a1da15a902395
4. Stockfish commit `0ad9b51deaaa1f2a8273ed064fbf6425cfbbe4f2` (2023-07-24), "Remove classical psqt", PR #4713. https://github.com/official-stockfish/Stockfish/commit/0ad9b51deaaa1f2a8273ed064fbf6425cfbbe4f2
5. Stockfish 12 release announcement (2020-09-02). No Elo figure; "at least ten times more game pairs than it loses". https://stockfishchess.org/blog/2020/stockfish-12/
6. Stockfish 16.1 release notes (2024-02-24). "Removal of handcrafted evaluation (HCE)". https://github.com/official-stockfish/Stockfish/releases/tag/sf_16.1
7. `official-stockfish/networks` README and LICENSE (CC0-1.0), plus the fishtest upload template requiring CC0. https://github.com/official-stockfish/networks and https://github.com/official-stockfish/fishtest/blob/master/server/fishtest/templates/nn_upload.html.j2
8. `bullet` documentation, `docs/2-getting-started.md` (CUDA, ROCm and Metal backends), `docs/3-data.md` (formats), `docs/1-basics.md` ("Just start with basic 768 inputs"). https://github.com/jw1912/bullet/blob/main/docs/0-contents.md
9. `bullet` source at commit `2ea3d2d0f7` (main, 2026-09-15): `crates/gpu/Cargo.toml`, `crates/gpu/src/runtime/metal.rs`, `crates/bullet_lib/Cargo.toml` feature flags, `crates/trainer/src/run.rs`, `.github/workflows/checks.yaml`. Metal backend added in PR #525, commit `3b7341725b` (2026-06-15); CPU backend removal discussed in issue #498. https://github.com/jw1912/bullet
10. `bullet` `examples/progression/1_simple.rs` and the rest of `examples/progression/`. Architecture `768 -> 128x2 -> 1`, SCReLU, `eval_scale` 400, WDL 0.75, QA 255, QB 64, 16,384 x 6,104 positions per superbatch. https://github.com/jw1912/bullet/blob/main/examples/progression/1_simple.rs
11. `official-stockfish/nnue-pytorch`, GPL-3.0. `config.py` accelerator `"auto" | "cuda" | "mps" | "cpu"`; README "Setup for Apple Silicon". https://github.com/official-stockfish/nnue-pytorch
12. Chess Programming Wiki, "NNUE" (Yu Nasu, 2018, for shogi; English translation by Dominik Klein, 2021-01-07). https://www.chessprogramming.org/NNUE
13. Viridithas README, "Evaluation Development History/Originality (HCE/NNUE)". First net "trained on a dataset of games played by Viridithas 2.7.0, 2.6.0, and 2.5.0, all rescored with a development version of Viridithas 2.7.0 at low depth"; marlinflow until 11.0.0, bullet since. https://github.com/cosmobobak/viridithas
14. Viridithas v3.0.0 release notes (2022-10-18). +249.1 ±10.6 Elo at 8s+0.08s, +207.9 ±30.1 at 3m+2s, against v2.7.0. https://github.com/cosmobobak/viridithas/releases/tag/v3.0.0
15. Viridithas `src/datagen.rs` at commit `e605537c8a` (master, 2026-09-06). `RANDOM_MOVES_ROOT = 8`, 25,000 soft nodes, hard limit 8x. https://github.com/cosmobobak/viridithas/blob/e605537c8a/src/datagen.rs
16. Stormphrax v1.0.0 release notes (2023-07-25). "five previous networks, the first being seeded with random weights and biases, so Stormphrax is a 'zero' engine"; +357.65 ±16.33 STC and +357.07 ±26.69 LTC against Polaris 1.8.1. https://github.com/Ciekce/Stormphrax/releases/tag/v1.0.0
17. Stormphrax README (rating table, "will never use data generated by a third party") and Polaris README (CCRL table: 1.7.0 at 2927 on 40/15, 1.8.x at 3048 Blitz). https://github.com/Ciekce/Stormphrax and https://github.com/Ciekce/Polaris
18. Stormphrax `src/datagen/datagen.cpp` at commit `d1468d99e3` (2026-08-29). 24,000 soft nodes, 1,000,000 hard, verification depth 10, adjudication constants. https://github.com/Ciekce/Stormphrax/blob/d1468d99e3/src/datagen/datagen.cpp
19. Chess Programming Wiki, "Stockfish NNUE" ("at least 80 Elo" on Fishtest, August 2020). https://www.chessprogramming.org/Stockfish_NNUE
20. Reckless release history v0.1.0 to v0.9.0-dev. HCE through v0.4.0, gradient-descent weight tuning at v0.2.0 and v0.3.0. https://github.com/codedeliveryservice/Reckless/releases
21. Reckless v0.5.0 release notes (2024-02-04). "generated through self-play, initially using a randomly initialized network"; +350.3 ±27.2 STC, +312.5 ±33.0 LTC against v0.4.0. https://github.com/codedeliveryservice/Reckless/releases/tag/v0.5.0
22. akimbo README. "starting from material values when akimbo still had an HCE and iteratively generating data and tuning"; rating table (0.4.1 PST-only 2841 Blitz, 0.5.0 HCE 3026 on 40/15, 0.6.0 NNUE 3335). Note the table's 1.0.0 entry of 2474 on 40/15 is inconsistent with its Blitz figure of 3583 and appears to be a typo. https://github.com/jw1912/akimbo
23. akimbo v0.6.0 release notes (2023-09-24). First net `768 -> 256x2 -> 1`, codenamed `bob`, trained with bullet on self-generated HCE data; +387.57 ±19.94 over 2,000 games at 60+0.6, one thread; 898 source lines. https://github.com/jw1912/akimbo/releases/tag/v0.6.0
24. akimbo v0.7.0, v0.8.0 and v1.0.0 release notes. `768->512x2->8` on 1 billion positions (+96.38); `(768->768)x2->1` on 1.5 billion (+73.01); `(768x4hm -> 1024)x2 -> 1` on 1.5 billion (+34.86). https://github.com/jw1912/akimbo/releases
25. akimbo `src/network.rs`. `HIDDEN = 1024`, `SCALE = 400`, `QA = 255`, `QB = 64`, 4 king buckets. https://github.com/jw1912/akimbo/blob/main/src/network.rs
26. Berserk 6 release notes (2021-10-19). "trained using a custom trainer from 350 million FENs from Berserk 5 self play games"; +137.82 ±5.35 over 10,000 games at 40+0.4, one thread, against Berserk 4.5.1. https://github.com/jhonnold/berserk/releases/tag/6
27. Alexandria v2.0.0 README (2022-06-18). "gave me some training data generated by Koivisto". https://github.com/PGG106/Alexandria
28. Obsidian `nnuehistory/net1.txt`: "Positions: 50M (from little_guinea_pig) / L1: 64 / Epochs: 45". Later nets use Stockfish binpacks. The identity of "little_guinea_pig" could not be established. https://github.com/gab8192/Obsidian
29. Altair v6.0.0 release notes and `changelog.md` (2023-12-05). "Altair's evaluation weights were completely set to zero, allowing for the generation of completely original data from zero knowledge"; "First net with 64 HL size, trained on 30M fens of HCE data" measured -46.02 ±19.68 against master. https://github.com/Alex2262/AltairChessEngine
30. Hobbes `network_history.txt` (kelseyde). First row: `hobbes-random | (768->32)x2->1 | source: hobbes-random | dataset: 33 million | wdl: constant(1)`. The `wdl: constant(1)` is the mechanism by which a random-weight bootstrap gets usable labels. https://github.com/kelseyde/hobbes-chess-engine/blob/main/network_history.txt
31. CCRL 40/15 rating list, computed 2026-09-10 (2,443,716 games). Conditions: "Equivalent to 40 moves in 15 minutes on an Intel i7-4770k", ponder off, book to 12 moves. Single-core entries: Stockfish 11 3495, Ethereal 12.75 3377, Weiss 2.0 3264, Xiphos 0.6 3303, RubiChess 1.8 3240, Igel 2.6.0 3157, Igel 2.3.0 3015, RubiChess 1.3 3006, Texel 1.04 3001, Berserk 14 3604. https://www.computerchess.org.uk/ccrl/4040/
32. Berserk 4.7.0 release notes (2026-05-24). "pairs the modern Berserk search with the classic v4.6.0 hand-crafted evaluation restored in place of the NNUE network"; 3449 ±18 on CCRL 40/15, one core. https://github.com/jhonnold/berserk/releases/tag/4.7.0
33. Ethereal v13.00 release notes (2021). Claims "up to +120 elo over Ethereal 12.75 in self-play". Note Ethereal 13.00 shipped in free and commercial builds and which one CCRL tested could not be confirmed, so the +40 CCRL delta should not be read as the NNUE gain. https://github.com/AndyGrant/Ethereal/releases/tag/v13.00
34. Lichess open database. "Database exports are released under the Creative Commons CC0 license. Use them for research, commercial purpose, publication, anything you like." `lichess_db_eval.jsonl.zst`, 409,710,113 positions, 22,086,532,809 bytes, updated 2026-09-10. Broadcast games are CC-BY-SA-4.0, not CC0. https://database.lichess.org/
35. Chess Programming Wiki, "Texel's Tuning Method", summarising Peter Österlund's TalkChess posts of 2014-01-31. Error function, sigmoid, K = 1.13, roughly 64,000 games and 8.8 million positions, and the per-iteration Elo list summing to 99.6. Prior art credited to Miguel Ballicora, Gaviota, 2009. https://www.chessprogramming.org/Texel%27s_Tuning_Method
36. Peter Österlund, TalkChess thread "How Do You Automatically Tune Your Evaluation Tables", 2014-01-31. https://talkchess.com/viewtopic.php?t=50823&start=20
37. Andrew Grant, "Evaluation & Tuning in Chess Engines" (AdaGrad gradient descent over the whole evaluation), in the Ethereal repository; announced on TalkChess 2020-08-24. Reports +10.0 Elo for linear terms and +3.4 for king safety when retuning an already-tuned engine. https://github.com/AndyGrant/Ethereal/blob/master/Tuning.pdf
38. Leela Chess Zero training data licence: `https://storage.lczero.org/files/training_data/LICENSE.txt`, ODbL-1.0 on the database and DbCL-1.0 on contents. Leela networks carry no licence statement at any of `storage.lczero.org/files/networks/`, `lczero.org/play/networks/bestnets/` or `training.lczero.org`.
39. Linrock NNUE datasets index and Hugging Face mirrors. No licence stated on the index page or on any of the nine datasets (no `cardData`, no README). https://robotmoon.com/nnue-training-data/
40. Stockfish README "Acknowledgements": "Stockfish uses neural networks trained on data provided by the Leela Chess Zero project, which is made available under the Open Database License (ODbL)." https://github.com/official-stockfish/Stockfish/blob/master/README.md
41. Open Database License 1.0, definitions of "Produced Work" (section 4.3, attribution only) and "Derivative Database" (section 4.4, share-alike). https://opendatacommons.org/licenses/odbl/1-0/
42. `bulletformat`, the 32-byte `ChessBoard` record (occupancy, packed pieces, score, result, king squares), side-to-move relative. MIT. https://github.com/jw1912/bulletformat
43. Engine network logs and hidden-size history read from source and git history: Viridithas `networkhistory.txt` (present at tags, e.g. v11.0.0; `viri0` "much weaker than the HCE", `viri1` "data was shuffled, which fixed problems", viri34 -31.8 ±11.4 versus viri44 +19.1 ±8.3 for the same widening, viri38 -87.6 ±19.1 on 80M positions) and PRs #108, #136, #167 for bucket Elo; Reckless `src/nnue.rs` commits `c9de498` (768 -> 32, "randomly initialized parameters"), `05659ad` (+59.7 ±27.2), `a34f6aa` (+112.4 ±28.5), `bee8f74` (SCReLU, +35.2 ±20.1), PR #56 (+3.03 ±2.46); Motor release notes for the 32 to 1536 ladder; Carp `src/nnue/nnue_history.txt`, `chess/src/nnue/mod.rs` and `src/nnue/nnue_notes.txt` (a walkthrough of `768 -> 256x2 -> 1` inference, good reading before hand-writing the Rust); Stormphrax `src/datagen/datagen.cpp` at `e22013ca` (5,000 soft nodes, 2023) versus `a029757` (24,000, 2025); Renegade README (8.4 billion positions). https://github.com/cosmobobak/viridithas , https://github.com/codedeliveryservice/Reckless , https://github.com/dede1751/carp , https://github.com/martinnovaak/motor
