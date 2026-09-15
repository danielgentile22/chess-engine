# Engine language: Rust or C++

Researched 2026-09-15. Target: bitboard alpha-beta engine, NNUE evaluation, roughly 3000 Elo on the Computer Chess Rating Lists (CCRL), one core, Apple Silicon.

## Answer

Keep Rust. The provisional pick is correct.

Rust costs you nothing measurable in engine strength. Reckless 0.9.0 is written in Rust and sits at rank 2 on the CCRL 40/15 list, 4 Elo behind Stockfish 18 and ahead of Torch, Obsidian, Alexandria and Stormphrax [2][3]. On CCRL Blitz, Reckless is rank 5 and Viridithas 20.0.0 is rank 11, both on a single core, against C and C++ engines running on eight [1]. Your 3000 Elo target sits around rank 246 on that list, roughly 750 Elo below where the Rust engines already are.

Rust costs you almost nothing in speed, and the one gap that used to exist closed last month. NNUE inference needs hand-written vector code in both languages, because the activation functions modern networks use do not autovectorise in any language [43]. On ARM the one instruction you want used to be missing from stable Rust, so Viridithas and Reckless both wrote the Advanced SIMD (Single Instruction Multiple Data) 8-bit dot product as inline assembly [9][6]. That intrinsic, `vdotq_s32`, became stable on 2026-08-20 in Rust 1.98.0 [20][21]. Neither engine has migrated yet. You can just use it.

The `unsafe` cost is also smaller than the engines make it look. Safe `#[target_feature]` functions stabilised in Rust 1.86.0 and safe architecture intrinsics in 1.87.0, so arithmetic intrinsics no longer need `unsafe` at all: only loads and stores do [44][45]. Every engine surveyed predates this and wraps everything in `unsafe` anyway. Write it the new way and your `unsafe` surface is a handful of loads.

Everything else is a tie or a Rust advantage. Profile-guided optimisation works the same way through the same LLVM machinery in both toolchains, and the Rust engines ship it [4][8][11][15]. Link-time optimisation, `codegen-units = 1` and `panic = "abort"` are one-line settings in `Cargo.toml` [5][17]. Testing tooling is language-agnostic by design [26]. Every strong Rust engine builds on stable Rust: not one `#![feature(...)]` attribute among them.

On Apple Silicon specifically, Rust is ahead. `-C target-cpu=native` resolves correctly through a sysctl [16][37], whereas Clang's `-march=native` is accepted and silently downgrades your build, and Homebrew GCC's falls back to generic ARMv8-A [36]. Two of the C and C++ engines surveyed get this wrong today.

## Rating evidence

CCRL runs two public lists. Blitz is equivalent to 2 minutes plus 1 second per move on an Intel i7-4770K [1]. 40/15 is 40 moves in 15 minutes on the same reference machine [2]. Entries marked `4CPU` or `8CPU` were tested on that many cores; entries with no such marking are single-core results. That distinction matters here: the top Rust engines post their Blitz numbers on one core.

### CCRL 40/15, computed 2026-09-10 [2]

| Rank | Engine | Language | Cores | Rating |
|---|---|---|---|---|
| 1 | Stockfish 18 | C++ | 4 | 3649 |
| 2 | **Reckless 0.9.0** | **Rust** | 4 | **3645** |
| 3 | PlentyChess 7.0.0 | C++ | 4 | 3644 |
| 5 | Torch v4d | C++ | 4 | 3639 |
| 6-7 | Obsidian 16.0 | C++ | 4 | 3637 |
| 8 | Alexandria 9.0.0 | C | 4 | 3635 |
| 9 | Stormphrax 8.0.0 | C++ | 4 | 3634 |
| 10-12 | **Viridithas 19.0.1** | **Rust** | 4 | **3633** |
| 15-16 | Berserk 14 | C | 4 | 3628 |
| 33 | Ethereal 14.25 | C | 4 | 3600 |
| 45 | **Velvet 8.1.1** | **Rust** | 4 | **3578** |
| 56 | Koivisto 9.0 | C++ | 4 | 3561 |
| 77 | **Carp 3.0.0** | **Rust** | 4 | 3503 |
| 185 | **Princhess 0.22.0** | **Rust** | 1 | 3254 |

### CCRL Blitz, computed 2026-09-12 [1]

| Rank | Engine | Language | Cores | Rating |
|---|---|---|---|---|
| 1 | Stockfish 17.1 | C++ | 8 | 3785 |
| 3 | PlentyChess 8.0.0 | C++ | 1 | 3774 |
| 4 | Obsidian 16.0 | C++ | 8 | 3771 |
| 5 | **Reckless 0.9.0** | **Rust** | 1 | **3766** |
| 7 | Berserk 20250606 | C | 8 | 3759 |
| 9 | Alexandria 8.0.0 | C | 8 | 3755 |
| 11 | **Viridithas 20.0.0** | **Rust** | 1 | **3747** |
| 14 | Stormphrax 8.0.0 | C++ | 1 | 3744 |
| 25 | Ethereal 14.25 | C | 8 | 3723 |
| 33-34 | **Velvet 8.1.0** | **Rust** | 8 | **3702** |
| 50 | Koivisto 9.0 | C++ | 8 | 3682 |
| 65 | **Black Marlin 9.0** | **Rust** | 8 | 3622 |
| 68 | **akimbo 1.0.0** | **Rust** | 8 | 3614 |
| 96 | **Carp 3.0.1** | **Rust** | 1 | 3525 |
| 151 | **Princhess 0.22.0** | **Rust** | 1 | 3357 |

Three things follow.

Rust engines are far above 3000 on one core. Reckless at 3766 and Viridithas at 3747 are both single-core Blitz results [1]. Your target of roughly 3000 corresponds to about rank 246 on the current Blitz list (Zappa Mexico II at 3007, Rengar 2.1.1 at 2997), which is well inside territory that several Rust engines passed years ago.

The strongest Rust engine is level with Stockfish on 40/15. Four Elo with a stated error bar of plus or minus 12 is not a gap you can attribute to anything, let alone to a language [2].

The weak Rust engines are weak for engine reasons, not language reasons. Carp at 3503 has no ARM code path and no vector code beyond a single architecture intrinsic, though it does ship profile-guided optimisation [12]. Princhess uses Monte Carlo tree search rather than alpha-beta, so it is not comparable at all [32]. Reckless, Viridithas and Velvet all have hand-written vector code for three instruction sets each. The spread inside Rust is 400 Elo, far wider than any spread between languages.

Language attributions above come from each repository's own source tree and README. Reckless states "Rust 1.88.0 or a later version" as its build requirement [3].

## Throughput

What can be concluded: nothing useful from cross-engine nodes-per-second comparisons.

Nodes per second measures how cheaply an engine visits a node, which is dominated by how much work the engine chooses to do per node. An engine with a large NNUE network, expensive move ordering and staged move generation will report a lower nodes-per-second figure than a simple engine and still be 500 Elo stronger. Comparing Viridithas nodes per second against Ethereal nodes per second tells you about their evaluation sizes, not about Rust against C.

CCRL does not publish nodes per second, so there is no same-machine apples-to-apples figure across languages to cite. I found none in the engines' own documentation either.

What can be concluded: within a single codebase, changes are measurable, and the Rust engine authors do measure them. Cosmo Bobak, the Viridithas author, published a post on NNUE performance work reporting speed relative to his own master branch: a naive rewrite came out at 95.70 percent of master, and an explicit 16-register unrolled version at 103.33 percent [30]. He also notes the relevant compiler caveat directly: "in NNUE inference we're typically working with `unsafe` calls directly to SIMD intrinsics where the compiler may perhaps be less able to slice through to see our intent" [30]. That is a developer's own blog post, not a controlled benchmark, but it is the right shape of evidence: same engine, same machine, one variable.

The practical reading is that both languages compile through LLVM, both engines hand-write the vector code that actually matters, and the resulting machine code for the NNUE inner loops is close to identical. The Elo table is the real throughput evidence.

## SIMD for NNUE

Neither language autovectorises NNUE inference well enough to matter. Every strong engine in either language writes the vector code by hand. This is the single most important fact in this section, because it means the comparison is not "Rust autovectorisation versus C++ autovectorisation". It is "Rust intrinsics versus C++ intrinsics", and those map one to one onto the same machine instructions.

The mechanical reason is the activation function, not the language. The Chess Programming Wiki's NNUE page says clipped rectified linear unit activation is "easy to implement and can be auto-vectorized", while squared clipped rectified linear unit, the one modern networks actually use, "cannot be auto-vectorized, and requires hand-written SIMD code to optimize" [43]. A C++ engine faces exactly the same wall.

Measured from the checked-out sources on 2026-09-15:

| Engine | Language | Vector backends present | Portable SIMD | Nightly required |
|---|---|---|---|---|
| Viridithas | Rust | AVX-512, AVX2, NEON, scalar fallback [9] | no | no |
| Reckless | Rust | AVX-512, AVX2, NEON, WebAssembly, scalar [6] | no | no |
| Velvet | Rust | AVX-512, AVX2, NEON, scalar [10] | no | no |
| Carp | Rust | essentially none | no | no |
| Stockfish | C++ | AVX-512, AVX2, SSSE3, NEON, NEON dot product, LoongArch [13][14] | not applicable | not applicable |

Rust's own portable SIMD library, `std::simd`, is still marked "nightly-only experimental API" as of September 2026 [22][23]. No chess engine surveyed uses it. Zero occurrences across Viridithas, Reckless, Velvet and Carp. This is a non-issue precisely because nobody wanted it: the engines use architecture-specific intrinsics, the same as C++ engines do.

None of the four engines contains a single `#![feature(...)]` attribute, so all of them compile on stable Rust. Viridithas declares `rust-version = "1.93.0"` in its `Cargo.toml`, Reckless's README asks for 1.88.0 or later [3][7]. The only nightly in any of these repositories is Reckless's optional WebAssembly target, which needs `-Z build-std`, and that is irrelevant to a native engine [4].

Rust's architecture intrinsics live in `std::arch` and have been stable on aarch64 since Rust 1.59.0 in 2022 [24]. **They are no longer all `unsafe`.** Two stabilisations changed this:

- Rust 1.86.0, 2025-04-03, stabilised `target_feature_11`: safe functions can carry `#[target_feature(enable = "...")]`, and calling one from another function with the same features needs no `unsafe` [44].
- Rust 1.87.0, 2025-05-15, made most intrinsics safe. The release notes say "Most `std::arch` intrinsics that are unsafe only due to requiring target features to be enabled are now callable in safe code that has those features enabled" [45]. The qualifier is pointers: intrinsics taking raw pointers stay `unsafe`.

I verified this against the current documentation. `pub fn vaddq_s32(...)` is safe. `pub fn vdotq_s32(...)` is safe. `pub unsafe fn vld1q_s16(ptr: *const i16)` is still `unsafe`, because it takes a pointer [20][24].

So the current idiomatic shape on stable is: one safe `#[target_feature(enable = "neon,dotprod")]` function holding the hot loop, arithmetic intrinsics called with no `unsafe` at all, and `unsafe` only around `vld1q_*` and `vst1q_*`. **No engine surveyed does this.** They all predate the change, wrap everything in `unsafe`, and Viridithas even opens `simd.rs` with `#![allow(clippy::undocumented_unsafe_blocks)]` to silence the resulting lint noise [9]. Treat their `unsafe` density as a historical artefact, not as the cost of writing this today.

### ARM and NEON

This is the part that matters for a Mac Studio, and it is where the one real Rust gap lives.

Stockfish has a first-class NEON path. `src/nnue/layers/affine_transform.h` branches on `USE_NEON` and `USE_NEON_DOTPROD` throughout [14]. The Makefile's `ARCH=apple-silicon` target sets `neon = yes` and `dotprod = yes`, and the dot product path compiles with `-march=armv8.2-a+dotprod -DUSE_NEON_DOTPROD` [13]. So Stockfish on an M-series Mac uses the ARM 8-bit signed dot product instruction `sdot` for NNUE inference.

Viridithas has a NEON path too. `src/nnue/simd.rs` contains a `neon` module built on `use std::arch::aarch64::*`, wrapping `int8x16_t`, `int16x8_t`, `int32x4_t`, `int64x2_t` and `float32x4_t` [9]. It is selected by `#[cfg(target_feature = "neon")]`, which is always true on aarch64 because NEON is part of the AArch64 baseline.

**The gap that used to exist, and closed on 2026-08-20.** `vdotq_s32`, the intrinsic for the `sdot` instruction, was nightly-only for years under the unstable feature `stdarch_neon_dotprod`. Its tracking issue, rust-lang/rust#117224, closed as completed on 2026-06-03, and the intrinsic shipped stable in Rust 1.98.0 [21]. I verified the current documentation directly: the signature reads `pub fn vdotq_s32(a: int32x4_t, b: int8x16_t, c: int8x16_t) -> int32x4_t`, marked stable since 1.98.0, with no nightly banner and no `unsafe` [20].

Both Rust engines wrote inline assembly to work around it, and neither has migrated. Reckless's version carries a dated rationale, from a commit of 2026-04-09, four months before stabilisation: "Those intrinsics are not stable so we use `std::arch::asm!`, the output looks tight and good. I left the intrinsics equivalent as a comment for future use. Overall speedup seems to be about 12%" [6]. Viridithas's version:

```rust
#[cfg(target_feature = "dotprod")]
unsafe fn vdotq_s32(a: int32x4_t, b: int8x16_t, c: int8x16_t) -> int32x4_t {
    let r: int32x4_t;
    unsafe {
        std::arch::asm!(
            "sdot {a:v}.4s, {b:v}.16b, {c:v}.16b",
            a = inout(vreg) a => r,
            b = in(vreg) b,
            c = in(vreg) c,
            options(pure, nostack, nomem, preserves_flags)
        );
    }
    r
}
```

Source: `src/nnue/simd.rs`, with an in-file credit to the contributor who wrote it [9]. Inline assembly, `asm!`, is itself stable in Rust, so this stays on the stable toolchain.

**Cost assessment: as of Rust 1.98.0, zero.** Call `vdotq_s32` directly. The inline assembly above is now historical, and the fact that both engines still ship it tells you how recent the change is, not that the problem persists. If you are on an older toolchain, the workaround is fifteen lines and publicly available. And it only applies once you quantise to 8-bit integers and want `sdot` at all. A 16-bit integer NNUE, which is what you will have for a long while, needs only `vaddq_s16`, `vsubq_s16`, `vmaxq_s16`, `vminq_s16`, `vqdmulhq_s16` and friends, all stable since Rust 1.59.0 in 2022 [24]. Reckless's `src/nnue/simd/neon.rs` is built entirely from those [6].

**What is still nightly on ARM.** Two things, neither of which you need. The 8-bit integer matrix multiply intrinsics (i8mm, `vmmlaq_s32`) are gated behind `stdarch_neon_i8mm`, and the Scalable Matrix Extension intrinsics behind `stdarch_aarch64_sve`. Stockfish has no path for either, so this is not a Rust deficit relative to C++ [13].

**Coverage across Rust engines is uneven, and this is worth knowing when you pick a reference implementation.** Full NEON: Viridithas (including its threat-feature geometry), Velvet, and Reckless (except its threat accumulator, which falls back to scalar on ARM). No NEON at all: akimbo, Carp, Princhess, Monty. Black Marlin has a NEON dot product that is commented out in the source, falling through to a scalar loop. Stockfish, by contrast, has NEON with a dot product fast path across the entire inference stack and ships an `apple-silicon` Makefile target that enables it by default [13][14]. Read Viridithas and Reckless for ARM reference code. Do not read the others.

The other NEON caveat is arithmetic, not language. Viridithas notes in its source that "NEON only supports i8-i8 dotprod", and that "on NEON, the instruction used for mulhi doubles the results", so one input must be pre-shifted [9]. Reckless carries the same note in `neon.rs`: `vqdmulhq_s16` "doubles the result, so one of the inputs must be preshifted" [6]. A C++ engine on ARM hits the identical problem. Stockfish handles it in its own NEON branches [14].

## Profile-guided optimisation and link-time optimisation

Both toolchains run profile-guided optimisation through LLVM's instrumentation. rustc exposes it as `-C profile-generate` and `-C profile-use`, documented officially, and it uses the same `llvm-profdata` tool as Clang [15]. Clang exposes it as `-fprofile-generate` and `-fprofile-use` [19]. There is no capability difference.

What the engines actually ship:

Stockfish's default recommended build is the profile-guided one. The Makefile's `profile-build` target is described as "standard build with profile-guided optimization", the plain `build` target is described as "skip profile-guided optimization", it runs `./stockfish bench` to collect the profile, and the macOS universal binary target invokes `make profile-build ARCH=apple-silicon` [13]. Ethereal, Berserk and Obsidian all ship it too. Stormphrax, at rank 9 on the 40/15 list, ships none at all [40], which is a useful reminder that this is an optimisation, not a requirement.

Viridithas does it by hand in its Makefile, and has a dedicated Apple Silicon target [8]:

```
aarch64-apple: tmp-dir
	cargo rustc -r --target=aarch64-apple-darwin --features final-release -- -C target-feature=+crt-static -C profile-generate=$(TMPDIR) --emit link=...
	./... bench
	llvm-profdata merge -o $(TMPDIR)/merged.profdata $(TMPDIR)/*.profraw
	cargo rustc -r --target=aarch64-apple-darwin --features final-release -- -C target-feature=+crt-static -C profile-use=$(TMPDIR)/merged.profdata --emit link=...
```

Reckless uses the `cargo-pgo` wrapper, which reduces the same sequence to three commands [4][18]:

```
pgo:
	cargo pgo instrument
	cargo pgo run -- bench
	cargo pgo optimize
```

Velvet also uses `cargo-pgo`, in its Makefile and in a GitHub Actions release workflow [11]. Its only currently active release job is the `aarch64-apple-darwin` one, so Velvet's shipped macOS binary is a profile-guided Apple Silicon build.

Carp hand-rolls it in its `makefile`, trained on `bench 16`, and also applies a second profile-guided pass to its data generation binary [42].

Princhess ships none [32]. It is also the weakest and the only one not using alpha-beta.

Two caveats worth carrying forward. Viridithas's GitHub Actions release workflow does not call the Makefile: it runs a plain `cargo build --release`, so the published release binaries are not profile-guided even though the Makefile supports it [7]. And several Rust engines pass `-C target-feature=+crt-static` on Apple targets, which the Rust reference says is simply ignored there, since Apple targets cannot switch C runtime linkage [33]. Harmless, but do not copy it.

Link-time optimisation in Rust is Cargo profile configuration, documented in the Cargo book [17]. Reckless's `Cargo.toml` sets `lto = "fat"`, `panic = "abort"` and `codegen-units = 1` [5]. Viridithas sets `lto = true`, `codegen-units = 1`, `panic = "abort"` and `strip = true`. Velvet and Carp both set `lto` and `codegen-units = 1`. This is strictly less friction than the C++ equivalent, which is compiler and linker flags spread across a Makefile.

Note also that Cargo's default `lto = false` is not "no link-time optimisation". It means thin-local optimisation across the crate's own codegen units. `"off"` is the real off switch. With `codegen-units = 1` and no `lto` key at all, you get none [17].

**How much is it worth?** I found no published Elo or percentage figure for what profile-guided optimisation buys a chess engine, in either language. Stockfish's Makefile calls it the standard build but quotes no number [13].

What is citable is the conversion. The Stockfish project's own data page gives, for small speedups under about 5 percent, a linear estimate of Elo from a speedup percentage x: `Elo_stc(x) = 2.10 x` at short time control and `Elo_ltc(x) = 1.43 x` at long [31]. So a 3 percent speedup is worth roughly 6 Elo at short time control. Use that to decide whether a given optimisation is worth your evening. Do not quote a specific Elo figure for profile-guided optimisation itself, because nobody has published one.

For scale on the general technique, the Rust compiler team measured profile-guided optimisation on rustc itself at roughly 1 percent instruction count, and on rustdoc, Clippy and Cargo at roughly 4 to 5 percent wall time [34][35]. Those are compiler workloads, not chess search. Treat them as an order of magnitude, not a prediction.

## Ergonomics and safety for a hand-written engine

**Bounds checks and `unsafe` surface.** This is the real Rust tax, and it is modest. Counting lines that mention `unsafe` in the checked-out sources:

| Engine | Rust lines | Lines mentioning `unsafe` | `get_unchecked` calls | Share |
|---|---|---|---|---|
| Viridithas | 26,647 | 503 | 41 | 1.9% |
| Reckless | 11,402 | 248 | 45 | 2.2% |
| Velvet | 16,228 | 84 | not counted | 0.5% |
| Carp | 7,501 | 34 | not counted | 0.5% |

Read these as an upper bound, not a target. Most of that count is the SIMD wrapper layer, where every intrinsic call was `unsafe` when these engines were written. Since Rust 1.87.0 most of it would not need to be (see the SIMD section above), so a new engine written today lands well below these numbers. The `get_unchecked` figures are the honest measure of how much indexing the authors felt they had to remove bounds checks from: 41 and 45 call sites in engines of that size. That is a handful of hot loops, not a pervasive rewrite. Note also that in C++ every one of those accesses is unchecked with no annotation at all, so the Rust number is a count of places where you consciously accepted the same risk C++ gives you by default.

Practical approach: index normally everywhere, and only reach for `get_unchecked` in a profiled hot loop. Better still, iterate rather than index, or index a fixed-size array with a type that cannot be out of range, which removes the check without `unsafe`.

**Integer overflow.** Rust panics on overflow in debug builds and wraps in release builds, per the book: "when you're compiling in release mode with the `--release` flag, Rust does not include checks for integer overflow that cause panics. Instead, if overflow occurs, Rust performs two's complement wrapping" [25]. For an engine this is a gift. Your debug and test builds catch a whole class of evaluation and search bugs that in C++ would be silent undefined behaviour on signed types. Your release build has zero overhead. Use `wrapping_mul` explicitly where you want wrapping, such as Zobrist hashing, so debug builds do not panic on intentional wraparound.

**Shared mutable state: transposition tables and Lazy Symmetric Multiprocessing.** This is the one place the borrow checker genuinely fights an engine, because Lazy SMP deliberately shares one transposition table across threads with racy unsynchronised access. Rust will not let you write that with a plain `&mut`. Both solutions are in the surveyed sources, and both are short:

- Atomics with relaxed ordering. Viridithas stores each entry as `memory: [AtomicU64; 4]` and reads and writes every field with `Ordering::Relaxed` [9]. Velvet does the same with `Vec<[AtomicU64; SLOTS_PER_SEGMENT]>` [10]. Relaxed atomic loads and stores compile to ordinary loads and stores on both x86-64 and AArch64, so this costs nothing at runtime.
- A raw pointer plus an explicit opt-out. Reckless holds `ptr: AtomicPtr<Cluster>` and writes one line, `unsafe impl Sync for TranspositionTable {}` [7].

Either is a few dozen lines, written once. Both stay on stable. Both engines also carry an architecture-specific prefetch: Viridithas gates one on `#[cfg(target_arch = "aarch64")]` in `src/transpositiontable.rs` [9], and Velvet calls `core::arch::aarch64::__prefetch` [10].

Plan to hit this. It is the first thing that will feel like the language is in your way, and it is a solved problem with three public reference implementations you can read.

**Compile-time lookup tables.** Rust `const fn` runs at compile time on stable, so magic bitboard tables, attack tables and zobrist seeds can be computed in the binary with no build script and no generated source file. The `shakmaty` crate does exactly this: its `magics` feature uses "large attack tables (currently 694 KiB precomputed at compile time)" [29]. C++ `constexpr` gets you the same thing. Tie, with a slight Rust edge in that `const fn` has fewer surprising restrictions than older `constexpr`.

**C++ side.** The real costs are undefined behaviour and build friction. Signed integer overflow, out-of-bounds reads, uninitialised reads and data races are all undefined behaviour, and in a search that runs billions of nodes they surface as non-reproducible wrong moves. The mitigation is good: Clang's AddressSanitizer, UndefinedBehaviorSanitizer and ThreadSanitizer are mature and well documented [19]. But they are opt-in, they slow the build down, and you have to remember to run them. Rust gives you a subset of the same guarantees with no build step. Build friction is the other cost: a Makefile with architecture detection, per-architecture flag sets and compiler version branching, versus `cargo build --release`. Stockfish's Makefile is 1,536 lines [13]. Reckless's is 70, and Viridithas's is 81.

**Data races.** Rust's `Send` and `Sync` traits make the Lazy SMP data sharing question explicit at compile time. In C++ the same design compiles silently and you find out later. Both give you the same machine code once you write the `unsafe impl Sync`, but Rust makes you name the assumption.

## Ecosystem

**Testing tooling is language-agnostic. Confirmed.** Everything talks to the engine over the Universal Chess Interface (UCI), a plain-text protocol on standard input and output, so the harness cannot tell what the engine was written in. cutechess-cli and fastchess both work this way, and fastchess is what OpenBench itself runs matches with [39].

OpenBench, the distributed testing framework most hobby engine authors use, states it directly in its wiki: "Engines written in languages other than C/C++ are still built by executing a make command, although the makefile might be as simple as executing `cargo`, and passing along an option or two" [26]. Its engine configuration file has a `compilers` field that lists whatever the engine needs [27]. Viridithas ships an `openbench` Makefile target for precisely this [8].

The OpenBench requirements worth designing for from day one, because retrofitting them is annoying [26]:

- The engine must support the `Hash` and `Threads` UCI options, even if `Threads` is fixed at 1.
- The engine must build by running `make`, and must honour `EXE=` to set the output binary name.
- Running `./binary bench` must print a final node count and a nodes-per-second count and then exit.
- The bench node count must be deterministic across runs and machines. Seed your Zobrist keys.
- If you embed a neural network, the build must honour `EVALFILE=/path/to/network`.

That list is also a good spec for your own build regardless of whether you ever use OpenBench.

**UCI libraries: you do not need one.** UCI is a line-based text protocol. Every engine surveyed parses it by hand in a few hundred lines. Adding a dependency here buys nothing and costs you control over the non-standard extensions every engine ends up adding.

**Rust chess crates.** Two are worth knowing about even though you plan to write move generation by hand:

- `cozy-chess` implements "fixed shift fancy black magic bitboards" and optionally PEXT bitboards via the BMI2 intrinsic, which is x86-only and irrelevant on your hardware [28].
- `shakmaty` supports standard chess and all Lichess variants, and claims "move generation performance in the ballpark of the world's best chess engines" [29].

Use them as reference implementations and as a perft oracle. Write your own move generator against their move counts. Do not depend on either in the engine: move generation is the part of the project you said you want to learn, and none of the strong Rust engines depends on an external move generator.

One number from `shakmaty` is worth internalising: its `magics` feature gains "about 20 percent perft speed on x86 and about 5 percent on ARM" [29]. Large lookup tables pay off less on Apple Silicon than on x86. That is a hardware fact, not a language fact, and it will shape several of your later optimisation choices.

## Apple Silicon notes

**Do not use `-march=native`.** Verified locally on 2026-09-15 with Apple clang 21.0.0 targeting `arm64-apple-darwin27.0.0`. Comparing predefined macros against a no-flag baseline, `-march=native` removed features:

```
base vs -march=native:
< #define __ARM_FEATURE_AES 1
< #define __ARM_FEATURE_CRYPTO 1
< #define __ARM_FEATURE_FP16_FML 1
< #define __ARM_FEATURE_FP16_VECTOR_ARITHMETIC 1
< #define __ARM_FEATURE_SHA2 1
< #define __ARM_FEATURE_SHA3 1
```

The flag is accepted without warning and quietly downgrades you toward generic ARMv8. On x86 the same flag is the standard advice, so this is an easy mistake to carry over.

**Use `-mcpu=apple-m4`.** Same test, same baseline, comparing `-mcpu=apple-m4`:

```
base vs -mcpu=apple-m4:
> #define __ARM_ARCH_8_6__ 1
> #define __ARM_ARCH_8_7__ 1
> #define __ARM_FEATURE_BF16 1
> #define __ARM_FEATURE_MATMUL_INT8 1
> #define __ARM_FEATURE_SME 1
> #define __ARM_FEATURE_SME2 1
```

`__ARM_FEATURE_MATMUL_INT8` is the one to notice. That is the i8mm extension, 8-bit integer matrix multiply, which is directly relevant to a quantised NNUE. `-mcpu=bogus-cpu` errors out, so the flag is validated, and `-mcpu=apple-m5` is also accepted by this compiler.

**Stockfish's ARM handling.** `ARCH=apple-silicon` sets `arch = arm64`, `neon = yes` and `dotprod = yes`, and the dot product path compiles with `-march=armv8.2-a+dotprod -DUSE_NEON_DOTPROD` [13]. The `macos-lipo` target builds the ARM half with `make profile-build ARCH=apple-silicon` and then `lipo`s it together with the x86 half [13]. No special workarounds appear in the Makefile for macOS ARM beyond that, which suggests building on this platform is uneventful.

**GCC is worse, and silently.** GCC's AArch64 host detection reads `/proc/cpuinfo`, with no sysctl fallback. On macOS that file does not exist, so detection fails and GCC falls back to its compiled-in defaults, `generic-armv8-a` [36]. Homebrew GCC's `-march=native` and `-mcpu=native` on an M-series Mac therefore give you generic ARMv8-A, losing dot product, FP16 and everything else the chip has. The fix is to spell the CPU out: GCC documents `apple-m1` through `apple-m5` as `-mcpu` values [36].

**Rust handles this correctly and this is a genuine Rust advantage.** rustc's `-C target-cpu=native` resolves through LLVM's `getHostCPUName()`, which on Apple platforms reads the `hw.cpufamily` sysctl and maps it to `apple-m1` through `apple-m5` [16][37]. It does not touch `/proc/cpuinfo`. So on Apple Silicon:

| Toolchain | `-march=native` | `-mcpu=native` or `target-cpu=native` | Correct explicit flag |
|---|---|---|---|
| Apple clang 21, arm64 | accepted, downgrades to a v8.5a base, loses AES, SHA and FP16 | correct, identical to the default | `-mcpu=apple-m4` |
| Homebrew GCC, aarch64-darwin | falls back to `generic-armv8-a` | falls back to `generic-armv8-a` | `-mcpu=apple-m4` |
| rustc, `aarch64-apple-darwin` | not applicable | correct, via `hw.cpufamily` sysctl | `-C target-cpu=apple-m4` |

Rust wins this cleanly. One flag, one spelling, works on every platform, no `-march` versus `-mcpu` distinction to get wrong. Two of the C and C++ engines surveyed get it wrong: Ethereal and Obsidian both hardcode `-march=native`. Stormphrax gets it right with an explicit `apple-m1` target using `-mcpu=apple-m1 --target=arm64-apple-macos11` [40], and Berserk gets it right by detecting `uname -m` as `arm64` and routing to plain `-arch arm64` instead of `-march=native` [41].

**Rust on Apple Silicon in practice.** `aarch64-apple-darwin` is a tier 1 Rust target. Viridithas has a dedicated `aarch64-apple` Makefile target that builds with profile-guided optimisation [8]. Velvet's only live release job produces an `apple-silicon` artifact via `cargo pgo` on that target [11]. Neither carries a workaround comment. `-C target-cpu` takes the same LLVM CPU names as Clang's `-mcpu` [16].

Run `rustc --print target-cpus --target aarch64-apple-darwin` before pinning a flag. I could not run it: no Rust toolchain is installed on this machine. The accepted names come from LLVM's AArch64 processor table, where `apple-m4` and `apple-m5` are real definitions and `apple-m1` through `apple-m3` are aliases for `apple-a14` through `apple-a16` [37]. Whether your installed rustc's bundled LLVM knows `apple-m5` depends on its version.

**One macOS build-friction point that favours Rust.** Apple does not ship `llvm-profdata` on the default path, so C++ profile-guided builds on macOS need `xcrun` or a separately installed LLVM. Stockfish works around this with an `xcrun` prefix and a three-level search, under a Makefile comment warning that `llvm-profdata` must be version-compatible with the chosen compiler [13]. Ethereal and Obsidian have no such handling, so their profile-guided paths do not work out of the box on a Mac. In Rust it is `rustup component add llvm-tools`, and `cargo-pgo` finds it [15][18].

Do not pass `-C target-feature=+crt-static` on an Apple target. It is ignored: only certain MSVC and musl targets can switch C runtime linkage [33]. Several Rust engines pass it anyway, copied from their Linux recipes.

**Practical build settings to start from.** In `Cargo.toml`:

```toml
[profile.release]
lto = "fat"
codegen-units = 1
panic = "abort"
```

And build with `RUSTFLAGS="-C target-cpu=apple-m4"`, falling back to `native` if `apple-m4` is not in your `--print target-cpus` output. Add profile-guided optimisation later, via `cargo-pgo`, once you have a `bench` command [18].

**A note on the research machine.** This note was written on the current laptop, an Apple M1 Pro. The Mac Studio (M4 Max) named in the map had not arrived yet. An M1 has neither i8mm nor SME, so any `-mcpu=apple-m4` flag in a Makefile must wait until builds run on the Studio.

## Open questions

**What profile-guided optimisation is actually worth in Elo.** Every serious engine in both languages ships it, but I found no published measurement for a chess engine specifically, in the Stockfish Makefile, the Stockfish documentation, or the `cargo-pgo` README. Measure the speedup yourself once you have a `bench` command, then convert with `Elo_stc(x) = 2.10 x` [31]. This is cheap and the answer is directly useful.

**Whether LLVM BOLT is worth stacking on top.** `cargo-pgo` wraps it [18], and it gave the Rust compiler team 2 to 5 percent on top of profile-guided optimisation [34]. But Rust's own distribution pipeline disables BOLT on aarch64 with a comment reading "Enable bolt for aarch64 once it's fixed upstream. Broken as of December 2024" [38]. Whether that is still true in 2026 I did not establish. Park it.

**Whether `dotprod` is enabled by default on `aarch64-apple-darwin`.** rustc hardcodes `cpu: "apple-m1"` for that target, and LLVM's `apple-m1` model includes dot product, so `target_feature = "dotprod"` should be on with no flag. I could not confirm this empirically: no Rust toolchain is installed on this machine. Run `rustc --print cfg --target aarch64-apple-darwin | grep dotprod` and check before relying on it. Stockfish does not rely on inference here; it sets `-march=armv8.2-a+dotprod` explicitly [13].

**Whether the RustWeek 2026 talk contains better evidence.** "Writing a top-10 chess engine in Rust", by Cosmo Bobak and Kora, promises coverage of "sub-microsecond neural network inference". The recording is at https://www.youtube.com/watch?v=Gx21yYLwn10. The transcript could not be fetched. It is the most likely place an engine author states something explicit about Rust performance on this workload.

**Nobody has actually written down that Rust autovectorisation is inadequate for NNUE.** I searched for it and found no such statement from any engine author, on TalkChess, in any README, or in any blog post. The evidence that hand-written SIMD is required is language-neutral and comes from the activation function [43]. Absence of evidence here, but the absence is consistent across both languages.

**Nodes per second, Rust versus C++, same algorithm.** No apples-to-apples public figure exists. The only way to settle it is to write the same hot loop in both and compare, which is a microbenchmark and therefore explicitly out of scope for this ticket. The Elo evidence makes it moot.

**Whether the M4's SME and SME2 extensions are usable for NNUE.** `-mcpu=apple-m4` enables them, but I found no chess engine using Scalable Matrix Extension. Both Stockfish and the Rust engines stop at NEON and dot product. Treat SME as unexplored territory, not as a reason to pick a language.

**Confirmation of some language attributions.** Language labels for PlentyChess, Obsidian, Alexandria, Torch, Berserk, Ethereal, Koivisto and Stormphrax in the rating tables come from repository language statistics and READMEs. Alexandria and Berserk are C rather than C++. None of these labels changes the conclusion, but verify before quoting the table elsewhere.

## Sources

1. CCRL Blitz Rating List, all engines, best versions only. Computed 2026-09-12 with Bayeselo on 2,115,612 games. https://computerchess.org.uk/ccrl/404/
2. CCRL 40/15 Rating List, all engines, best versions only. Computed 2026-09-10 with Bayeselo on 2,443,716 games. https://computerchess.org.uk/ccrl/4040/
3. Reckless chess engine repository and README. https://github.com/codedeliveryservice/Reckless
4. Reckless Makefile. https://github.com/codedeliveryservice/Reckless/blob/main/Makefile
5. Reckless Cargo.toml. https://github.com/codedeliveryservice/Reckless/blob/main/Cargo.toml
6. Reckless NEON SIMD module. https://github.com/codedeliveryservice/Reckless/blob/main/src/nnue/simd/neon.rs
7. Viridithas chess engine repository and README. https://github.com/cosmobobak/viridithas
8. Viridithas Makefile. https://github.com/cosmobobak/viridithas/blob/master/Makefile
9. Viridithas NNUE SIMD module. https://github.com/cosmobobak/viridithas/blob/master/src/nnue/simd.rs
10. Velvet chess engine repository. https://github.com/mhonert/velvet-chess
11. Velvet Makefile and profile-guided release workflow. https://github.com/mhonert/velvet-chess/blob/master/.github/workflows/release_pgo.yml
12. Carp chess engine repository. https://github.com/dede1751/carp
13. Stockfish Makefile. https://github.com/official-stockfish/Stockfish/blob/master/src/Makefile
14. Stockfish NNUE affine transform layer, including NEON and NEON dot product paths. https://github.com/official-stockfish/Stockfish/blob/master/src/nnue/layers/affine_transform.h
15. rustc book, profile-guided optimization. https://doc.rust-lang.org/rustc/profile-guided-optimization.html
16. rustc book, codegen options, including `target-cpu` and `target-feature`. https://doc.rust-lang.org/rustc/codegen-options/index.html
17. Cargo book, profiles, including `lto`, `codegen-units` and `panic`. https://doc.rust-lang.org/cargo/reference/profiles.html
18. cargo-pgo. https://github.com/Kobzol/cargo-pgo
19. Clang user manual, including profile-guided optimization and the sanitizers. https://clang.llvm.org/docs/UsersManual.html
20. `core::arch::aarch64::vdotq_s32`, documented as a safe function stable since Rust 1.98.0. Verified 2026-09-15. https://doc.rust-lang.org/core/arch/aarch64/fn.vdotq_s32.html
21. Rust tracking issue 117224, NEON dot product intrinsics, closed as completed 2026-06-03. https://github.com/rust-lang/rust/issues/117224
22. `std::simd`, documented as a nightly-only experimental API. https://doc.rust-lang.org/std/simd/index.html
23. Rust tracking issue 86656, portable SIMD. https://github.com/rust-lang/rust/issues/86656
24. `core::arch::aarch64`, stable NEON intrinsics. https://doc.rust-lang.org/core/arch/aarch64/index.html
25. The Rust Programming Language, chapter 3.2, integer overflow in debug and release builds. https://doc.rust-lang.org/book/ch03-02-data-types.html
26. OpenBench wiki, Requirements For Public Engines. https://github.com/AndyGrant/OpenBench/wiki/Requirements-For-Public-Engines
27. OpenBench wiki, Configuring New Engines. https://github.com/AndyGrant/OpenBench/wiki/Configuring-New-Engines
28. cozy-chess. https://github.com/analog-hors/cozy-chess
29. shakmaty. https://github.com/niklasf/shakmaty
30. Cosmo Bobak, "NNUE performance improvements", 2024-06-01. Developer blog post, not a controlled benchmark. https://asteri.sm/files/2024-06-01-nnue.html
31. Stockfish documentation, Useful Data, "Elo from speedups", giving `Elo_stc(x) = 2.10 x` and `Elo_ltc(x) = 1.43 x`. https://official-stockfish.github.io/docs/stockfish-wiki/Useful-data.html
32. Princhess, a Monte Carlo tree search engine in Rust. https://github.com/princesslana/princhess
33. Rust reference, linkage, on `+crt-static` being ignored on targets that cannot switch C runtime linkage. https://doc.rust-lang.org/reference/linkage.html
34. Jakub Beranek, "Optimizing Rust programs with PGO and BOLT using cargo-pgo", reporting roughly 1 percent instruction count from profile-guided optimisation on rustc and 2 to 5 percent cycles from BOLT. Developer blog post. https://kobzol.github.io/rust/cargo/2023/07/28/rust-cargo-pgo.html
35. Jakub Beranek, Rust compiler performance report, June and July 2026, reporting roughly 4 to 5 percent wall-time gains from profile-guided optimisation on rustdoc, Clippy and Cargo. Developer blog post. https://kobzol.github.io/rust/2026/08/03/stf-june-july-2026.html
36. GCC documentation, AArch64 options, including the `native` caveat and the `apple-m1` through `apple-m5` CPU names. https://gcc.gnu.org/onlinedocs/gcc/AArch64-Options.html
37. LLVM AArch64 processor definitions and host CPU detection. https://github.com/llvm/llvm-project/blob/main/llvm/lib/Target/AArch64/AArch64Processors.td and https://github.com/llvm/llvm-project/blob/main/llvm/lib/TargetParser/Host.cpp
38. Rust `opt-dist`, the distribution build pipeline that applies profile-guided optimisation and BOLT to rustc, with BOLT disabled on aarch64. https://github.com/rust-lang/rust/blob/master/src/tools/opt-dist/README.md
39. fastchess, the match runner used by OpenBench. https://github.com/Disservin/fastchess
40. Stormphrax build configuration, showing `-mcpu=apple-m1` for its Apple Silicon target. https://github.com/Ciekce/Stormphrax/blob/master/build.mk
41. Berserk makefile, showing arm64 detection routing away from `-march=native`. https://github.com/jhonnold/berserk/blob/main/src/makefile
42. Carp makefile, showing its profile-guided optimisation passes. https://github.com/dede1751/carp/blob/main/makefile
43. Chess Programming Wiki, NNUE, on clipped rectified linear unit activation being autovectorisable and squared clipped rectified linear unit not being. https://www.chessprogramming.org/NNUE
44. Rust 1.86.0 release announcement, 2025-04-03, stabilising `target_feature_11`, safe functions carrying `#[target_feature]`. https://blog.rust-lang.org/2025/04/03/Rust-1.86.0/
45. Rust 1.87.0 release announcement, 2025-05-15, "Safe architecture intrinsics". https://blog.rust-lang.org/2025/05/15/Rust-1.87.0/
