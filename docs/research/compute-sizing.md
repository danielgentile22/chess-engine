# Compute sizing: what the Mac Studio covers, and what is worth renting

Researched 2026-09-16.

Scope: how much self-play testing and how much neural-network training this project needs, whether a Mac Studio (M4 Max, 16 CPU cores, 40 GPU cores, 64 GB unified memory) can supply it, and what the cloud alternatives cost. The question is issue #7, and the note also resolves half of the `bullet` Metal spike in issue #16 by reading the trainer's source. Which engines to read is covered in `docs/research/reference-engines.md`, the network architecture and dataset-size evidence in `docs/research/evaluation-path.md`, and search techniques in `docs/research/search-survey.md`; none of that is repeated here. Game counts were computed by running OpenBench's own `OpenBench/stats.py` (Andrew Grant's implementation of the pentanomial sequential probability ratio test) against a pentanomial distribution taken from that file's own worked example [1]. Throughput figures were derived from arithmetic, from time controls, or from network architecture read out of `bullet`'s source; **nothing in this note was measured on an M4 Max, because the machine has not arrived** [16]. Every price carries the date, the region and the exact instance type.

## Answer

**Buy no cloud compute. The Mac Studio covers both workloads with room to spare, and the one thing worth renting, an NVIDIA box for `bullet`, costs under a dollar a run and is insurance against a three-month-old Metal backend rather than a capacity purchase.**

The reasoning runs in three parts.

**Self-play testing is cheap in wall clock because a 10-second-per-side game takes ten seconds per side.** Throughput at a fixed time control is set by the clock, not by how fast the processor is, so twelve performance cores produce roughly 1,500 games per hour at 10+0.1 regardless of what silicon is underneath. A sequential probability ratio test (a stopping rule that accepts or rejects a patch as soon as the accumulated results decide it, rather than at a fixed game count) with `[0.0, 5.0]` Elo bounds resolves in about 13,000 games when the patch is worth nothing and about 4,500 when it is worth 10 Elo. That is eight hours and three hours respectively. For a person working a few hours a week, one patch test is one overnight run, and during the first year, when patches are worth tens of Elo rather than three, most tests finish over lunch. Renting cores buys parallelism you have no patches to fill.

**Data generation is a one-night job, not a compute problem.** A first `768 -> 128x2 -> 1` network wants somewhere between 50 and 300 million positions, and at akimbo's 5,000-soft-node setting twelve cores plausibly produce 2,400 to 7,200 positions per second, putting 100 million positions between four and twelve hours [8][9]. The estimate is soft in both directions and has to be measured, but no plausible value of it turns an overnight run into a cloud purchase.

**Training is the only place where the machine might genuinely fail, and the failure would be a software one.** `bullet` has a real Metal backend, added in 2026-06 and present at commit `2ea3d2d0f7` [3]. It also has no CPU backend at all, no macOS continuous integration, and one specific unfinished wiring detail that will bite the wrong example program [3]. The training job itself is tiny: the architecture from `examples/progression/1_simple.rs` processes 4.00 billion samples over 40 superbatches, and a forward and backward pass over a sparse 768-input network with a 128-wide hidden layer is roughly 26,000 floating-point operations per position, so the whole run is on the order of 100 teraflops of arithmetic [2]. That is seconds of work for any modern accelerator and minutes of work once data loading is counted. If Metal works the run is short. If Metal does not work, the correct response is to rent an hour of a small NVIDIA instance, not to buy anything.

Against that, the marginal cost of the Mac Studio is electricity, and Apple's published maximum for the nearest configuration it documents is 145 W [12]. At the U.S. average residential rate of 18.34 cents per kilowatt-hour for June 2026, a machine pinned at that ceiling costs 2.7 cents an hour, or about 64 cents for a 24-hour test [12][14]. The cheapest cloud core-hour found in this survey, an AWS Graviton4 spot core, is 1.24 cents, so sixteen of them are 19.8 cents an hour, roughly 7.5 times the Mac Studio's electricity for the same core count and with worse cores [10].

The one thing that is *not* free is the reader's attention, and the note's real recommendation is about measurement rather than money: five numbers in this document are estimates, and the first week with the machine should replace all five. They are listed in "What must be measured" below, with the exact command for each.

## (a) Self-play regression testing

### How many games per hour sixteen cores produce

Start from an arithmetic ceiling that needs no measurement. In a time control written `base+increment`, each side's total clock over a game of `M` moves per side cannot exceed `base + increment * M` seconds. A game therefore cannot last longer than twice that, plus whatever the match runner spends between moves. Engines with competent time management consume nearly all of their clock, so this ceiling is close to tight, and it is a *floor* on games per hour that is independent of processor speed.

| Time control | Moves per side | Seconds per game (ceiling) | Games/hour per slot | 12 slots | 16 slots |
|---|---|---|---|---|---|
| 10+0.1 | 40 | 28.0 | 129 | 1,543 | 2,057 |
| 10+0.1 | 60 | 32.0 | 112 | 1,350 | 1,800 |
| 10+0.1 | 80 | 36.0 | 100 | 1,200 | 1,600 |
| 60+0.6 | 40 | 168.0 | 21 | 257 | 343 |
| 60+0.6 | 60 | 192.0 | 19 | 225 | 300 |
| 60+0.6 | 80 | 216.0 | 17 | 200 | 267 |

Take **1,500 games per hour at 10+0.1 and 250 at 60+0.6 on twelve slots** as the planning numbers. Game length is the one input here that is not fixed, and 40 moves per side is the assumption; the adjudication rules in `fastchess`'s own example (`-resign movecount=3 score=600 -draw movenumber=34 movecount=8 score=20`) exist precisely to truncate long games, so the shorter rows of the table are the realistic ones [4].

`fastchess` overhead does not change this materially. Engines are not restarted between games by default (`restart=(off|on)`, default `off`), so process startup is paid once per match rather than once per game, and the per-game cost is a `ucinewgame` plus a position setup [4]. The project's own published stress result is the relevant reassurance: at 0.2+0.002 seconds with up to 250 threads, "only 10 matches out of 20,000" timed out [5]. At 10+0.1 the margin is fifty times larger.

One caveat that matters if this project ever copies OpenBench's design. OpenBench does not run a test at the time control as written. `Client/worker.py` benchmarks both binaries, computes a scale factor from the nodes-per-second ratio, and multiplies the base and increment by it, appending `timemargin=250` [1]. A machine whose cores are half as fast gets twice the clock and therefore half the throughput. All the numbers above assume no such scaling, which is the right assumption for a single-machine setup where every test runs on the same silicon.

### The efficiency cores, and why you cannot simply pin around them

The M4 Max's 16-core CPU is 12 performance cores plus 4 efficiency cores [13]. Running a time-controlled game on an efficiency core does not merely make it slower, it makes the result wrong, because both engines are measured against a wall clock that one of them is now searching far fewer nodes inside.

**The usable concurrency is 12, and the tooling will not work this out for you.** On Apple Silicon there is no simultaneous multithreading, so every "core count" the operating system exposes is the full 16. Verified on an Apple Silicon Mac on 2026-09-16: `hw.ncpu`, `hw.physicalcpu` and `hw.logicalcpu` all report the total including efficiency cores, and only `hw.perflevel0.physicalcpu` and `hw.perflevel1.physicalcpu` distinguish them [16]. Two consequences follow directly from source:

- `fastchess` calls `std::thread::hardware_concurrency()` to cap concurrency and to fill in a negative or zero `-concurrency` argument, so it sees 16 [4]. `-quick` sets concurrency to `hardware_concurrency() - 2`, which would be 14 and would still land on two efficiency cores [4].
- OpenBench's worker, with `--threads auto`, sets `self.threads = psutil.cpu_count(logical=False)`, which on Apple Silicon is also 16 [1].

**Worse: `fastchess`'s `-use-affinity` flag does nothing on macOS.** In `app/src/affinity/affinity.hpp`, the `__APPLE__` branch of `setThreadAffinity` is a one-line `return false` commented "Not implemented", and `setProcessAffinity` is a block of commented-out `thread_policy_set` code ending "do nothing for now" [4]. The macOS CPU-topology helper is candid about it too: `cpuinfo_mac.hpp` opens with "Some dumb code for macOS, setting the affinity is not really supported" and simply enumerates every logical processor as its own physical core [4].

This is not a `fastchess` bug. macOS deliberately exposes no interface for pinning a thread to a specific core, and the one scheduling control it does offer, `taskpolicy(8)`'s `-c background` quality-of-service clamp, pushes work *onto* the efficiency cores, which is the opposite of what a time-controlled match needs [16]. So the mitigation is not pinning but restraint: **pass `-concurrency 12` explicitly and never let the tool choose.** macOS's scheduler prefers performance cores for threads at default quality of service, so twelve busy engine pairs should occupy the twelve performance cores and leave the efficiency cores for the operating system and the match runner itself.

Whether that actually holds is a measurement, not a fact, and it is the first thing to check on the new machine. The check is in "What must be measured".

### How many games one test needs

OpenBench decides this with Wald's sequential probability ratio test. `OpenBench/workloads/create_workload.py` sets the two stopping thresholds as `lowerllr = log(beta / (1 - alpha))` and `upperllr = log((1 - beta) / alpha)`, which at the conventional `alpha = beta = 0.05` are −2.9444 and +2.9444 [1]. `OpenBench/stats.py` then accumulates a log-likelihood ratio, in the pentanomial form that scores *game pairs* on the five outcomes `(LL, LD, DD, DW, WW)` rather than individual games [1]. Pairing is why `fastchess` defaults to `-games 2` per round and warns that "setting this higher than 2 does not provide meaningful results": the two games of a pair share an opening, which removes most of the opening's contribution to the variance [4].

**The repository ships no default bounds.** The `test_bounds` field in `Templates/OpenBench/create_workload.html` is a bare input with no value, and validation only requires `[float1, float2]` with the first below the second [1]. So "the bounds small patches use" is a convention among engine authors, not something OpenBench encodes, and the honest answer is a table across the range rather than a single figure.

The load-bearing subtlety is that game counts do not depend only on the bounds. `PentanomialSPRT` converts its Elo arguments into normalised Elo, a scale on which one unit means the same amount of statistical evidence regardless of how drawish the games are [1]. Reading the conversion out of the code, normalised Elo equals logistic Elo when the standard deviation of a game pair's score is `1/(2*sqrt(2))`, about 0.3536, and scales as `0.3536 / sd` otherwise. A drawish, low-variance distribution therefore needs *fewer* pairs for the same logistic-Elo bounds. **The pair standard deviation is the single number that decides how long your tests take, and it is a property of your engine, your opening book and your time control, so nobody else's figure substitutes for it.**

The anchor used below is the worked example inside `stats.py` itself: `R5 = (39, 8843, 26675, 9240, 44)` over 44,841 pairs, which has a mean pair score of 0.502269, a variance of 0.025662 and therefore a pair standard deviation of 0.16019 [1]. That is a drawish distribution (59.5% of pairs split 1-1), consistent with a long time control. Shorter controls and sharper books are more decisive, so a plausible range for 10+0.1 is 0.22 to 0.26 and for 60+0.6 is 0.16 to 0.18. **Those two ranges are the estimate on which every wall-clock figure below rests.**

Games to resolve, `alpha = beta = 0.05`, computed by running `PentanomialSPRT` on distributions tilted to each hypothesis with the file's own `MLE_tvalue` routine [1]:

| Bounds (logistic Elo) | Pair sd | Patch worth 0 Elo | Worth the upper bound | Worth 2x the bound | Worth 4x the bound |
|---|---|---|---|---|---|
| `[0.0, 5.0]` | 0.18 | 7,336 | 7,630 | 2,509 | 1,082 |
| `[0.0, 5.0]` | 0.24 | 13,021 | 13,625 | 4,461 | 1,913 |
| `[0.0, 3.0]` | 0.18 | 20,314 | 21,388 | 6,976 | 2,982 |
| `[0.0, 3.0]` | 0.24 | 36,021 | 38,230 | 12,428 | 5,295 |

Two properties of this table decide the workflow. Game counts fall roughly with the square of the bound width, so widening `[0.0, 3.0]` to `[0.0, 5.0]` cuts the test by about two-thirds. And game counts fall steeply with the true strength of the patch: a patch worth four times the upper bound resolves in about a seventh of the games of one worth nothing. What the table does not show is the pathological case, a patch whose true value sits exactly at the midpoint of the bounds, where the expected sample size runs into the millions and the test never resolves. That case is why every framework offers a maximum game count, and why the practical rule is to cap a test and treat the cap as a failure.

### Therefore: one patch test on the Mac Studio

Combining the two tables, at twelve concurrency slots:

| Time control | Bounds | Patch worth 0 (reject) | Worth the bound (accept) | Worth 2x | Worth 4x |
|---|---|---|---|---|---|
| 10+0.1, sd 0.24 | `[0.0, 5.0]` | 8.4 h | 8.8 h | 2.9 h | 1.2 h |
| 10+0.1, sd 0.24 | `[0.0, 3.0]` | 23.3 h | 24.8 h | 8.1 h | 3.4 h |
| 60+0.6, sd 0.18 | `[0.0, 5.0]` | 28.5 h | 29.7 h | 9.8 h | 4.2 h |
| 60+0.6, sd 0.18 | `[0.0, 3.0]` | 79.0 h | 83.2 h | 27.1 h | 11.6 h |

**This is fast enough, and it is fast enough for a reason that will not survive success.** For the first year the engine is climbing from nothing to roughly 3000, and the patches on that climb are the ones the search survey lists at tier 1 and tier 2: null-move pruning, late move reductions, a transposition table. Those are worth tens of Elo each, which is the right-hand column of the table, so they resolve in one to three hours at 10+0.1 [`search-survey.md`]. Run them at short time control with `[0.0, 5.0]` bounds and a 20,000-game cap, and a week's worth of patches fits in a week's worth of nights.

The regime this machine does *not* cover is the endgame: once the engine is near its ceiling and patches are worth 2 to 3 Elo, confirming one at 60+0.6 with `[0.0, 3.0]` bounds is a three-and-a-half-day run. That is the point where engine authors join a distributed OpenBench instance rather than buy cloud capacity, because the problem at that stage is not cost but the number of tests per week, and other people's idle cores are free. It is out of scope here and years away.

### Pricing cloud cores for games

The normalisation has to be stated carefully, because the obvious one is wrong. **At a fixed time control, self-play throughput is a function of the clock and not of the processor**, so a core-hour buys the same number of games on any machine and the right unit is dollars per core-hour. The assumptions in the numbers below: 10+0.1, 40 moves per side, 28 seconds per game, one game per physical core, 128.6 games per core-hour, so one million games is 7,776 core-hours. For 60+0.6 the same arithmetic gives 46,667 core-hours per million games. Two caveats. This prices raw throughput and ignores that a slower core plays a weaker game, which matters if the test is meant to predict rating-list strength; OpenBench's nodes-per-second rescaling is the standard fix and it would make slow cores proportionally more expensive [1]. And simultaneous multithreading is excluded: a hyperthread is not a game slot, because two engines sharing a physical core distort each other's node counts.

All AWS and Google prices read 2026-09-16, Linux, on demand, from the vendors' own pricing data feeds; the rendered pricing pages are JavaScript applications that serve no price text, so the JSON endpoints those pages query were used instead [10][11]. The AWS on-demand feed is stamped `2026-09-10T19:55:14Z`. Spot prices were read at 2026-09-16T17:39Z and move continuously.

| Instance | Region | Physical cores | Threads/core | On demand $/hr | Spot $/hr | $/core-hr (spot) | $/1M games at 10+0.1 (spot) |
|---|---|---|---|---|---|---|---|
| AWS `c8g.48xlarge` (Graviton4) | us-east-1 | 192 | 1 | 7.6570 | 2.3746 | 0.01237 | **96** |
| AWS `c7a.48xlarge` (AMD Genoa) | us-east-1 | 192 | 1 | 9.8534 | 3.1045 | 0.01617 | 126 |
| AWS `c7i.48xlarge` (Intel Sapphire Rapids) | us-east-1 | 96 | 2 | 8.5680 | 2.9545 | 0.03078 | 239 |
| GCP `c4-standard-288` | us-central1 | 144 | 2 | 14.2322 | 8.5187 | 0.05916 | 460 |

On demand the same column reads 310, 399, 694 and 769 dollars per million games. At 60+0.6 multiply everything by six: 577 dollars per million games on Graviton4 spot, 1,861 on demand.

**The ordering here is not the obvious one, and the reason is multithreading.** AWS's own CPU-options table shows `c7a` and `c8g` running one thread per core while `c7i` and Google's C4 run two [10]. Per *vCPU* the Intel instance looks competitive; per *physical core*, which is the only unit a chess engine can use, it costs two and a half times what Graviton4 does. Google's C4 is the most expensive per physical core of anything surveyed, at 9.9 cents on demand, and it is the only option priced as separate vCPU and memory components rather than a machine-type rate [11].

Hetzner is the interesting non-hyperscaler, because its dedicated servers are physical machines billed at a monthly cap with hourly granularity, so the per-core-hour figure sits between spot and on demand without spot's eviction risk. Read 2026-09-16 from Hetzner's own price feed, joined to the specs on its product pages; prices are net of value-added tax and exclude the primary IPv4 address at USD 1.90 a month [15]. Per-core-hour below divides the monthly USD cap by 730 hours.

| Hetzner plan | CPU | Physical cores | USD/month | One-off setup | $/core-hr | $/1M games at 10+0.1 |
|---|---|---|---|---|---|---|
| AX41 | AMD Ryzen 5 3600 (Zen 2) | 6 | 67.10 | 0 | **0.01532** | 119 |
| AX42 | AMD Ryzen 7 PRO 8700GE (Zen 4) | 8 | 117.10 | 59 | 0.02005 | 156 |
| AX162 | AMD EPYC 9454P (Genoa) | 48 | 722.10 | 359 | 0.02061 | 160 |
| AX102 | AMD Ryzen 9 7950X3D (Zen 4, 3D V-Cache) | 16 | 302.10 | 149 | 0.02586 | 201 |
| CCX63 (cloud) | AMD, dedicated vCPU | 48 vCPU | 1006.99 | 0 | 0.02874 | 224 |

Dedicated-server traffic is unlimited and free of charge except on the 10-gigabit uplink add-on [15]. Two things about this table are worth a sentence. AX41, at 1.53 cents per core-hour with no setup fee, is the cheapest *uninterruptible* core-hour in the whole survey and only 24% above Graviton4 spot, though Zen 2 cores are the slowest here and that matters for data generation but not for fixed-time-control games. And AX102's Ryzen 9 7950X3D carries 3D V-Cache, a very large L3, which is the one architectural feature that genuinely helps a chess engine's transposition table; nothing primary was found that quantifies the effect for an engine, so it is noted rather than credited. The plan named in the question, AX52, no longer exists: Hetzner's product endpoint lists AX41, AX42, AX102 and AX162 and nothing else in the AX line, and the AX52 page redirects to the server finder [15].

Set against all of this, the Mac Studio's twelve usable slots at 1,543 games per hour need 648 hours for a million games at 10+0.1, and 648 hours at Apple's documented 145 W ceiling and the U.S. average rate is **$17.22 of electricity** [12][14]. At California's 34.74 cents per kilowatt-hour it is $32.63 [14]. That is a fifth of the cheapest spot alternative before anyone counts the hours spent configuring it.

## (b) NNUE training

### What `bullet` will actually run on

Read from `bullet` at commit `2ea3d2d0f7` (`main`, 2026-09-15), cloned 2026-09-16 [3]. The answer to issue #16's first half, which is "does a Metal backend exist", is **yes, and it is a real implementation rather than a stub.** The answer to its second half, "does it train on an M4 Max", is not in the source and can only be settled by running it.

Three backends exist and they are mutually exclusive features on `bullet_lib`: `cuda`, `rocm` and `metal` [3]. `crates/gpu/src/runtime/metal.rs` is 469 lines against CUDA's 545 and ROCm's 504, which is the right order of magnitude for a peer implementation rather than a sketch [3]. It binds `objc2-metal` for buffers, command queues, libraries and compute pipelines, compiles kernels at runtime, and routes matrix multiplication through `MPSMatrixMultiplication` from Metal Performance Shaders [3]. The kernel compiler carries a real second dialect: `crates/gpu/src/runtime/dialect.rs` defines `Dialect::Msl` alongside `Dialect::CudaHip` and emits different text for pointer casts, vector construction, atomic adds and `pow` [3]. There is not one `unimplemented!`, `todo!` or `panic!` in the Metal file [3].

**There is no CPU backend.** With no backend feature enabled, `ExecutionContext` resolves to `bullet_gpu::runtime::mock::MockGpu`, whose error string reads, verbatim, "This is a mock runtime! It can't actually do anything! You need to enable either the `cuda` or `rocm` features!" [3]. Note that the message does not mention Metal, which is a small sign of how recent the backend is. So on a Mac, Metal is the only path, not a fallback.

Three specific risks, all read out of the repository:

**Continuous integration never touches it.** `.github/workflows/checks.yaml` has three jobs, all `runs-on: ubuntu-latest`. Clippy runs `--features=cuda,rocm`, the test job runs `cargo test --workspace` with no features at all, and the format job compiles nothing [3]. Since `metal.rs` is gated behind both `#[cfg(feature = "metal")]` and a `compile_error!` on non-macOS targets, **no CI job has ever type-checked the Metal backend**, let alone run it [3].

**One wiring gap survives, and it is the same one flagged in `evaluation-path.md` three months ago.** `crates/trainer/src/run.rs` defines `DefaultDevice` with arms for `cuda` and `rocm` and a catch-all of `Device<runtime::mock::MockGpu>` for everything else [3]. There is no `metal` arm. The high-level `ValueTrainerBuilder` path, which is what `examples/progression/1_simple.rs` uses, goes through `ExecutionContext` in `crates/bullet_lib/src/nn.rs`, which *does* have a Metal arm, so the recipe this project plans to use is fine [2][3]. Any example that reaches for `DefaultDevice` instead will silently build against the mock and fail at runtime. Stay on the `ValueTrainerBuilder` path.

**The documentation for Metal is one line.** `docs/2-getting-started.md` gives CUDA four bullet points and ROCm five, covering toolkit installation and required environment variables. The Metal section reads, in full: "For users on macOS. Enable the `metal` feature" [3]. That is either a genuinely zero-configuration backend or an under-documented one, and only running it distinguishes the two.

`bullet-utils` (convert, interleave, shuffle, validate) builds without any backend and is explicitly documented as not requiring CUDA or HIP, so data preparation is Mac-native whatever happens to Metal [3].

### How big the job is

The architecture is `examples/progression/1_simple.rs` verbatim, as the evaluation-path note settled: 768 inputs, a 128-wide hidden layer computed from both perspectives, one output, SCReLU activation, `eval_scale` 400, WDL proportion 0.75, and the quantisation the note requires (first layer at 255, second at 64, bias at 255 × 64) [2][`evaluation-path.md`].

| Quantity | Value | Where it comes from |
|---|---|---|
| Parameters | 98,689 | `768*128 + 128 + 256 + 1`, from the architecture [2] |
| Quantised network on disk | 197,378 bytes | Two bytes per parameter at `i16` [2] |
| Batch size | 16,384 | `TrainingSteps` in `1_simple.rs` [2] |
| Batches per superbatch | 6,104 | same, giving 100,007,936 positions per superbatch |
| Superbatches | 40 | same |
| **Total samples processed** | **4,000,317,440** | 40 × 6,104 × 16,384 |
| Optimiser | AdamW, cosine-decayed learning rate 0.001 to 0.0000024 | `lr::CosineDecayLR` in `1_simple.rs` [2] |
| Data loader threads | 2 | `LocalSettings` in `1_simple.rs` [2] |

The arithmetic is negligible and can be bounded from the architecture without measuring anything. The first layer is sparse: a chess position sets about 32 of the 768 inputs, so the forward pass is 32 columns of 128 accumulated per perspective, about 8,200 operations, and the output layer is 256 multiply-accumulates. At the usual rule of three times forward for forward-plus-backward, that is roughly 26,000 operations per sample and **about 105 teraflops for the entire 40-superbatch run**. A mid-range GPU delivers that in under a minute of pure arithmetic. **Training this network is a data-movement problem, not a compute problem**, and the binding constraints are the two loader threads and the disk.

Disk volume follows from the record formats, both read from source. `bulletformat`'s `ChessBoard` is 32 bytes, enforced by a `const _RIGHT_SIZE: () = assert!(std::mem::size_of::<ChessBoard>() == 32)` [6]. `viriformat` stores a whole game as one 32-byte `marlinformat::PackedBoard` for the starting position plus four bytes per move (a two-byte move and a two-byte little-endian evaluation) and a four-byte null terminator [7], so a 160-ply game yielding 80 usable positions is 676 bytes, about 8.5 bytes per position, roughly a quarter of bulletformat's size.

| Dataset | bulletformat on disk | viriformat on disk | Bytes read over 40 superbatches |
|---|---|---|---|
| 50M positions | 1.6 GB | ~0.43 GB | 64 GB |
| 100M positions | 3.2 GB | ~0.85 GB | 128 GB |
| 300M positions | 9.6 GB | ~2.6 GB | 384 GB |

At 100 million positions in bulletformat the dataset fits in the Mac Studio's 64 GB of unified memory with room to spare, so after the first superbatch the file system cache serves it and disk bandwidth stops mattering. That is a genuine advantage of 64 GB over a rented instance with 16 GB attached to a cheap GPU.

**Wall-clock training time is an estimate and cannot be anything else today.** `bullet` publishes no throughput figures on any hardware, in its README, its docs or its source, and no third-party figure was accepted for this note [3]. From the FLOP bound and the in-memory dataset, tens of minutes is the expectation rather than hours, and the risk is not that it is slow but that the Metal path errors out. That risk is what a rented NVIDIA hour insures against.

### How many positions, and how long to generate them

The dataset-size evidence is in `evaluation-path.md` and is not repeated: Altair's 30 million was too few at hidden size 64, Obsidian used 50 million, Berserk 350 million, and Viridithas's own log records an 80-million set measuring worse than an older mixed one [`evaluation-path.md`]. **Plan on 100 million for the first network and expect to want 300 million for the second.**

The generation settings come from the two datagen modules named in the reference-engines note, both read at their tags on 2026-09-16:

| | akimbo `v1.0.0` `src/datagen.rs` | Viridithas `v20.0.0` `src/datagen.rs` |
|---|---|---|
| Random opening plies | `8 + (rng % 2)` | `RANDOM_MOVES_ROOT = 8` plus a coin flip |
| Soft node limit per move | `SOFT_NODE_LIMIT = 5_000` | `nodes: 25_000` (default) |
| Hard node limit | `HARD_NODE_LIMIT = 1_000_000` | `nodes * 8`, so 200,000 |
| Hard timeout | `HARD_TIMEOUT_MS = 10_000` | none in the options struct |
| Adjudication | `ADJ_WIN_SCORE = 3000` | verification search, reject openings past 1000 |
| Positions written | quiet only: best move not a capture, side to move not in check | filtered similarly |
| Output | `bulletformat::ChessBoard` | marlinformat or viriformat |
| Lines | 294 | 1,567 |

Source: [8] and [9].

**Both tools print positions per second, which is the single most useful fact in this section.** akimbo's writer emits `#[{id}] written games {games} fens {fens} fens/sec {rate}` on every flush, once per sixteen games per thread [8]. Viridithas prints `|> FENs generated: {fens} (FENs/sec = {fps})` along with an estimated completion time [9]. So the throughput number that every cost estimate here depends on can be obtained in ten minutes on the new machine by building someone else's engine, without writing a line of this project's datagen.

Until then it is an estimate, and here is how it is built. At a 5,000-node soft limit, a position is written only when the best move is quiet, which in a typical game is roughly half the plies, so **about 10,000 nodes of search per written position**. Dividing an assumed per-core node rate by that gives positions per core-second.

| Nodes/sec per core (assumed) | Positions/core-sec | 12 cores | 100M positions | 300M positions |
|---|---|---|---|---|
| 2 million | 200 | 2,400/s | 11.6 h | 34.7 h |
| 4 million | 400 | 4,800/s | 5.8 h | 17.4 h |
| 6 million | 600 | 7,200/s | 3.9 h | 11.6 h |

**Every figure in that table is an estimate.** The middle row matches the derived figure already in `evaluation-path.md`, which is reassuring only in the sense that it is the same assumption twice [`evaluation-path.md`]. The range it spans, four hours to a day and a half for 100 million positions, does not change any decision: all three rows are one or two overnight runs.

Unlike self-play at a fixed time control, datagen throughput *is* proportional to processor speed, so cloud cost per billion positions depends on how the rented cores compare to an M4 Max performance core. No primary source was found that benchmarks a chess engine's node rate across Apple M4, Graviton4, AMD Genoa and Sapphire Rapids, so the table below assumes they are equal, which flatters the cloud.

| Instance | $/1B positions at 400 pos/core-sec, spot | On demand |
|---|---|---|
| AWS `c8g.48xlarge` | **9** | 28 |
| AWS `c7a.48xlarge` | 11 | 36 |
| AWS `c7i.48xlarge` | 21 | 62 |
| GCP `c4-standard-288` | 41 | 69 |

**A billion positions costs nine dollars of Graviton4 spot time and about six hours of wall clock on 192 cores.** On the Mac Studio the same billion positions is 58 hours at twelve cores and roughly $1.50 of electricity. So cloud datagen is not expensive in absolute terms, it is just unnecessary: the thing it buys is finishing a two-night job in an afternoon, and the project's own bottleneck is a person working a few hours a week, not a machine working overnight. The one scenario that inverts this is a dataset regeneration sprint, where a stronger engine rescores several hundred million positions and the whole set must be remade at once. If that day comes, nine dollars is the right answer and it should be spent without hesitation.

### Pricing a GPU for one training run

A 105-teraflop job on a network of 98,689 parameters fits on the smallest accelerator anyone rents. Twenty-four gigabytes of video memory is already absurd overkill, and the practical selection criterion is not the GPU at all but whether the instance has enough attached RAM to cache the dataset and enough vCPUs to feed `bullet`'s loader, which `1_simple.rs` sets to two threads [2].

The wall-clock column below is an assumption, not a measurement: **no primary source publishes `bullet`'s throughput on any hardware, so the 20, 45 and 90 minute columns bracket a plausible range rather than reporting one** [3]. Prices read 2026-09-16; AWS is us-east-1, Linux, from the same feeds as the CPU table [10], RunPod from its pricing page's embedded product data (page `dateModified` 2026-09-13) [17], Lambda from its GPU cloud page, per GPU per hour and before sales tax [18].

| Option | GPU | $/hr | 20-minute run | 45-minute run | 90-minute run |
|---|---|---|---|---|---|
| RunPod, Community Cloud | RTX A5000 24 GB | 0.16 | **$0.05** | $0.12 | $0.24 |
| RunPod, Community Cloud | RTX 4090 24 GB | 0.34 | $0.11 | $0.26 | $0.51 |
| RunPod, Community Cloud | L4 24 GB | 0.44 | $0.15 | $0.33 | $0.66 |
| RunPod, Secure Cloud | RTX 4090 24 GB | 0.74 | $0.25 | $0.55 | $1.11 |
| Lambda | Quadro RTX 6000 24 GB | 0.69 | $0.23 | $0.52 | $1.03 |
| AWS `g6.xlarge`, spot | L4 24 GB | 0.5691 | $0.19 | $0.43 | $0.85 |
| AWS `g6.xlarge`, on demand | L4 24 GB | 0.8048 | $0.27 | $0.60 | $1.21 |
| AWS `g5.xlarge`, on demand | A10G 24 GB | 1.0060 | $0.34 | $0.75 | $1.51 |

**One training run costs between five cents and a dollar and a half, and the spread across the whole table is smaller than the uncertainty in the run time.** That collapses the decision. Do not evaluate GPU vendors. If Metal fails, take whichever of these is quickest to get a shell on, run the network, and move on; the difference between the cheapest and the dearest row is less than the cost of an hour spent choosing. The only real friction is uploading the dataset, and viriformat's roughly 8.5 bytes per position keeps 100 million positions under a gigabyte [7].

Vast.ai was checked and is excluded on sourcing grounds. Its pricing page publishes no rates at all by design, stating only that "Prices are set by the market, not by Vast" across three tiers [19]. A snapshot of its public offers endpoint showed consumer cards between roughly 9 and 63 cents an hour, but that is a marketplace reading at one instant from a sample of a few dozen listings, not a rate card, and this note will not treat it as a price [19].

## Marginal cost of the Mac Studio

Apple publishes measured idle and maximum power for Mac Studio configurations, taken at the wall and including power-supply losses, with "max" defined as "Running a compute-intensive test application that maximizes processor usage and therefore power consumption" at a 20.2 °C ambient [12]. **Apple publishes no row for the 16-core CPU / 40-core GPU / 64 GB configuration.** The nearest documented one, the 14-core CPU / 32-core GPU / 36 GB M4 Max, is 6 W idle and 145 W maximum [12]. The 480 W figure on the tech-specs page is "maximum continuous power", a chassis and power-supply electrical rating that is identical across the M4 Max and M3 Ultra models, and it is not a draw figure; do not use it for cost arithmetic [12].

So 145 W is an over-estimate for a twelve-thread CPU-only workload with the 40-core GPU idle, and an under-estimate for the larger die under a Metal training run. Treating it as a ceiling:

| Load assumption | $/hour at 18.34 ¢/kWh (U.S. avg) | 24-hour test | 1,000 hours |
|---|---|---|---|
| 80 W (twelve engine threads, GPU idle) | $0.0147 | $0.35 | $15 |
| 110 W | $0.0202 | $0.48 | $20 |
| 145 W (Apple's documented max) | $0.0266 | $0.64 | $27 |

At California's 34.74 ¢/kWh, multiply by 1.89: $0.0504 an hour and $1.21 for a 24-hour test at 145 W [14]. Both rates are EIA bundled retail averages for June 2026 and are not marginal rates; a household on a tiered or time-of-use plan can face considerably more at peak [14]. **The 80 W and 110 W rows are estimates**, interpolated between Apple's 6 W idle and 145 W all-out figures for the smaller die; only the 145 W row is published, and even that is for a different configuration.

Per core-hour at 145 W and the U.S. average, sixteen cores cost 0.17 cents. The cheapest cloud core-hour in this survey is 1.24 cents on Graviton4 spot [10]. The Mac Studio is about 7.5 times cheaper per core-hour than the cheapest rentable core, before counting that a spot instance can be reclaimed mid-test and that an interrupted sequential probability ratio test loses nothing but also gains nothing.

## What must be measured

Five numbers in this note are estimates and the machine settles all five. Each has an exact command.

**1. Do twelve concurrent engines actually land on the twelve performance cores?** Start a twelve-way match, then run `sudo powermetrics --samplers cpu_power -i 1000 -n 10` and read the per-cluster residency. If the efficiency cluster is busy, the scheduler is not doing what this note assumes and the concurrency has to come down. There is no `fastchess` flag that fixes it, since `-use-affinity` is a no-op on macOS [4].

**2. The pentanomial pair standard deviation, at both time controls.** Every game count in this note scales with its square. Run a 2,000-game match of the engine against itself with the intended opening book at 10+0.1 and read the `Ptnml(0-2)` line out of `fastchess`'s output, then compute `sd = sqrt(sum(p_i * (i/4 - mean)^2))`. Repeat at 60+0.6. Replace the two assumed ranges (0.22-0.26 and 0.16-0.18) with the measured values and recompute the tables with `stats.py`.

**3. Average game length in moves per side.** Falls out of the same match from the PGN. It sets the seconds-per-game row that the whole throughput table is built on.

**4. Datagen positions per second per core.** Build akimbo `v1.0.0` with `cargo build --release --features datagen` and run the binary with a thread count as its only argument; it reads `dfrc.epd` and prints `fens/sec` per thread [8]. Ten minutes of work, and it replaces the entire three-row estimate table. Note it measures akimbo's node rate, not this project's, so treat it as an upper bound for a first engine and a lower bound once this engine is tuned.

**5. Does `bullet --features metal` build, and does it train?** This is issue #16 and the source cannot answer it. `cargo build --release --package bullet_lib --features metal --example 1_simple`, then run it against a small dataset and time one superbatch. A superbatch is 100,007,936 positions, so superbatch time times 40 is the run time [2]. Stay on the `ValueTrainerBuilder` path and avoid anything that touches `DefaultDevice` [3].

A sixth thing is worth checking while the toolchain is out: whether the Rust engines in the reference-engines note build at all on this machine, which that note flags as an open gap because no Rust toolchain was installed when it was written [`reference-engines.md`].

## Open questions

- **How do rented cores compare to an M4 Max performance core on chess node rate?** The datagen costing assumes parity between Apple M4, Graviton4 (Neoverse V2), AMD Genoa and Intel Sapphire Rapids, and no primary benchmark of an engine's node rate across those four was found. Resolvable for about a dollar: run the same engine's `bench` command on a spot instance of each and compare against the Mac Studio.
- **What is `bullet`'s actual training throughput on Metal versus CUDA?** Nobody publishes it, including `bullet` [3]. The spike in item 5 above produces the Metal half; the CUDA half would cost one rented hour.
- **Does the Metal backend produce numerically identical networks to CUDA?** Untested by anyone, since CI never builds it [3]. `MPSMatrixMultiplication` and a hand-written CUDA GEMM need not agree bit for bit, and a small divergence in a quantised network is the kind of thing that costs a weekend. Training the same dataset on both and comparing the quantised output files would settle it.
- **Is the drawish pentanomial anchor from `stats.py` representative of anything this project will run?** It is one worked example in one file, chosen because it is primary rather than because it matches the intended conditions [1]. Measurement 2 replaces it.
- **Would a second-hand machine beat cloud for the endgame regime?** Not costed here. Once patches are worth 2 to 3 Elo and tests run three days, a used many-core desktop bought outright may beat both the Mac Studio and any rental, and the comparison should be made on cost per core-hour amortised over its life rather than on sticker price.
- **Which machine did `reference-engines.md` actually verify Weiss's build on?** That note's build table is headed "Verified locally on the M4 Max (arm64, Apple clang 21.0.0)", but the Apple Silicon Mac available on 2026-09-16 reports itself as an M1 Pro with six performance and two efficiency cores [16], and the Mac Studio has not arrived. The Weiss build result itself is unaffected, since it is a compile-and-respond-to-`uci` check rather than a performance measurement, but the heading is wrong and should be corrected to name the machine it ran on.
- **What does an interrupted spot instance cost a sequential probability ratio test?** Nothing in principle, since results accumulate and the test resumes, but no framework surveyed here handles partial-pair results cleanly, and pentanomial statistics need complete pairs. Only matters if cloud games are ever bought.

## Sources

1. OpenBench (AndyGrant), GPL-3.0, cloned 2026-09-16 at commit `6b63eb0c31` (`master`, 2026-09-08). `OpenBench/stats.py`, 173 lines: `TrinomialSPRT` at line 33, `PentanomialSPRT` at line 52 ("Implements https://hardy.uhasselt.be/Fishtest/normalized_elo_practical.pdf", `nelo_divided_by_nt = 800 / math.log(10)`), `Elo` at 74, `MLE_tvalue` at 139, and the `__main__` block giving `R5 = (39, 8843, 26675, 9240, 44)`, `R3 = (22569, 44137, 22976)`, `elo0, elo1 = (0.50, 2.50)`. `OpenBench/workloads/create_workload.py` lines 169-170: `test.lowerllr = math.log(test.beta / (1.0 - test.alpha))`, `test.upperllr = math.log((1.0 - test.beta) / test.alpha)`. `OpenBench/workloads/verify_workload.py` `verify_sprt_bounds` requires only `[float1, float2]` with the first strictly below the second. `Templates/OpenBench/create_workload.html` line 159 declares `test_bounds` with no default value; `Config/config.json` contains no bounds, confidence or time-control defaults. `Client/worker.py` line 93 `self.physical_cores = psutil.cpu_count(logical=False)` and line 112 `self.threads = int(args.threads) if args.threads != 'auto' else self.physical_cores`; `scale_time_control` at line 838 multiplies base and increment by an NPS-derived scale factor and appends `timemargin=250`; `determine_scale_factor` at line 897. All game counts in this note were produced by importing this `stats.py` unmodified and calling `PentanomialSPRT` and `MLE_tvalue` directly. https://github.com/AndyGrant/OpenBench
2. `bullet` (jw1912), MIT, `examples/progression/1_simple.rs`, 63 lines, read 2026-09-16 at commit `2ea3d2d0f7`. `hl_size = 128`, `superbatches = 40`, `initial_lr = 0.001`, `final_lr = 0.001 * 0.3^5`, `wdl_proportion = 0.75`, `eval_scale: 400.0`, `batch_size: 16_384`, `batches_per_superbatch: 6104`, `lr::CosineDecayLR`, `AdamW`, `Chess768` inputs, `.screlu()`, `SavedFormat` quantising `l0w`/`l0b` at 255, `l1w` at 64 and `l1b` at `255 * 64`, `LocalSettings { threads: 2, batch_queue_size: 32 }`. https://github.com/jw1912/bullet/blob/main/examples/progression/1_simple.rs
3. `bullet` source, cloned 2026-09-16 at commit `2ea3d2d0f7e597b0d645f6e8040cf37818f51bce` (`main`, 2026-09-15, "Add interleave option for viriformat loader (#545)"). `crates/bullet_lib/Cargo.toml` declares mutually exclusive `cuda`, `rocm` and `metal` features. `crates/bullet_lib/src/nn.rs` selects `ExecutionContext` per feature and defaults to `bullet_gpu::runtime::mock::MockGpu`; a `compile_error!` forbids `metal` alongside `cuda` or `rocm`. `crates/trainer/src/run.rs` lines 22-29 define `DefaultDevice` with `cuda` and `rocm` arms only, falling through to `Device<runtime::mock::MockGpu>` with no `metal` arm. `crates/gpu/src/runtime/mock.rs` line 18: "This is a mock runtime! It can't actually do anything! You need to enable either the `cuda` or `rocm` features!". `crates/gpu/src/runtime/metal.rs` is 469 lines against `cuda.rs` 545 and `rocm.rs` 504, uses `objc2`, `objc2-metal` and `objc2-metal-performance-shaders`, sets `dialect: Dialect::Msl` at line 112, compiles Metal Shading Language at runtime in `program_compile` at line 320, and issues GEMM through `MPSMatrixMultiplication::initWithDevice_transposeLeft_...` at line 414; it contains no `unimplemented!`, `todo!` or `panic!`. `crates/gpu/src/runtime.rs` line 9 gates the module on `feature = "metal"` and line 10 adds `compile_error!("the `metal` feature requires macOS")`. `crates/gpu/src/runtime/dialect.rs`, 40 lines, defines `Dialect::Msl` with distinct `reinterpret_cast`, `make_vec`, `atomic_add` and `pow` emission. `.github/workflows/checks.yaml` has three jobs, all `runs-on: ubuntu-latest`: clippy with `--features=cuda,rocm`, `cargo test --workspace` with no features, and `cargo fmt --check`. `docs/2-getting-started.md` gives the Metal section in full as "For users on macOS. / Enable the `metal` feature", against four bullets for CUDA and five for ROCm, and states of `bullet-utils` that "This does **not** require CUDA or HIP." No throughput or benchmark figure appears anywhere in the README, `docs/` or the source. https://github.com/jw1912/bullet
4. `fastchess` (Disservin), MIT, cloned 2026-09-16 at commit `60d7a7a26c` (`master`, 2026-09-12). `man.md`: `-concurrency (1|N)` "Play N games concurrently, limited by the number of hardware threads. Default value is 1"; `-force-concurrency` "Ignore the hardware concurrency limit"; `-games (2|N)` "Setting this higher than 2 does not provide meaningful results"; `-sprt elo0= elo1= alpha= beta= model=(normalized|logistic|bayesian)` with `normalized` the default; `-use-affinity [CPUS]`; `restart=(off|on)` "default is off"; `timemargin=N`; the worked example `-resign movecount=3 score=600 -draw movenumber=34 movecount=8 score=20`. `app/src/cli/sanitize.cpp` `adjustConcurrency` fills a non-positive `-concurrency` from `std::thread::hardware_concurrency()` and throws "Error: Concurrency exceeds number of CPUs" above it. `app/src/cli/cli.cpp` line 705 sets `-quick` concurrency to `max(1, hardware_concurrency() - 2)`. `app/src/affinity/affinity.hpp`, `#elif defined(__APPLE__)` branch: `setThreadAffinity` is `// Not implemented. return false;` and `setProcessAffinity` is commented-out `thread_policy_set` code ending `// do nothing for now, is affinity_tag supposed to be a mask or the core number? return false;`. `app/src/affinity/cpuinfo/cpuinfo_mac.hpp` opens `// Some dumb code for macOS, setting the affinity is not really supported.` and maps every index from `std::thread::hardware_concurrency()` to its own physical core. https://github.com/Disservin/fastchess
5. `fastchess` README, "Extensively tested for high concurrency (with up to 250 threads) and short time controls (0.2+0.002s), it exhibits minimal timeout issues, with only 10 matches out of 20,000 experiencing timeouts." Read 2026-09-16. https://github.com/Disservin/fastchess
6. `bulletformat` (jw1912), MIT, cloned 2026-09-16. `src/chess.rs` line 9 `#[repr(C)]`, line 11 `pub struct ChessBoard`, line 21 `const _RIGHT_SIZE: () = assert!(std::mem::size_of::<ChessBoard>() == 32);`. https://github.com/jw1912/bulletformat
7. `viriformat` (cosmobobak), cloned 2026-09-16. `src/dataformat.rs` line 253: `pub struct Game { pub initial_position: marlinformat::PackedBoard, pub moves: Vec<(Move, marlinformat::util::I16Le)> }`; line 260 `const SEQUENCE_ELEM_SIZE: usize = size_of::<Move>() + size_of::<marlinformat::util::I16Le>()` (four bytes), line 262 a null terminator of that size, line 315 `MAX_SPLATTABLE_GAME_SIZE: usize = 512`. `PackedBoard` is marlinformat's 32-byte record. The 8.5 bytes per position figure is this note's arithmetic (32 + 4 x plies + 4, over roughly half the plies written), not a published figure. https://github.com/cosmobobak/viriformat
8. akimbo (jw1912), MIT, cloned 2026-09-16 at tag `v1.0.0` (commit `3bfea08c8c`, 2024-03-26). `src/datagen.rs`, 294 lines: `ADJ_WIN_SCORE = 3000`, `SOFT_NODE_LIMIT = 5_000`, `HARD_NODE_LIMIT = 1_000_000`, `HARD_TIMEOUT_MS = 10_000`, `DATA_WRITE_RATE = 16`; `8 + (self.rng() % 2)` random opening plies at line 213; positions written only when `!bm.is_capture() && !pos.in_check()`; output through `bulletformat::ChessBoard::from_raw`. The throughput print at line 170: `println!("#[{}] written games {} fens {} fens/sec {:.0}", self.id, self.games, self.fens, self.fens as f32 / self.start_time.elapsed().as_secs_f32())`. `src/main.rs` lines 21-25: under `#[cfg(feature = "datagen")]`, the binary reads a thread count from `std::env::args().nth(1)` and calls `datagen::run_datagen(threads, None, Some("dfrc.epd"))`. https://github.com/jw1912/akimbo/tree/v1.0.0
9. Viridithas (cosmobobak), MIT at this tag, cloned 2026-09-16 at tag `v20.0.0` (commit `0631113e22`, 2026-06-27). `src/datagen.rs`, 1,567 lines: `RANDOM_MOVES_ROOT = 8` at line 52, `DataGenOptions::new` at line 112 defaulting `nodes: 25_000`, `num_threads: 1`, `generate_dfrc: true`; `soft_limit: options.nodes` and `hard_limit: options.nodes * 8` at lines 469-470; the throughput print at line 650, `" |> FENs generated: {fens} (FENs/sec = {fps:.2})"`, alongside an estimated time remaining and completion time. https://github.com/cosmobobak/viridithas/tree/v20.0.0
10. AWS EC2 pricing, read 2026-09-16, region US East (N. Virginia) / us-east-1, Linux, shared tenancy. The rendered pages at `aws.amazon.com/ec2/pricing/on-demand/` and `aws.amazon.com/ec2/spot/pricing/` are JavaScript applications whose served HTML contains no price strings, so the figures were taken from the AWS-owned JSON feeds those pages query: `https://b0.p.awsstatic.com/pricing/2.0/meteredUnitMaps/ec2/USD/current/ec2-ondemand-without-sec-sel/US East (N. Virginia)/Linux/index.json`, stamped `hawkFilePublicationDate: 2026-09-10T19:55:14Z`, and `https://website.spot.ec2.aws.a2z.com/spot.json`, which carries no stamp and was read at 2026-09-16T17:39Z. On demand: `c7i.16xlarge` 2.8560, `c7i.48xlarge` 8.5680, `c7a.16xlarge` 3.2845, `c7a.48xlarge` 9.8534, `c8g.16xlarge` 2.5523, `c8g.48xlarge` 7.6570, `g6.xlarge` 0.8048, `g6.2xlarge` 0.9776, `g5.xlarge` 1.0060, `g5.2xlarge` 1.2120. Spot at the same reading: `c7i.16xlarge` 0.9171, `c7i.48xlarge` 2.9545, `c7a.16xlarge` 1.0661, `c7a.48xlarge` 3.1045, `c8g.16xlarge` 0.7035, `c8g.48xlarge` 2.3746, `g6.xlarge` 0.5691, `g6.2xlarge` 0.8424, `g5.xlarge` 0.5460, `g5.2xlarge` 0.7993. Threads per core from the static documentation table at https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/cpu-options-supported-instances-values.html: `c7i` two threads per core, `c7a` one, `c8g` one (Graviton4 has no simultaneous multithreading). GPU specifications from https://aws.amazon.com/ec2/instance-types/g6/ and https://aws.amazon.com/ec2/instance-types/g5/: `g6` is NVIDIA L4 24 GB, `g5` is NVIDIA A10G 24 GB. https://aws.amazon.com/ec2/pricing/on-demand/
11. Google Cloud C4, read 2026-09-16, region us-central1 (Iowa). The rendered pricing pages are likewise JavaScript applications; figures come from the unauthenticated SKU search endpoint the pricing page itself queries, `https://cloud.google.com/__/json/searchskus?...&currency=USD&region=us-central1`. C4 is priced as separate components: "C4 Instance Core running in Americas" (SKU `4AA0-9497-B351`) at 0.03465 USD per vCPU hour and "C4 Instance Ram running in Americas" (SKU `E259-34D3-C50F`) at 0.003938 USD per gibibyte hour, with Spot Preemptible equivalents (`F8D5-5FD0-DFD1`, `6DF6-6CCD-40BB`) at 0.02074 and 0.002357. Derived: `c4-standard-48` (48 vCPU, 180 GB) 2.3720 on demand and 1.4198 spot; `c4-standard-288` (288 vCPU, 1,080 GB) 14.2322 and 8.5187. The C4 footnote at https://cloud.google.com/compute/docs/general-purpose-machines reads "A CPU uses two threads per core, and a vCPU represents a single thread", so `c4-standard-288` is 144 physical cores. The Cloud Billing Catalog API returns HTTP 403 without a key and could not be used as a cross-check. https://cloud.google.com/compute/vm-instance-pricing
12. Apple, Mac Studio power. https://support.apple.com/en-us/102027 ("Mac Studio power consumption and thermal output (BTU) information"), read 2026-09-16: Mac Studio (2025) with M4 Max, 14-core CPU / 32-core GPU / 36 GB / 512 GB SSD, idle 6 W (20 BTU/h), maximum 145 W (495 BTU/h). Apple's definitions, verbatim: maximum is "Running a compute-intensive test application that maximizes processor usage and therefore power consumption. No external peripherals are attached during testing"; idle is "the power used with only Finder open, using the default power management settings"; measurements are at the wall at 20.2 °C ambient and include power-supply losses. **Apple publishes no row for the 16-core CPU / 40-core GPU / 64 GB configuration.** https://support.apple.com/en-us/122211 ("Mac Studio (2025) Tech Specs") gives "Maximum continuous power: 480W", which is a chassis and power-supply electrical rating, identical for the M3 Ultra model and for the 2026 M5 Mac Studio, and not a consumption figure. The Mac Studio Product Environmental Report (March 2025) gives Off 0.19 W, Sleep 1.2 W and idle-with-display-on 5.5 W at 100 V, a power-supply efficiency of 93.4%, and a total product footprint of 276 kg CO2e for the M4 Max with 512 GB, of which 41% is product use over a modelled four-year life; it publishes no maximum-power figure and no annual kilowatt-hour figure. https://www.apple.com/environment/pdf/products/desktops/Mac_Studio_PER_March2025.pdf
13. Apple, M4 Max core split. https://support.apple.com/en-us/121554 (MacBook Pro 16-inch, 2024 tech specs), read 2026-09-16, verbatim: "16-core CPU with 12 performance cores and 4 efficiency cores" (and "14-core CPU with 10 performance cores and 4 efficiency cores"). The Mac Studio specs page spells the split out only for the 14-core option, which is why this claim cites 121554. Memory bandwidth, same sources: 546 GB/s for the 16-core CPU / 40-core GPU part, 410 GB/s for the 14-core / 32-core part, both stated as peak figures.
14. U.S. Energy Information Administration, Electric Power Monthly, read 2026-09-16. Table 5.3, "Average Price of Electricity to Ultimate Customers: Total by End-Use Sector, 2016 - June 2026 (Cents per Kilowatthour)": U.S. residential average 18.34 ¢/kWh for June 2026. Table 5.6.A, by state for June 2026: California residential 34.74 ¢/kWh. EIA notes that "values for 2026 and 2025 are preliminary estimates based on a cutoff model sample". Both are bundled retail averages across all customers, not marginal or time-of-use rates. https://www.eia.gov/electricity/monthly/epm_table_grapher.php?t=epmt_5_3 and https://www.eia.gov/electricity/monthly/epm_table_grapher.php?t=epmt_5_6_a
15. Hetzner, read 2026-09-16. Specifications from https://www.hetzner.com/dedicated-rootserver/matrix-ax/ and https://www.hetzner.com/cloud/general-purpose/; prices from the price feed those pages render, `https://www.hetzner.com/_resources/app/data/app/live_data_prices.json`, since the rendered markup carries empty price slots. Dedicated AX line, Falkenstein and Helsinki, monthly and one-off setup: AX41 (Ryzen 5 3600, 6 cores, 64 GB DDR4) USD 67.10, no setup; AX42 (Ryzen 7 PRO 8700GE, 8 cores, 64 GB DDR5 ECC) USD 117.10 plus USD 59; AX102 (Ryzen 9 7950X3D, 16 cores, 128 GB DDR5 ECC) USD 302.10 plus USD 149; AX162 (EPYC 9454P, 48 cores, 128 GB DDR5 ECC registered) USD 722.10 plus USD 359. Cloud CCX (dedicated vCPU), EU locations: CCX13 2 vCPU USD 50.49, CCX23 4 vCPU USD 101.49, CCX33 8 vCPU USD 162.99, CCX43 16 vCPU USD 325.49, CCX53 32 vCPU USD 629.49, CCX63 48 vCPU USD 1006.99, all per month and excluding the primary IPv4 address at USD 0.60 a month. Monthly figures are caps: "will never exceed its monthly price cap", with hourly billing if a server is deleted mid-month. Dedicated-server traffic "is unlimited and free of charge", except with the 10-gigabit uplink add-on where outgoing traffic above 20 TB is billed per terabyte. Prices are net; Hetzner's own VAT table gives Germany 19%, Finland 25.5%, USA 0%. The AX52 named in issue #7 no longer exists: `https://www.hetzner.com/_resources/app/data/app/live_data_products.json` lists AX41, AX42, AX102 and AX162 as the entire AX line, and the AX52 page redirects to the server finder. The primary IPv4 for a dedicated server is USD 1.90 a month. https://www.hetzner.com/dedicated-rootserver/ and https://www.hetzner.com/cloud/
16. Local verification, 2026-09-16, on the Apple Silicon Mac available today, which is an **M1 Pro (6 performance + 2 efficiency cores), not the M4 Max**. `sysctl` reports `hw.ncpu: 8`, `hw.physicalcpu: 8`, `hw.logicalcpu: 8`, `hw.perflevel0.physicalcpu: 6` with `hw.perflevel0.name: Performance`, and `hw.perflevel1.physicalcpu: 2` with `hw.perflevel1.name: Efficiency`; `os.cpu_count()` returns 8. This establishes the reporting behaviour that the note relies on, that on Apple Silicon the logical, physical and total core counts are all equal to the sum of performance and efficiency cores, and only the `perflevel` keys separate them. `man taskpolicy` confirms that the available scheduling controls are `-c background|utility|maintenance` quality-of-service clamps and I/O policies, with no facility for binding a process to a chosen core. **No figure anywhere in this note was measured on an M4 Max.**
17. RunPod pricing, read 2026-09-16 from https://www.runpod.io/pricing, whose visible table renders one tier at a time; both tiers were taken from the page's own embedded product structured data, `dateModified` 2026-09-13. Community Cloud per hour: RTX A5000 24 GB 0.16, RTX 3090 24 GB 0.22, RTX 4090 24 GB 0.34, A40 48 GB 0.35, L4 24 GB 0.44. Secure Cloud: RTX A5000 0.27, RTX 3090 0.50, RTX 4090 0.74, A40 0.49, L4 0.49. The Secure Cloud figures match the rendered table, which also gives RTX 4090 at 41 GB RAM and 6 vCPU, A5000 at 25 GB and 9 vCPU, L4 at 50 GB and 12 vCPU, A40 at 50 GB and 9 vCPU. https://www.runpod.io/pricing
18. Lambda GPU cloud pricing, read 2026-09-16 from https://lambda.ai/service/gpu-cloud, single-GPU tab, per GPU per hour, footnoted "plus applicable sales tax/VAT/GST": Quadro RTX 6000 24 GB 0.69, A6000 48 GB 1.09, A10 24 GB 1.29, A100 PCIe 40 GB 1.99, A100 SXM 40 GB 1.99, GH200 96 GB 2.29, H100 PCIe 80 GB 3.29, H100 SXM 80 GB 4.29, B200 SXM6 180 GB 6.99. No L40S and no RTX 6000 Ada appear on the current page. https://lambda.ai/service/gpu-cloud
19. Vast.ai pricing page, read 2026-09-16: no rates are published. The page states "Prices set by supply and demand across 40+ data centers", "Prices are set by the market, not by Vast", and describes three tiers (on demand, interruptible at "50%+ cheaper", and reserved at "up to 50% off" on one, three or six month terms) with per-second billing across "68+ GPU Types". A snapshot of Vast's own public offers endpoint, `https://console.vast.ai/api/v0/bundles/`, returned 64 live listings with RTX 3090 from $0.088, RTX 4090 from $0.121 and RTX 5090 from $0.213 per GPU hour. That is a marketplace reading at one instant from a small sample and is explicitly not treated as a price in this note. https://vast.ai/pricing
