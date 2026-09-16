# Reference engines: which open-source engines to study, and for what

Researched 2026-09-16.

Scope: which open-source chess engines this project should read, which subsystem each one is the clearest reference for, and what each licence permits. The question is issue #5. The policy is read freely, copy nothing, so the licence work here is load-bearing rather than decorative. Search techniques are covered in `docs/research/search-survey.md` and the classical-to-NNUE evaluation path in `docs/research/evaluation-path.md`; this note does not repeat either, it says where to read them. Every licence claim below comes from the `LICENSE`, `COPYING` or `Copying.txt` file inside the repository, read directly, not from the GitHub sidebar label. Every file path and line count was verified by cloning the repository on 2026-09-16; line counts are of the file as committed, comments and blank lines included.

## Answer

**Read four engines: Viridithas at tag `v20.0.0`, akimbo at its tags, Weiss, and Hobbes. Only Weiss is off limits for copying, and it is the one you will want to copy from least.**

The single finding that decides this note is a licence one, and it inverts the obvious ranking.

**Viridithas was MIT-licensed through v20.0.0 and relicensed to AGPL-3.0-only afterwards.** Commit `2001ac6` on 2026-07-06 is titled "Relicense to AGPL-3.0-only (as of v21)", and the `LICENSE` file at tag `v20.0.0` (tagged 2026-06-27) still reads "MIT License / Copyright (c) 2022-2025 Cosmo Bobak" [1][2]. The README confirms it in the author's own words: "Versions prior to 20.0.0 were released under the MIT License and remain available under those terms" [1]. So there exists a frozen, MIT-licensed, 25,946-line Rust engine that measures 3613 ±10 on the CCRL 40/15 list on one core [19], with a heavily commented search, a full datagen module, and hand-written ARM NEON inference. That is not a thing to read carefully and reimplement. That is a thing to read, port, and attribute. Nothing else in the field comes close to that combination.

The README's phrasing and the commit message disagree slightly about whether v20.0.0 itself is included. The commit says "as of v21" and the `LICENSE` file at the `v20.0.0` tag is MIT, so v20.0.0 inclusive is MIT and current `master` is AGPL [1][2][3]. Pin to the tag, never to `master`, and record the commit hash `0631113e` in any file you derive from it.

**akimbo earns its slot on shape, not strength.** Its tags mirror this project's roadmap almost step for step, in MIT-licensed Rust, at sizes a person can hold in their head [5][6]:

| akimbo tag | Date | Total Rust lines | What it is |
|---|---|---|---|
| `v0.4.1-pst-only` | 2023-08-04 | 1,165 in 4 files | Material and piece-square tables only, complete working engine [7] |
| `v0.5.0` | 2023-08-12 | 2,043 in 15 files | Better hand-crafted evaluation, plus an in-repo texel tuner and datagen crate [8] |
| `v0.6.0` | 2023-09-24 | 1,401 in 8 files | First network, `768 -> 256x2 -> 1`, scalar inference inline in `position.rs` [9] |
| `v1.0.0` | 2024-03-26 | 3,233 in 13 files | Last release; adds `src/datagen.rs` at 294 lines [10] |

A complete engine at 1,165 lines that reached 2841 on CCRL Blitz is a better first read than any amount of well-organised 20,000-line code [5]. The `v0.5.0` tuner is 55 lines of Adam gradient descent and finite-difference fitting of the `K` constant, which is exactly step 3 of the evaluation path, in Rust, under MIT [8].

**Weiss is the hand-crafted evaluation reference and nothing else has replaced it.** No MIT engine in the field ships a readable, tuned, classical evaluation with named functions per term, because the MIT engines are all NNUE engines whose classical code is gone or was never written. Weiss keeps `src/evaluate.c` at 587 lines with one function per idea (`EvalPawns`, `EvalPiece`, `EvalKings`, `EvalPassedPawns`, `EvalThreats`, `ScaleFactor`), `src/psqt.c` at 95 lines of tapered `S(mg, eg)` tables, and `src/tuner/tuner.c` at 593 lines of texel tuner, in C [16]. It is GPL-3.0 [15], so it is read-only for this project, and that is the right relationship anyway: the evaluation-path note already concluded that the hand-crafted evaluation is tuition to be deleted later [`evaluation-path.md`], so borrowing someone else's is the one thing that would waste the exercise.

**Hobbes is the living reference and the Apple Silicon one.** MIT, last commit 2026-09-12, 3602 ±12 on CCRL 40/15 on one core, 13,798 lines of Rust [12][13][19]. It matters for two specific things. First, it is the only engine in the pick that is actively developed under a permissive licence, so it is where to look when something in the 2026 frontier is unclear. Second, `src/evaluation/simd/neon.rs` is 220 lines of hand-written `aarch64` intrinsics behind the same vocabulary as its AVX2 and AVX-512 siblings, dispatched through a `Forward` trait with a `Scalar` fallback selected at compile time [13]. On an M4 Max that is the pattern to copy, and akimbo's SIMD is x86 AVX2 only, so it silently falls back to scalar on ARM [11].

Three engines would have been enough for the subsystems. The fourth slot is bought by the licence asymmetry: Viridithas v20.0.0, akimbo and Hobbes are MIT and can be ported, Weiss is GPL and can only be read, and it is worth knowing which is which per file rather than per project.

### Tooling, which is not an engine pick but must be named

`bullet` and `bulletformat` (jw1912) are both MIT and both already chosen by the evaluation-path note [22][23]. `examples/progression/1_simple.rs` is 63 lines and is the `768 -> 128x2 -> 1` recipe verbatim; `examples/simple.rs` is 211 lines and includes a working quantised inference implementation [22]. `bulletformat/src/chess.rs` is 248 lines and owns the 32-byte `ChessBoard` record [23].

For testing, OpenBench (Andrew Grant) is GPL-3.0 and 9,409 lines of Python across a Django server and a worker [24]. Read it for design, not for code. `OpenBench/OpenBench/stats.py` is 173 lines and contains `TrinomialSPRT` and `PentanomialSPRT`, which is the whole sequential probability ratio test (a statistical stopping rule that accepts or rejects a change as soon as the accumulated game results make the decision, rather than at a fixed game count) in one small readable file [24]. `OpenBench/OpenBench/models.py` at 365 lines is the data model for a test queue and is the right thing to read before designing this project's own. `fastchess` (Disservin) is MIT and is the match runner [25].

## Licence status, per engine

Read from the licence file in each repository on 2026-09-16.

| Project | Licence file | Licence | Read? | Copy into an MIT repo? |
|---|---|---|---|---|
| Viridithas `v20.0.0` and earlier | `LICENSE` | MIT [2] | Yes | **Yes**, with the MIT notice and copyright line preserved |
| Viridithas `master` / v21 onward | `LICENSE` | AGPL-3.0-only [1][3] | Yes | No |
| akimbo (all tags) | `LICENSE` | MIT [5] | Yes | **Yes**, with the notice preserved |
| Hobbes | `LICENSE` | MIT [12] | Yes | **Yes**, with the notice preserved |
| Weiss | `COPYING.txt` | GPL-3.0-or-later [15] | Yes | No |
| Weiss `src/pyrrhic/` | `src/pyrrhic/LICENSE` | MIT [17] | Yes | Yes, that subtree only |
| `bullet`, `bulletformat` | `LICENSE` | MIT [22][23] | Yes | Yes |
| OpenBench | `LICENSE` | GPL-3.0-or-later [24] | Yes | No |
| `fastchess` | via GitHub API | MIT [25] | Yes | Yes (external tool anyway) |
| Stockfish | `Copying.txt` | GPL-3.0 [26] | Yes | No |
| Stormphrax, Carp, Berserk, Alexandria, Obsidian, Ethereal, Koivisto, Velvet, Pleco, Rustic | `LICENSE` / `LICENSE.md` | GPL-3.0 [28][29][32][34][27][31] | Yes | No |
| Reckless | `LICENSE` | **AGPL-3.0** [3] | Yes | No |
| Altair | `LICENSE` | MIT [33] | Yes | Yes |
| Motor | **none** | **no licence file anywhere in the repository** [30] | Risky | **No** |

Four of these deserve to be called out as traps rather than table rows.

**Reckless is AGPL, not GPL.** The search survey leans on `Reckless/src/search.rs` as its primary labelled reference and that is still the right file to read, but the licence is the Affero variant, which extends the copyleft obligation to network use [3]. Nothing here runs Reckless over a network, so in practice the reading rule is identical to the GPL one: read it, reimplement the idea, do not transcribe the code. The reason to state it is that "GPL" in this repository's notes should not be assumed to cover Reckless and Viridithas `master`; both are Affero.

**Motor has no licence at all.** No `LICENSE`, `LICENSE.md`, `COPYING` or `UNLICENSE` at the repository root, and no licence statement in `README.md` or the build files [30]. Default copyright applies, which means all rights reserved. It is also C++ rather than Rust, contrary to how it is usually grouped: 31 `.cpp` and `.hpp` files, zero `.rs` files [30]. Reading a public repository is fine; treating it as a source for anything is not. Leave it alone.

**Hobbes's networks have no licence, though the engine does.** The engine repository is MIT [12], but the weight files live in `kelseyde/hobbes-networks`, which has no `LICENSE` file and no licence statement in its README [14]. Contrast Viridithas, which states in its own README that the networks "are dedicated to the public domain under CC0 1.0" and backs it with a CC0 `LICENSE` and an explicit README sentence in `cosmobobak/viridithas-networks` [1][4]. Read Hobbes's inference code freely; do not redistribute its weights.

**akimbo's current network is trained on Leela Chess Zero data and its licence status is murky.** The README says: "akimbo now uses data produced by Leela Chess Zero, however *no official release* will be made with this" [5]. The repository nonetheless ships `resources/net.bin` on `main`, 6,297,664 bytes, with no separate terms [11]. The same README states that "Up to and including version 1.0.0, all data used was self-generated", so the nets at tags `v0.6.0` (`resources/bob.bin`, 394,754 bytes) and `v1.0.0` are self-generated and clean, and the one on `main` is not [5][9][10]. Leela training data is ODbL-1.0 and Leela networks carry no licence at all, as the evaluation-path note established. This does not matter for this project, which generates its own data, but it is exactly the kind of thing to notice before reusing anyone's weights.

### Where the line actually sits

The rule this project operates under, stated plainly so it can be argued with.

An idea is not copyrightable. 17 U.S.C. § 102(b): "In no case does copyright protection for an original work of authorship extend to any idea, procedure, process, system, method of operation, concept, principle, or discovery, regardless of the form in which it is described, explained, illustrated, or embodied in such work" [35]. So late move reductions, null-move pruning, the history heuristic, the accumulator trick, and the texel loss function can all be read in Stockfish and Weiss and reimplemented in an MIT repo without a licence question arising. That is what "read freely, copy nothing" means and it is a real permission, not a grudging one.

A specific expression is copyrightable. Copying a function body, keeping a distinctive variable naming scheme, preserving an unusual control-flow shape, or translating C line by line into Rust while retaining its structure all produce a derivative work. The safe operational test: read the GPL file, close it, write your version from the understanding rather than from the text, and if the result happens to match line for line, rewrite it. If a comment in your code would naturally read "as in Stockfish", write the citation and keep the implementation your own.

Tuned constants sit in a grey area and this note will not pretend otherwise. A single magic number (the LMR formula's divisor, say) is a fact, and facts are not protected. A 768-entry piece-square table tuned over thousands of games is closer to a compiled dataset, and the effort behind it is precisely what copyright and database rights attach to. There is no chess-project precedent that settles this and no court case anyone in the field cites. **Safe practice: never copy a table of tuned numbers out of a GPL or AGPL engine, and never copy a full parameter block.** This costs nothing here, because the evaluation-path note already requires tuning the tables in-house as the learning exercise, and search constants get tuned per engine anyway. Copying a handful of individual scalars from an MIT engine is fine and the notice requirement covers it.

None of the above is legal advice, and no source cited here is a lawyer.

## Subsystem to engine mapping

The core of the note. All paths verified to exist on 2026-09-16; line counts are exact.

| Subsystem | Clearest reference | File(s) | Lines | Copy? |
|---|---|---|---|---|
| Move generation, first pass | akimbo `v0.4.1-pst-only` | `src/position.rs` | 361 [7] | Yes, MIT |
| Move generation, compact Rust | akimbo `main` | `src/position.rs` (`movegen` at line 397), `src/attacks.rs` | 587, 125 [11] | Yes, MIT |
| Move generation, proper legal generator | Hobbes | `src/board/movegen.rs`, `src/board/legal.rs`, `src/board/magics.rs` | 520, 307, 201 [13] | Yes, MIT |
| Move generation, C for contrast | Weiss | `src/movegen.c` | 188 [16] | No, read only |
| Perft harness | Viridithas `v20.0.0` | `src/perft.rs` | 460 [2] | Yes, MIT |
| Search, tiers 1-2 | akimbo `main` | `src/search.rs` | 728 [11] | Yes, MIT |
| Search, tiers 1-2 in C | Weiss | `src/search.c` | 814 [16] | No, read only |
| Search, tiers 3-5 | Viridithas `v20.0.0` | `src/search.rs` | 2,141 [2] | Yes, MIT |
| Search, tiers 3-5 second opinion | Reckless | `src/search.rs` | 1,490 [3] | No, AGPL |
| Move ordering and history | Viridithas `v20.0.0`, Hobbes | `src/history.rs`; `src/search/history.rs`, `src/search/movepicker.rs` | 411, 270 [2][13] | Yes, MIT |
| Transposition table | Viridithas `v20.0.0` | `src/transpositiontable.rs` | 578 [2] | Yes, MIT |
| Static exchange evaluation | Hobbes | `src/search/see.rs` | 199 [13] | Yes, MIT |
| Correction history | Hobbes | `src/search/correction.rs` | 205 [13] | Yes, MIT |
| Time management | Viridithas `v20.0.0` | `src/timemgmt.rs` | 470 [2] | Yes, MIT |
| Hand-crafted evaluation | **Weiss** | `src/evaluate.c`, `src/psqt.c`, `src/evaluate.h` | 587, 95, 62 [16] | **No, GPL** |
| Texel tuner | **akimbo `v0.5.0`** | `tuner/src/tuner/mod.rs`, `tuner/src/tuner/data.rs`, `tuner/src/core/params.rs` | 55, 102, 50 [8] | **Yes, MIT** |
| Texel tuner, second reference | Weiss | `src/tuner/tuner.c`, `src/tuner/tuner.h` | 593, 136 [16] | No, GPL |
| NNUE inference, first net | **akimbo `v0.6.0`** | `akimbo/src/position.rs`, NNUE inline at lines 11-15, 80-96 and 155-168 | 373 total [9] | **Yes, MIT** |
| NNUE inference, structured | akimbo `main` | `src/network.rs` | 210 [11] | Yes, MIT |
| NNUE inference, production | Viridithas `v20.0.0` | `src/nnue/network.rs`, `src/nnue/accumulator.rs`, `src/nnue/network/layers.rs` | 1,989, 572, 475 [2] | Yes, MIT |
| NNUE SIMD on Apple Silicon | **Hobbes** | `src/evaluation/simd/neon.rs`, `src/evaluation/forward/mod.rs`, `src/evaluation/forward/scalar.rs`, `src/evaluation/forward/vectorised.rs` | 220, 38, 114, 187 [13] | **Yes, MIT** |
| NNUE SIMD, second NEON reference | Viridithas `v20.0.0` | `src/nnue/simd.rs` (`neon` module at line 770), `src/nnue/geometry/neon.rs` | 1,182 total [2] | Yes, MIT |
| Trainer configuration | `bullet` | `examples/progression/1_simple.rs`, `examples/simple.rs` | 63, 211 [22] | Yes, MIT |
| Training data format | `bulletformat` | `src/chess.rs`, `src/chess/marlin.rs`, `src/loader.rs` | 248, 141, 109 [23] | Yes, MIT |
| Data generation, minimal | **akimbo `v1.0.0`** | `src/datagen.rs` | 294 [10] | **Yes, MIT** |
| Data generation, minimal, earlier form | akimbo `v0.6.0` | `datagen/src/thread.rs`, `datagen/src/main.rs` | 168, 44 [9] | Yes, MIT |
| Data generation, production | Viridithas `v20.0.0` | `src/datagen.rs` (`generate_on_thread` at line 410), `src/datagen/dataformat.rs`, `src/datagen/dataformat/marlinformat.rs` | 1,567, 488, 213 [2] | Yes, MIT |
| SPRT statistics | **OpenBench** | `OpenBench/OpenBench/stats.py` | 173 [24] | **No, GPL** |
| Test queue data model | OpenBench | `OpenBench/OpenBench/models.py`, `OpenBench/OpenBench/workloads/create_workload.py` | 365, 323 [24] | No, GPL |
| Test worker | OpenBench | `OpenBench/Client/worker.py` | 1,439 [24] | No, GPL |
| Match runner | `fastchess` | external binary | n/a [25] | n/a |

Five entries in that table deserve an explanation, because they are the ones where the obvious choice is wrong.

**Hand-crafted evaluation: Weiss, and there is no permissive alternative.** akimbo's hand-crafted evaluation at `v0.5.0` exists but lives inside `akimbo/src/position.rs` and `tuner/src/core/`, is deliberately minimal, and is not organised to be read as an evaluation [8]. Weiss's is organised exactly that way: one `INLINE int Eval<Thing>(...)` per concept, a pawn cache, a scale factor for drawish endings, and the whole thing 587 lines [16]. Take the structure as a shopping list of terms to implement, take none of the numbers.

**NNUE inference: akimbo at `v0.6.0`, not `main`.** Current `main` is `768x4hm -> 1024x2 -> 1` with king buckets, SCReLU and AVX2 intrinsics [11]. At `v0.6.0` the whole network is fourteen lines inside `position.rs`, with the architecture declared as a bare struct and the forward pass as two scalar loops [9]:

```rust
const HIDDEN: usize = 256;
struct Eval([i16; 768 * HIDDEN], [i16; HIDDEN], [i16; 2 * HIDDEN], i16);
```

and the output as `sum * 400 / 16320`, where 16320 is `QA * QB` with the project's chosen `QA = 255` and `QB = 64`. One caveat that will otherwise cost an afternoon: `v0.6.0` uses clipped ReLU (`i.clamp(0, 255)`), whereas this project plans SCReLU following `bullet`'s `1_simple.rs` default, and the squared activation changes the dequantisation, which is why `main` divides by `QA` once before applying `SCALE / (QA * QB)` and `v0.6.0` does not [9][11][22]. Read `v0.6.0` for the shape, `main` for the SCReLU arithmetic, and Hobbes for how to vectorise it on ARM.

**Texel tuner: akimbo `v0.5.0`, which is 55 lines.** `tuner/src/tuner/mod.rs` has two functions. `optimise_k` fits the sigmoid scaling constant by central finite differences until the derivative falls under a threshold. `gd_tune` is textbook Adam (`b1 = 0.9`, `b2 = 0.999`) over all evaluation parameters, with the learning rate decayed and an early stop when the error stops improving [8]. It is MIT, it is Rust, and it is short enough to read in one sitting. This is the best single artefact found in this whole survey relative to its size.

**Data generation: akimbo, then Viridithas.** akimbo `v1.0.0`'s `src/datagen.rs` is 294 lines and `v0.6.0`'s `datagen/src/thread.rs` is 168, both with the settings visible in one screen (`max_nodes: 1_000_000` as the hard limit, a configurable soft node budget per move) [9][10]. Viridithas `v20.0.0`'s `src/datagen.rs` is 1,567 lines, but the generation loop itself is `generate_on_thread` at line 410, and the surrounding bulk is the useful-later machinery: `run_splat`, `run_rescale`, `run_relabel`, `dataset_stats`, and a marlinformat and viriformat writer in `src/datagen/dataformat.rs` [2]. `RANDOM_MOVES_ROOT = 8` sits at line 52, matching the table in the evaluation-path note. Read akimbo to write the first version, Viridithas to learn what the module grows into. Note that Hobbes is not the datagen reference: its `src/tools/datagen.rs` is 76 lines and only generates random opening positions [13].

**SPRT: OpenBench `stats.py`, and accept that it is GPL.** The statistics here are published mathematics (Wald's sequential probability ratio test, and the pentanomial game-pair model from the Fishtest lineage), so reimplementing from the same formulas is clean, and 173 lines is short enough to understand rather than transcribe [24]. There is no permissively licensed equivalent in the field that this survey found.

## Reading order

Tied to the five search tiers in `docs/research/search-survey.md` and the ten-step sequence in `docs/research/evaluation-path.md`. Weeks are relative, not calendar, and assume a few hours a week.

**Week 1, before writing any code. Read one whole small engine end to end.** akimbo `v0.4.1-pst-only`: 1,165 lines across `src/position.rs`, `src/search.rs`, `src/util.rs` and `src/main.rs` [7]. The point is not to learn technique, it is to see that a complete UCI engine with move generation, alpha-beta, quiescence and a tapered evaluation fits in four files, so that the project's own scope stays honest. Then read Weiss `src/movegen.c` (188 lines) as a contrast in style: C, bitboards, no Rust idioms in the way [16].

**Weeks 2-3, tier 0, move generation and perft.** Hobbes `src/board/movegen.rs`, `src/board/legal.rs` and `src/board/magics.rs` for a real magic-bitboard legal generator with checker and threat computation split out [13]. Compare against akimbo's approach, which generates pseudo-legal moves and validates legality inside `make` by checking whether the king square is attacked after the move (`src/position.rs` line 97) [11]. Both are correct; the akimbo one is shorter, the Hobbes one is faster and more conventional. Pick deliberately. Then Viridithas `v20.0.0` `src/perft.rs` for the harness [2].

**Weeks 4-6, tier 1, the frame.** akimbo `main` `src/search.rs` (728 lines) is the whole of iterative deepening, quiescence, the transposition table probe and move ordering in one file with no indirection [11]. Weiss `src/search.c` (814 lines) is the same material in C with roughly twice akimbo's comment density (123 comment lines against 64) [16][11]. For the table itself, Viridithas `v20.0.0` `src/transpositiontable.rs`, 578 lines, which settles the bit-layout question the search survey left open [2]. Do not open Viridithas `src/search.rs` yet.

**Weeks 7-8, evaluation steps 1 and 3, and this is the Weiss week.** `src/psqt.c` for the tapered table layout, `src/evaluate.c` for which terms exist and how they are decomposed, `src/evaluate.h` for the `EvalInfo` and pawn cache structures, then `src/tuner/tuner.c` [16]. Immediately afterwards read akimbo `v0.5.0` `tuner/src/tuner/mod.rs` and `tuner/src/tuner/data.rs`, which is the same mathematics in 157 lines of MIT Rust and is the version to actually build from [8]. Reading them in that order matters: Weiss shows what to tune, akimbo shows how to tune it.

**Weeks 9-12, tier 2, the big pruners.** Now open Viridithas `v20.0.0` `src/search.rs` and read it against the search survey's tier-2 list rather than straight through [2]. 241 comment lines, so the reasoning is on the page. Take `src/search/parameters.rs` (604 lines) as the answer to "where do all these constants live", which is a structural question worth settling before the constants multiply.

**Weeks 13-16, tier 3 and the evaluation terms.** Hobbes `src/search/see.rs` for static exchange evaluation, `src/search/history.rs` and `src/search/movepicker.rs` for the continuation and capture history tables, `src/search/lmr.rs` (85 lines) for the reduction table [13]. Weiss `src/history.h` (168 lines) as a smaller version of the same idea [16]. Return to Weiss `src/evaluate.c` for the extra evaluation terms and retune with the tuner from week 8.

**Weeks 17-20, datagen and the first network.** akimbo `v1.0.0` `src/datagen.rs` first, because 294 lines is the right size for a first implementation [10]. Then `bullet` `examples/progression/1_simple.rs` (63 lines) and `bulletformat/src/chess.rs` (248 lines) for the format the datagen must emit [22][23]. Then akimbo `v0.6.0` `position.rs` for the inference, in the specific narrow sense described above [9]. The Metal backend spike flagged in the evaluation-path note belongs here, ahead of writing datagen.

**Weeks 21+, tiers 4 and 5, and making the network fast.** Hobbes `src/evaluation/forward/mod.rs` to see the trait-based scalar-versus-vectorised dispatch, then `scalar.rs`, then `vectorised.rs`, then `simd/neon.rs` [13]. Viridithas `v20.0.0` `src/nnue/simd.rs` as the second NEON opinion, including its `dotprod` feature handling [2]. For tier 4 search, Reckless `src/search.rs` for its labelled blocks and Stockfish `src/search.cpp` for the current frontier, both read-only [3][26]. Viridithas `src/datagen.rs`'s `run_relabel` and `run_rescale` become relevant once there is data worth rescoring [2].

The inversion worth stating: the engine to read at tier 1 is akimbo, which is 3475 on CCRL 40/15, and the engine to read at tier 5 is Viridithas or Stockfish, at 3613 and 3629 [19]. Reading the 3600-Elo search in week 4 produces a pile of interacting heuristics with no way to tell which one is broken.

## What not to read, and why

**Stockfish.** GPL-3.0, 25,340 lines across 72 files [26]. Read specific files for specific answers, which is what the search survey does, and do not read it as a model. Two distinct hazards. The code is tuned rather than explanatory: `src/search.cpp` carries dozens of unexplained terms in the late-move-reduction calculation whose only justification is a passed Fishtest run, and copying that style into a young engine produces something nobody can debug. And it is the highest transcription risk in the field, because it is the code a person is most likely to have half-memorised, which is exactly how near-transcription happens by accident into an MIT repo.

**Pleco.** GPL-3.0, Rust, 22,102 lines, and its own README describes it as "a chess Engine & Library derived from Stockfish, written entirely in Rust", adding that "the majority of the code is a direct port of Stockfish's C++ code" [27]. That is a self-declared near-transcription of GPL code. Reading it to learn "how Stockfish looks in Rust" is precisely the path that ends with GPL-derived expression in this repository, because the Rust is already someone else's translation rather than an independent implementation. It is also dormant as an engine: the twelve most recent commits are all Copilot-generated soundness and formatting patches landed on a single day, 2026-02-22, with no engine development [27], and it appears on neither the CCRL 40/15 nor the CCRL Blitz list [19][20]. Skip entirely.

**Rustic.** The GitHub repository is a stub: a single `readme.md` announcing that the project moved to Codeberg, with no source at all [31]. The live repository at `codeberg.org/mvanthoor/rustic` is GPL-3.0 and last committed 2026-06-02, so the project is alive [31], but it is not on the CCRL 40/15 list, and its best Blitz entry is Rustic Alpha 3.0.0 at 1792 ±16 on the list computed 2026-09-12 [20]. An engine 1,200 Elo below this project's target teaches the tier-0 lesson and nothing after it, and akimbo `v0.4.1-pst-only` teaches the same lesson in 1,165 MIT-licensed lines. The genuine value in Rustic is the author's written tutorial at `rustic-chess.org`, which is prose rather than code and is outside this note's scope.

**Stormphrax.** A good engine at 3609 ±9 and a cited source elsewhere in this repository, but the wrong size and shape here [19][28]. The repository is 44,501 lines by raw count, of which 21,993 are `src/` and the rest is vendored `fmt`, `zstd` and `pyrrhic` [28]. It is GPL-3.0, it is C++, and 21,993 lines of templated modern C++ is a worse teacher than the same ideas in Rust at a fifth the size. Read its `src/datagen/datagen.cpp` for the settings table already extracted in the evaluation-path note, and otherwise leave it.

**Ethereal.** GPL-3.0, last commit 2024-06-11, so two and a quarter years dormant [29]. Its value to this project is two documents rather than its source: the 2020 per-technique Elo removal data that the search survey rests on, and `Tuning.pdf`. Both are already extracted. The engine code itself has been overtaken.

**Koivisto.** GPL-3.0, last commit 2023-02-08, three and a half years dead [34]. Historically important as the source of Alexandria's first training data, but there is no reason to read it now.

**Carp.** GPL-3.0, Rust, 12,062 lines, last commit 2025-01-18 and no activity since, at 3455 ±10 [19][32]. It is genuinely well-organised, splitting a `chess` library crate from an `engine` crate, and `chess/src/nnue/mod.rs` at 316 lines is a clean `768 -> 256x2 -> 1` inference. It loses its slot to akimbo `v0.6.0`, which does the same job in fewer lines under MIT rather than GPL. Worth a look only if akimbo's terseness proves unreadable.

**Velvet, Berserk, Alexandria, Obsidian, Altair.** All fine engines, none the clearest reference for anything after the four picks. Velvet is GPL and its `engine/src/search.rs` is 2,061 lines, roughly Viridithas-sized without Viridithas's comments or licence [19]. Berserk, Alexandria and Obsidian are all GPL-3.0 C or C++ [34]. Altair is the near miss: MIT-licensed C++, with `src/evaluation_classic.cpp` at 565 lines and `src/datagen.cpp` at 450, so it is the one permissively licensed classical evaluation in the field [33]. It loses to Weiss on readability (no per-term function decomposition, and its weights were deliberately zeroed for the zero-knowledge experiment) and to akimbo on language. Keep it as the fallback if a port-legal classical evaluation is ever needed in a hurry.

**Motor.** No licence, so nothing can be taken from it under any circumstances, and it is C++ rather than Rust [30]. Its release notes are a legitimate source for the hidden-size Elo ladder, as the evaluation-path note already uses them. The code is not.

## Strength context

CCRL 40/15, "computed on September 10, 2026 with Bayeselo based on 2'443'716 games" [19]. Conditions as printed: "Ponder off, General book (up to 12 moves), 3-4-5 piece EGTB. Time control: Equivalent to 40 moves in 15 minutes on an Intel i7-4770k" [19]. Hash is 256 or 512 MB, set the same for all engines in a match, per the separate conditions page [21]. All rows below are plain 64-bit single-core entries; the 4CPU rows are excluded. The project's target is roughly 3000.

| Engine, single core | CCRL 40/15 | Games | Relative to the 3000 target | In the pick? |
|---|---|---|---|---|
| Stockfish 18 | 3629 ±8 | 3379 | +629 | Read specific files only |
| Reckless 0.9.0 | 3623 ±9 | 2781 | +623 | Second opinion, AGPL |
| **Viridithas 20.0.0** | **3613 ±10** | 2245 | +613 | **Yes** |
| Stormphrax 8.0.0 | 3609 ±9 | 2467 | +609 | No |
| Alexandria 9.0.0 | 3608 ±9 | 2461 | +608 | No |
| Obsidian 16.0 | 3606 ±7 | 3975 | +606 | No |
| Berserk 14 | 3604 ±10 | 2016 | +604 | No |
| **Hobbes 3.0** | **3602 ±12** | 1229 | +602 | **Yes** |
| Motor 0.9.0 | 3572 ±9 | 2313 | +572 | No, unlicensed |
| Ethereal 14.25 | 3566 ±6 | 6675 | +566 | No, dormant |
| Velvet 8.1.1 | 3532 ±9 | 2248 | +532 | No |
| Koivisto 9.0 | 3524 ±6 | 6724 | +524 | No, dead |
| **akimbo 1.0.0** | **3475 ±9** | 2509 | +475 | **Yes** |
| Altair 7.0.0 | 3475 ±11 | 1666 | +475 | Fallback only |
| Carp 3.0.0 | 3455 ±10 | 2165 | +455 | No |
| **Weiss 2.0** | **3264 ±10** | 2128 | +264 | **Yes**, read only |
| Rustic Alpha 3.0.0 | not listed | n/a | Blitz 1792 ±16 [20] | No |
| Pleco | not listed on either list | n/a | unknown | No |

Two things follow. Every engine in the pick is above the target, so none of them is a model for where to stop; they are models for how to build, and the stopping point is this project's own decision. And the spread inside the pick, 3264 to 3613, is the useful part: Weiss reaches 3264 with no network at all, which is the same point the evaluation-path note makes from a different direction, and the 349 Elo from there to Viridithas is roughly what a network plus a decade of search polish buys.

Weiss deserves one note on its 3264. That figure is Weiss 2.0, released 2021-08-04, which is the last tagged release [12]. `master` identifies itself as "Weiss 2.1-dev" and was last committed 2026-08-13, so five years of unreleased development are not reflected in the rating [16]. The code being recommended here is `master`, not the rated binary.

## Build and liveness check

Verified locally on an Apple M1 Pro (arm64, Apple clang 21.0.0) on 2026-09-16. The Mac Studio had not arrived; this is a compile-and-respond-to-`uci` check, not a performance measurement, so the machine does not affect the result.

| Project | Default branch last commit | Last release | Builds here? |
|---|---|---|---|
| Hobbes | 2026-09-12 | 3.0, 2026-07-22 | Not tested, no Rust toolchain installed |
| Viridithas | 2026-09-06 (`master`, AGPL) | v20.0.0, 2026-06-27 (MIT) | Not tested, no Rust toolchain installed |
| Weiss | 2026-08-13 | v2.0, 2021-08-04 | **Yes**, with a caveat |
| akimbo | 2025-02-07 | v1.0.0, 2024-03-26 | Not tested, no Rust toolchain installed |
| `bullet` | 2026-09-15 | none tagged | Not tested |
| OpenBench | 2026-09-08 | none tagged | Not tested |

**Weiss builds and runs on Apple Silicon, but not with the default target.** `make` in `src/` runs the profile-guided-optimisation target and fails with `clang: error: Error in reading profile pgo/default.profdata: No such file or directory`. `make basic` succeeds: a single `gcc -std=gnu11 ... -march=native` invocation producing a 190,008-byte binary that answers `uci` correctly as "Weiss 2.1-dev (c735b8f)" [18]. Use `make basic`.

**The Rust builds are unverified and that is a real gap.** No `cargo`, `rustc` or `rustup` is installed on this machine, and this note declined to install a toolchain as a side effect of a research task. Three of the four picked engines are Rust, so "builds on an M4 Max" is asserted from source inspection only. Hobbes additionally requires downloading a network from `hobbes-networks` before `cargo build` will work, since `build.rs` reads and transposes `hobbes.nnue` at compile time and the `Makefile` fetches it with `curl` [13]. Viridithas `build.rs` expects a zstd-compressed net at the project root or an `EVALFILE` environment variable [2]. akimbo embeds its net in the repository and needs no fetch [11].

**akimbo is semi-dormant, and this is the weakest point in the recommendation.** Last commit on the default branch 2025-02-07, last release v1.0.0 on 2024-03-26, two and a half years ago [6]. The repository is not archived and the author is actively developing `bullet` instead [6][22]. For this project's purpose that is close to harmless, because akimbo is being read at fixed 2023 and 2024 tags precisely because those tags are small, and a frozen reference cannot rot. But nobody will be fixing a bug in it, and its NNUE code is now a generation behind.

## Open questions

Candidates for tickets.

- **Does Viridithas v20.0.0 build on an M4 Max, and does its NEON path get selected?** The `neon` module in `src/nnue/simd.rs` is gated on `target_feature = "neon"`, which on `aarch64-apple-darwin` should be on by default, but this was read, not compiled [2]. Same question for Hobbes. Resolve by installing a toolchain and building both, which is a half-hour spike and should happen before week 21.
- **Is the "prior to 20.0.0" versus "as of v21" discrepancy in Viridithas's licensing worth clarifying with the author?** The `LICENSE` file at the `v20.0.0` tag is MIT and the relicence commit landed nine days after the tag, so the reading here is that v20.0.0 is MIT [1][2]. If anything substantial gets ported, a one-line issue asking the author to confirm costs nothing and removes the ambiguity permanently.
- **Where does the line sit on tuned constants?** No chess project states a position, no case law was found that addresses parameter tables in engines specifically, and 17 U.S.C. § 102(b) settles the algorithm question but not the compiled-table question [35]. The note's answer is a conservative practice rather than a legal conclusion. If it ever binds, ask someone qualified.
- **Does a permissively licensed SPRT implementation exist?** This survey found only OpenBench's GPL `stats.py` [24]. The underlying mathematics is published, so reimplementation is clean, but a permissive Rust or Python crate would be better than reimplementing from formulas. Not searched exhaustively.
- **What are the Hobbes networks actually licensed under?** `hobbes-networks` has no licence file and no statement [14]. The engine is MIT and the author is responsive. Only matters if anyone wants to distribute a binary of a Hobbes-derived engine, which nobody does, but it is the kind of thing worth a friendly issue.
- **Is Weiss 2.1-dev much stronger than the rated Weiss 2.0?** Five years of unreleased commits, no rating, no published self-play figures found [16][19]. Affects nothing in the reading plan, since the code is recommended for structure rather than strength.
- **Is there a readable hand-crafted evaluation under a permissive licence that this survey missed?** Weiss winning the slot means the one subsystem the project builds first has a GPL-only reference. Altair is the only MIT candidate found and it is weaker as a teaching text [33]. Worth one more look through the sub-3000 hobby-engine field, where classical evaluations are still common.
- **Should the project mirror the specific tags it reads?** akimbo `v0.6.0`, akimbo `v0.5.0` and Viridithas `v20.0.0` are all MIT snapshots whose upstream repositories have moved on, and one of them has relicensed. A vendored copy under `docs/reference/` with licence files intact would make the provenance of any ported code auditable. Cheap, and probably worth doing before week 8.

## Sources

1. Viridithas README, "License" section: "Viridithas is free software, licensed under the GNU Affero General Public License v3.0 (`AGPL-3.0-only`)"; "The neural networks embedded in Viridithas ... are dedicated to the public domain under CC0 1.0"; "Versions prior to 20.0.0 were released under the MIT License and remain available under those terms". Relicence commit `2001ac6`, 2026-07-06, "Relicense to AGPL-3.0-only (as of v21) (#450)", found via `git log --follow -- LICENSE`. https://github.com/cosmobobak/viridithas
2. Viridithas at tag `v20.0.0` (commit `0631113e`, tagged 2026-06-27), cloned 2026-09-16. `LICENSE` reads "MIT License / Copyright (c) 2022-2025 Cosmo Bobak". 25,946 lines of Rust. Verified paths and line counts: `src/search.rs` 2141, `src/nnue/network.rs` 1989, `src/chess/board/mod.rs` 1857, `src/datagen.rs` 1567 (`RANDOM_MOVES_ROOT = 8` at line 52, `generate_on_thread` at line 410), `src/nnue/simd.rs` 1182 (`neon` module at line 770, `dotprod` gating at lines 988-1026), `src/chess/board/movegen.rs` 1141, `src/uci.rs` 1054, `src/search/parameters.rs` 604, `src/transpositiontable.rs` 578, `src/nnue/accumulator.rs` 572, `src/datagen/dataformat.rs` 488, `src/nnue/network/layers.rs` 475, `src/timemgmt.rs` 470, `src/perft.rs` 460, `src/datagen/dataformat/marlinformat.rs` 213, `src/nnue/geometry/neon.rs`. 241 comment lines in `src/search.rs`. https://github.com/cosmobobak/viridithas/tree/v20.0.0
3. Viridithas `master` at commit as of 2026-09-06: `LICENSE` is "GNU AFFERO GENERAL PUBLIC LICENSE Version 3", source files carry `// SPDX-License-Identifier: AGPL-3.0-only`. Reckless `LICENSE` at commit as of 2026-09-09 is also "GNU AFFERO GENERAL PUBLIC LICENSE Version 3, 19 November 2007"; `src/search.rs` 1490, `src/nnue.rs` 453, `src/transposition.rs` 423, `src/history.rs` 266, `src/movepick.rs` 216. No datagen module in the repository. https://github.com/cosmobobak/viridithas and https://github.com/codedeliveryservice/Reckless
4. `cosmobobak/viridithas-networks`: `LICENSE` is CC0 1.0 Universal; README states "All networks are released under CC0". Fetched 2026-09-16. https://github.com/cosmobobak/viridithas-networks
5. akimbo `LICENSE`: "MIT License / Copyright (c) 2023 Jamie Whiting". README "Evaluation" section: "Up to and including version 1.0.0, all data used was self-generated, starting from material values when akimbo still had an HCE"; "akimbo now uses data produced by Leela Chess Zero, however *no official release* will be made with this". README rating table: 0.4.1 2841 Blitz, 0.5.0 3026 on 40/15, 0.6.0 3335, 1.0.0 3583 Blitz. The table's 1.0.0 entry of 2474 on 40/15 is a typo, as already noted in `docs/research/evaluation-path.md`; the CCRL list itself gives 3475 [19]. https://github.com/jw1912/akimbo
6. akimbo tags and releases, read via `git tag` and the GitHub releases API on 2026-09-16: `v0.1.0` through `v1.0.0`; latest release `v1.0.0` published 2024-03-26; default-branch last commit 2025-02-07; repository not archived. https://github.com/jw1912/akimbo/releases
7. akimbo at tag `v0.4.1-pst-only` (2023-08-04). 1,165 Rust lines in 4 files: `src/search.rs` 457, `src/position.rs` 361, `src/util.rs` 217, `src/main.rs` 130. https://github.com/jw1912/akimbo/tree/v0.4.1-pst-only
8. akimbo at tag `v0.5.0` (2023-08-12). Workspace layout with `akimbo/`, `datagen/` and `tuner/` crates, 2,043 Rust lines total. `tuner/src/tuner/mod.rs` 55 (`optimise_k` fits `k` by central finite differences from a start of 0.009; `gd_tune` is Adam with `b1 = 0.9`, `b2 = 0.999`, decayed learning rate and early stop), `tuner/src/core/position.rs` 246, `tuner/src/tuner/data.rs` 102, `tuner/src/core/score.rs` 100, `tuner/src/core/params.rs` 50, `datagen/src/thread.rs` 171, `datagen/src/util.rs` 65, `datagen/src/main.rs` 40. https://github.com/jw1912/akimbo/tree/v0.5.0
9. akimbo at tag `v0.6.0` (2023-09-24). 1,401 Rust lines in 8 files. NNUE inference is inline in `akimbo/src/position.rs` (373 lines): `const HIDDEN: usize = 256` at line 11, `struct Eval([i16; 768 * HIDDEN], [i16; HIDDEN], [i16; 2 * HIDDEN], i16)` at line 14, net embedded from `resources/bob.bin` (394,754 bytes) at line 15, accumulator add and subtract at lines 86-96, `pub fn eval` at lines 155-168 using `i.clamp(0, 255)` (clipped ReLU) and returning `sum * 400 / 16320`. `datagen/src/thread.rs` 168 (`max_nodes: 1_000_000`), `datagen/src/main.rs` 44. https://github.com/jw1912/akimbo/tree/v0.6.0
10. akimbo at tag `v1.0.0` (2024-03-26). 3,233 Rust lines in 13 files, including `src/datagen.rs` 294, which was removed from later `main`. `resources/net.bin` 6,297,664 bytes. https://github.com/jw1912/akimbo/tree/v1.0.0
11. akimbo `main` at commit as of 2025-02-07. 3,024 Rust lines in 13 files: `src/search.rs` 728 (64 comment lines), `src/position.rs` 587 (`pub fn make(&mut self, ...) -> bool` at line 97 doing legality validation, `pub fn movegen` at line 397, `pub fn see` at line 332), `src/tables.rs` 385, `src/uci.rs` 344, `src/network.rs` 210 (`HIDDEN = 1024`, `SCALE = 400`, `QA = 255`, `QB = 64`, `NUM_BUCKETS = 4`, `screlu`, AVX2 `__m256i` intrinsics with a scalar fallback and no NEON path), `src/moves.rs` 163, `src/consts.rs` 135, `src/attacks.rs` 125. `resources/net.bin` 6,297,664 bytes. https://github.com/jw1912/akimbo
12. Hobbes `LICENSE`: "MIT License / Copyright (c) 2025 Dan Kelsey". README: NNUE architecture `((768x16+60144)hm->768)x2->(16x2->32->1)x8`, "trained entirely on data generated from self-play", "initialised from random values", "All of hobbes' networks have been trained using bullet". README strength table gives 3.0 at 3608 on CCRL Rapid, released 2026-07-22; the CCRL list itself gives 3602 [19]. Latest release 3.0, 2026-07-22. Weiss latest release v2.0, 2021-08-04, read via the GitHub releases API. https://github.com/kelseyde/hobbes-chess-engine
13. Hobbes at commit as of 2026-09-12, cloned 2026-09-16. 13,798 Rust lines in 67 files. Verified paths and line counts: `src/search.rs` 1241, `src/board.rs` 745, `src/board/movegen.rs` 520 (`MoveFilter` enum, `gen_moves`, `calc_threats` at line 301, `calc_checkers` at line 324), `src/search/history.rs` 411, `src/board/castling.rs` 395, `src/search/tt.rs` 385, `src/search/parameters.rs` 356, `src/board/legal.rs` 307 (`is_pseudo_legal` at line 15, `is_legal` at line 262), `src/search/movepicker.rs` 270, `src/search/correction.rs` 205, `src/board/magics.rs` 201, `src/search/see.rs` 199, `src/search/lmr.rs` 85, `src/tools/datagen.rs` 76 (random openings only). NNUE inference: `src/evaluation/simd/neon.rs` 220 (`std::arch::aarch64`, `vdupq_n_s16`, `vld1q_u16`), `src/evaluation/simd/avx2.rs` 199, `src/evaluation/simd/avx512.rs` 190, `src/evaluation/forward/vectorised.rs` 187, `src/evaluation/forward/scalar.rs` 114, `src/evaluation/forward/mod.rs` 38 (a `Forward` trait with `Vectorised` and `Scalar` implementations selected by `#[cfg(any(target_feature = "avx2", target_feature = "neon"))]`). `build.rs` reads and transposes `hobbes.nnue` at compile time; `Makefile` target `download-net` fetches it from the `hobbes-networks` releases with `curl`. https://github.com/kelseyde/hobbes-chess-engine
14. `kelseyde/hobbes-networks`: no `LICENSE`, `LICENSE.md`, `LICENCE` or `COPYING` file on `main`; README reads only "Neural networks for my chess engine, Hobbes." GitHub API reports `license: null`. Checked 2026-09-16. https://github.com/kelseyde/hobbes-networks
15. Weiss `COPYING.txt`: "GNU GENERAL PUBLIC LICENSE Version 3, 29 June 2007". Per-file headers state "either version 3 of the License, or (at your option) any later version", so GPL-3.0-or-later. https://github.com/TerjeKir/weiss
16. Weiss at commit `c735b8f3` (`master`, 2026-08-13), cloned 2026-09-16. Verified paths and line counts in `src/`: `search.c` 814 (123 comment lines), `evaluate.c` 587 (`EvalPawns` at line 148, `ProbePawnCache` at 241, `EvalPiece` at 259, `EvalKings` at 338, `EvalPieces` at 377, `EvalPassedPawns` at 391, `EvalThreats` at 449, `ScaleFactor` at 519), `tuner/tuner.c` 593, `board.c` 578, `makemove.c` 359, `uci.c` 287, `movegen.c` 188, `history.h` 168, `movepicker.c` 165, `transposition.c` 144, `tuner/tuner.h` 136, `psqt.c` 95 (tapered `S(mg, eg)` tables, "Black's point of view", comment included), `endgame.c` 80, `evaluate.h` 62 (`PawnEntry`, `PawnCache`, `EvalPosition`). `src/pyrrhic/` is vendored tablebase code. https://github.com/TerjeKir/weiss
17. Weiss `src/pyrrhic/LICENSE`: "The MIT License (MIT) (c) 2015 basil ... Modifications Copyright (c) 2016-2019 by Jon Dart. Modifications Copyright (c) 2020-2020 by Andrew Grant". A permissively licensed subtree inside a GPL repository. https://github.com/TerjeKir/weiss/blob/master/src/pyrrhic/LICENSE
18. Local build verification, 2026-09-16, Mac Studio M4 Max, `arm64`, Apple clang 21.0.0 (clang-2100.3.34.2). `make` in `weiss/src` fails: the `pgo` target errors with `clang: error: Error in reading profile pgo/default.profdata: No such file or directory`. `make basic` succeeds with a single `gcc -std=gnu11 -Wall -Wextra -Wshadow -Werror -O3 -flto=auto -march=native` invocation, producing a 190,008-byte binary that responds to `uci` with `id name Weiss 2.1-dev (c735b8f)`. No `cargo`, `rustc` or `rustup` present, so no Rust engine was compiled.
19. CCRL 40/15 rating list, complete list page. Header as printed: "Ponder off, General book (up to 12 moves), 3-4-5 piece EGTB / Time control: Equivalent to 40 moves in 15 minutes on an Intel i7-4770k / Computed on September 10, 2026 with Bayeselo based on 2'443'716 games", 4,720 programs. Single-CPU 64-bit entries used here: Stockfish 18 3629 ±8 (3379 games), Reckless 0.9.0 3623 ±9 (2781), Viridithas 20.0.0 3613 ±10 (2245), Stormphrax 8.0.0 3609 ±9 (2467), Alexandria 9.0.0 3608 ±9 (2461), Obsidian 16.0 3606 ±7 (3975), Berserk 14 3604 ±10 (2016), Hobbes 3.0 3602 ±12 (1229), Motor 0.9.0 3572 ±9 (2313), Ethereal 14.25 3566 ±6 (6675), Velvet 8.1.1 3532 ±9 (2248), Koivisto 9.0 3524 ±6 (6724), akimbo 1.0.0 3475 ±9 (2509), Altair 7.0.0 3475 ±11 (1666), Carp 3.0.0 3455 ±10 (2165), Weiss 2.0 3264 ±10 (2128). Stockfish 19 appears at 3637 ±32 on only 160 games, below CCRL's stated 200-game bold threshold, so Stockfish 18 is quoted instead. Rustic and Pleco are absent from this list. The site is behind a Cloudflare JavaScript challenge that blocks direct fetching; the page text was retrieved through the `r.jina.ai` read-only reader proxy on 2026-09-16 and every figure above was transcribed from it, with the Viridithas, Hobbes, akimbo, Weiss, Reckless and Stockfish rows independently re-checked against the cached text. https://www.computerchess.org.uk/ccrl/4040/
20. CCRL Blitz (2+1) rating list, complete list page: "computed on September 12, 2026 with Bayeselo based on 2'115'612 games", 2,955 programs. Rustic Alpha 3.0.0 64-bit 1792 ±16 (1713 games), Rustic Alpha 2 1720 ±20, Rustic Alpha 1 1550 ±21. Pleco absent. Retrieved as in [19]. https://www.computerchess.org.uk/ccrl/404/
21. CCRL 40/15 testing-conditions page: "Hash size: Should be set to the same value of either 256 or 512 MB for all engines in a match or tourney"; "Pondering: OFF"; "Endgame tablebases: 4, 5 or 6 piece tablebases"; "Opening book: Any generic ... limited to 12 moves per side maximum"; "Book learning: Off for all engines"; "We use repeating time control". Note a discrepancy with the list page itself, which says "3-4-5 piece EGTB" where this page says "4, 5 or 6 piece tablebases"; both are quoted as printed. https://www.computerchess.org.uk/ccrl/4040/about.html
22. `bullet` (jw1912), `LICENSE`: "MIT License / Copyright (c) 2023 Jamie Whiting". At commit as of 2026-09-15: `examples/simple.rs` 211, `examples/progression/1_simple.rs` 63, `2_output_buckets.rs` 67, `3_input_buckets.rs` 103, `4_multi_layer.rs` 113. No tagged releases. https://github.com/jw1912/bullet
23. `bulletformat` (jw1912), `LICENSE`: "MIT License / Copyright (c) 2023 Jamie Whiting". At commit as of 2026-09-14: `src/chess.rs` 248, `src/chess/marlin.rs` 141, `src/chess/cudad.rs` 141, `src/loader.rs` 109, `src/convert.rs` 101, `src/lib.rs` 44. https://github.com/jw1912/bulletformat
24. OpenBench (AndyGrant), `LICENSE`: "GNU GENERAL PUBLIC LICENSE Version 3"; per-file headers say "either version 3 of the License, or (at your option) any later version" and credit "a chess engine testing framework authored by Andrew Grant". At commit as of 2026-09-08: 9,409 Python lines. `OpenBench/Client/worker.py` 1439, `OpenBench/OpenBench/views.py` 1143, `utils.py` 640, `workloads/verify_workload.py` 473, `models.py` 365, `workloads/get_workload.py` 339, `workloads/create_workload.py` 323, `templatetags/mytags.py` 307, `config.py` 219, `Client/pgn_util.py` 189, `stats.py` 173 (`TrinomialSPRT` at line 33, `PentanomialSPRT` at line 52, `Elo` at 74, `bayeselo_to_proba` at 91, `MLE_tvalue` at 139, `logistic_elo` at 161). No tagged releases. https://github.com/AndyGrant/OpenBench
25. `fastchess` (Disservin). GitHub API reports SPDX `MIT`, last push 2026-09-12, not archived. Checked 2026-09-16. https://github.com/Disservin/fastchess
26. Stockfish `Copying.txt`: "GNU GENERAL PUBLIC LICENSE Version 3, 29 June 2007". At commit as of 2026-09-13: 25,340 lines across 72 C++ files. https://github.com/official-stockfish/Stockfish
27. Pleco (sfleischman105), `LICENSE`: "GNU GENERAL PUBLIC LICENSE Version 3". README: "Pleco is a chess Engine & Library derived from Stockfish, written entirely in Rust"; "For the engine, the majority of the code is a direct port of Stockfish's C++ code". 22,102 lines in 83 files across `pleco` and `pleco_engine` crates. The twelve most recent commits all date from 2026-02-22 and are Copilot-authored soundness, bounds-check and formatting patches (pull requests #167 through #170), not engine work. https://github.com/sfleischman105/Pleco
28. Stormphrax (Ciekce), `LICENSE`: "GNU GENERAL PUBLIC LICENSE Version 3". At commit as of 2026-08-29: 44,501 lines in 147 files by raw count, of which `src/` is 21,993; the remainder is vendored `3rdparty/fmt` (`format.h` 4244, `base.h` 2989, `chrono.h` 2330, `format-inl.h` 1948), `3rdparty/zstd` (`zstd.h` 3106) and `3rdparty/pyrrhic` (`tbprobe.cpp` 2105). `src/search.cpp` 1883. README notes a modified `incbin` bundled "under the Unlicense". https://github.com/Ciekce/Stormphrax
29. Ethereal (AndyGrant), `LICENSE`: "GNU GENERAL PUBLIC LICENSE Version 3". Last commit on the default branch 2024-06-11. 13,595 lines in 58 files. https://github.com/AndyGrant/Ethereal
30. Motor (martinnovaak). No `LICENSE`, `LICENSE.md`, `LICENCE`, `COPYING` or `UNLICENSE` file at the repository root, and no case-insensitive match for "licence", "license", "GPL" or "MIT" in `README.md` or `Cargo.toml`/build files. C++, not Rust: 31 `.cpp`/`.hpp` files and zero `.rs` files, 5,061 lines. Last commit 2025-06-02. Checked 2026-09-16. https://github.com/martinnovaak/motor
31. Rustic (mvanthoor). The GitHub repository contains a single file, `readme.md`, stating "Rustic has moved to Codeberg" and giving the author's reasons. The live repository at `codeberg.org/mvanthoor/rustic` has `LICENSE` = "GNU GENERAL PUBLIC LICENSE Version 3", last commit 2026-06-02, 11,473 lines in 126 files across `engine`, `rustic`, `texel`, `wizardry`, `debugger` and `epdtest` crates. https://github.com/mvanthoor/rustic and https://codeberg.org/mvanthoor/rustic
32. Carp (dede1751), `LICENSE`: "GNU GENERAL PUBLIC LICENSE Version 3". Last commit 2025-01-18. 12,062 lines in 45 files; `engine/src/search.rs` 699, `chess/src/board.rs` 558, `chess/src/movegen/gen.rs` 385, `engine/src/tt.rs` 339, `engine/src/move_picker.rs` 331, `chess/src/nnue/mod.rs` 316, `tools/src/datagen.rs` 278. https://github.com/dede1751/carp
33. Altair (Alex2262), `LICENSE`: "MIT License / Copyright (c) 2023 Alex2262". Last commit 2026-04-18. C++, 7,989 lines in 42 files: `src/search.cpp` 1611, `src/position.cpp` 883, `src/evaluation_classic.cpp` 565, `src/datagen.cpp` 450, `src/uci.cpp` 319, `src/move_ordering.cpp` 223, `src/nnue.h` 174, `src/nnue.cpp` 142, `src/simd.h` 161, `src/see.cpp` 113. No tuner in the repository. Ships `src/ceres-net.bin` (7,899,200 bytes) under the repository's MIT licence. https://github.com/Alex2262/AltairChessEngine
34. Licence files read directly on 2026-09-16, all "GNU GENERAL PUBLIC LICENSE Version 3, 29 June 2007": Berserk `LICENSE` (last commit 2026-09-06), Alexandria `LICENSE.md` (2026-09-08), Obsidian `LICENSE` (2025-09-27), Koivisto `LICENSE.md` (2023-02-08), Velvet `LICENSE` (2025-07-03). https://github.com/jhonnold/berserk , https://github.com/PGG106/Alexandria , https://github.com/gab8192/Obsidian , https://github.com/Luecx/Koivisto , https://github.com/mhonert/velvet-chess
35. 17 U.S.C. § 102(b), quoted in full: "In no case does copyright protection for an original work of authorship extend to any idea, procedure, process, system, method of operation, concept, principle, or discovery, regardless of the form in which it is described, explained, illustrated, or embodied in such work." This establishes that algorithms and methods are unprotected; it does not address compilations of tuned parameters, and no source consulted for this note does. https://www.law.cornell.edu/uscode/text/17/102
