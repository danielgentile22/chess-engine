# Search survey: what a 2026 top engine's search consists of

Researched 2026-09-16.

Scope: the search half of a single-threaded alpha-beta engine, from plain minimax to the pruning, reduction and history machinery in Stockfish, Reckless, Viridithas and Stormphrax as of September 2026. For calibration, Reckless 0.9.0 and Stormphrax 8.0.0 sit near 3600 on the CCRL 40/15 list, several hundred Elo above the 3000 target [34][36]. Evaluation (NNUE, the neural network evaluation) is out of scope except where search depends on it. Elo figures are quoted with their source; they were measured in specific engines at specific eras and are not additive.

## Answer

A modern engine's search is a depth-first minimax search with alpha-beta pruning, written in negamax form (one function, scores always from the side to move's point of view, so the child score is negated). Everything else in this document is a way to search fewer nodes for the same answer, or to spend the saved nodes where they matter. The techniques divide into four families:

1. **Bookkeeping that makes everything else possible.** Iterative deepening (search depth 1, then 2, then 3, reusing what was learned), a transposition table (a hash table of positions already searched), and move ordering (try the likely best move first so alpha-beta cuts off early). These are not optional. Ethereal's removal test measured the history tables alone at over 700 Elo, because without them every other reduction and pruning rule misfires [9].
2. **Forward pruning: refuse to search a subtree at all.** Null-move pruning, reverse futility pruning, razoring, futility pruning, late move pruning, static exchange evaluation (SEE) pruning, history pruning, ProbCut, multi-cut. Null-move pruning was worth about 93 Elo and late move pruning about 77 Elo when removed from Ethereal in 2020 [9].
3. **Reductions and extensions: search a subtree, but shallower or deeper.** Late move reductions (LMR) are the single largest gain after move ordering: removing them cost Ethereal about 249 Elo [9]. Singular extensions, check extensions and internal iterative reductions are in the 10 to 60 Elo range [9][17].
4. **Accuracy at the leaves.** Quiescence search (search captures until the position is quiet before calling the evaluation) is worth roughly 155 Elo in Stockfish's own annotation [10]. Correction history (learn how wrong the static evaluation tends to be and fix it) is the main post-2023 addition [32].

The consistent picture from the removal data is: a handful of techniques (move ordering with history, LMR, quiescence, null move, late move pruning, transposition table) account for many hundreds of Elo each or together, and everything after that is 5 to 30 Elo of polish per item that only shows up in tests of tens of thousands of games [9][10][17][20]. The polish items also depend on the big ones: LMR needs good ordering, singular extensions need the transposition table, correction history needs a stable static evaluation.

### Summary table

Elo figures: "E" is Ethereal 2020 removal test at 12s+0.12s, one thread [9]; "SF" is a Stockfish self-test or code annotation [10][15][16]; "W" is a Weiss addition test [17][18][19][20]. Removal numbers overstate what a fresh implementation gains, because the rest of the engine was tuned around the feature.

| Technique | Problem solved | Rough worth | Depends on | Where to read |
|---|---|---|---|---|
| Alpha-beta, negamax | Minimax visits every node | Foundation, no figure | Nothing | Weiss `search.c` `AlphaBeta` [8] |
| Iterative deepening | Unknown depth budget, cold move ordering | Foundation; required by aspiration, time management, TT move | Nothing | Reckless `search.rs` `start` [4] |
| Transposition table | Repeated positions, no best move memory | Large; Stockfish loses about 48 Elo going from 64 MB to 1 MB at 60s, so table quality matters [23] | Zobrist keys | Reckless `transposition.rs`; Weiss TT replacement commit [19] |
| Move ordering: hash move, MVV-LVA, SEE, killers, history | Alpha-beta cutoff rate | History tables: minus 759 Elo when removed (E) [9]; main history about 11 Elo and continuation history about 63 Elo in Stockfish 2022 [10] | TT for hash move | Stockfish `movepick.cpp` `MovePicker::score` [2]; Reckless `history.rs` [5] |
| Principal variation search | Full-window searches everywhere | About 10 percent fewer nodes with good ordering [29] | Move ordering | Reckless `search.rs` "Principal Variation Search" [4] |
| Quiescence search | Horizon effect on captures | About 155 Elo (SF annotation) [10] | Capture ordering, SEE | Stockfish `search.cpp` `qsearch` [1] |
| Aspiration windows | Root search with infinite window wastes nodes | About 11 Elo when introduced (SF 2009) [15]; tweaks 5 to 7 Elo (W) [20] | Iterative deepening | Stockfish `search.cpp` `iterative_deepening` [1] |
| Null-move pruning | Positions so good the opponent cannot recover | Minus 93 Elo removed (E) [9] | Static eval, TT | Reckless `search.rs` "Null Move Pruning" [4] |
| Reverse futility pruning | Static eval far above beta at low depth | Minus 32 Elo removed (E, as "Beta Pruning") [9] | Static eval | Reckless `search.rs` "Reverse Futility Pruning" [4] |
| Razoring | Static eval far below alpha at low depth | Near zero in Ethereal 2016 [22]; removed from Stockfish in 2020 as ineffective, later restored [13][1] | Quiescence | Stockfish `search.cpp` Step 8 [1] |
| Late move reductions | Late moves searched at full depth | Minus 249 Elo removed (E) [9] | Move ordering, history | Stockfish `search.cpp` Step 18 and `reduction()` [1] |
| Late move pruning | Many hopeless quiet moves at low depth | Minus 77 Elo removed (E) [9] | Move ordering, improving flag | Ethereal LMP commit [21]; Reckless "Late Move Pruning" [4] |
| Futility pruning (parent) | Quiet moves that cannot raise alpha | Minus 3 Elo removed (E) [9] | Static eval | Stockfish Step 15 [1] |
| SEE pruning | Losing captures and bad quiets at low depth | Minus 42 Elo removed (E) [9] | SEE | Viridithas `static_exchange_eval` [6] |
| History pruning | Quiets with terrible history | Counter-move pruning minus 8 Elo (E) [9] | Continuation history | Stockfish Step 15 [1] |
| Singular extensions, multi-cut | Forced moves need more depth | Plus 12 to 24 Elo added (W) [17]; extensions in total minus 60 Elo removed (E) [9] | TT with move, depth, bound | Stockfish Step 16 [1] |
| Internal iterative reductions | No hash move means poor ordering | Small, order of a few Elo per tweak (SF) [14] | TT | Stockfish Step 11 [1] |
| ProbCut | Good captures that clearly refute the node | Minus 9 Elo removed (E) [9]; plus 7 Elo when extended (SF 2011) [16] | SEE, capture history | Stockfish Step 12 [1] |
| Improving heuristic | Pruning margins should respect eval trend | About 5 Elo per use (E) [9] | Eval stack | Reckless `improvement` [4] |
| Correction history | Static eval has systematic bias | Passed Fishtest as a gainer; larger at long time control [11][32] | Pawn and piece hash keys | Stockfish `search.cpp` `correction_value` [1]; Reckless `eval_correction` [4] |
| Time management | Fixed time per move wastes clock | Later item, tens of Elo in practice | Iterative deepening | Reckless `time.rs`; Viridithas `timemgmt.rs` |
| Lazy SMP | Multiple cores | Later item, scales well to 8 cores [31] | Shared TT | Reckless `threadpool.rs` |

## Suggested implementation order

The order follows dependencies and Elo-per-effort. Each tier should be measured with a self-play match (SPRT, sequential probability ratio test, the standard stopping rule) before moving on, because every later technique silently depends on the earlier ones being correct.

**Tier 0: plays legal chess.** Move generation verified by perft (node counts at fixed depth against known values), negamax alpha-beta, a material plus piece-square evaluation, and a fixed-depth search. No figure; this is the baseline.

**Tier 1: the frame (worth hundreds of Elo together).** Iterative deepening, quiescence search with MVV-LVA capture ordering, transposition table with Zobrist hashing and a hash move, and basic move ordering (hash move, captures by MVV-LVA, killers, history). Rationale: everything below assumes these. Ethereal's data puts history alone at hundreds of Elo [9] and Stockfish puts quiescence at about 155 [10].

**Tier 2: the big pruners (about 100 to 300 Elo each).** Principal variation search, null-move pruning, late move reductions with the log formula, reverse futility pruning, late move pruning. Rationale: LMR and null move are the two largest measured gains after ordering [9]. LMR needs Tier 1 ordering or it reduces the wrong moves.

**Tier 3: sharper leaves and ordering (about 10 to 60 Elo each).** Static exchange evaluation, then SEE pruning in quiescence and main search, delta and futility pruning in quiescence, continuation history (counter-move and follow-up), capture history, aspiration windows, check extensions, improving flag. Rationale: each is small but they compound and are prerequisites for Tier 4.

**Tier 4: the modern layer (about 5 to 30 Elo each).** Singular extensions with multi-cut and negative extensions, ProbCut, internal iterative reductions, history pruning, futility pruning at the parent, razoring, correction history, cutnode and TT-move dependent reduction adjustments, hindsight reductions. Rationale: these need a mature TT, histories and a stable evaluation to be measurable.

**Tier 5: later.** Time management with best-move and score stability, Lazy SMP, tablebases, NNUE (its own ticket). Correction history moves up if the evaluation is already NNUE.

## Minimax, alpha-beta and negamax

Problem: chess is a two-player zero-sum game; the value of a position is the best move for me assuming the best reply for you, recursively. Minimax computes that but visits every node. Alpha-beta keeps two bounds while searching: alpha (the best score I can already guarantee) and beta (the best the opponent can already guarantee). Once a move proves the position is at least beta, the opponent will never allow it, so the remaining moves at that node are skipped (a "beta cutoff" or "fail high"). With perfect move ordering alpha-beta searches roughly the square root of the minimax tree; Knuth and Moore's 1975 analysis is the reference [29][37]. Negamax is the standard coding form: one function, `score = -search(-beta, -alpha)`, so both sides share the code. A "fail low" means no move beat alpha; a "fail soft" search returns the best score seen even when it is outside the window, which later techniques (multi-cut, ProbCut) rely on. Node types matter for later steps: a PV node (principal variation, the line both sides consider best) has an open window; a cut node is expected to fail high; an all node is expected to fail low [1].

Pitfalls: mate scores must be stored relative to the root and adjusted by ply; the search must never return a value outside the bounds it promised. Read: Weiss `src/search.c`, `AlphaBeta` [8]; Viridithas `alpha_beta` [6].

## Iterative deepening

Problem: you do not know how deep you can afford to search in the time available, and the transposition table and history tables start empty. Idea: search to depth 1, then 2, then 3, until time runs out, and play the best move from the last completed iteration. Each iteration's results (hash moves, killers, history) make the next iteration's ordering good enough that the total cost is little more than the last iteration alone [4]. It is also what makes aspiration windows and time management possible. Read: Reckless `src/search.rs`, `start`, loop marked "Iterative Deepening" [4]; Viridithas `iterative_deepening` [6].

## Transposition table

Problem: the same position is reached by different move orders (transpositions) and by successive iterations of iterative deepening. Idea: hash every position to a 64-bit Zobrist key (a random number per piece-square, XORed together and updated incrementally on make and unmake, Zobrist 1970) and store per position: the key or part of it, the depth searched, the score, the bound type (exact, lower bound from a fail high, upper bound from a fail low), the best move, and an age or generation counter [27]. On entry, if the stored depth is sufficient and the bound allows it, return immediately; otherwise use the stored move first in ordering. Replacement: Weiss only overwrites when the new entry is for a different position, at least as deep, or exact [19]. Stockfish uses buckets of a few entries per cache line and an aging generation so entries from old searches get replaced first [1][27]. Stockfish's own measurement: shrinking the table from 64 MB to 1 MB at 60s+0.6s costs about 48 Elo, and from 256 MB to 4 MB at 240s costs about 52 Elo [23].

Pitfalls: mate scores must be converted to "mate in N from this node" before storing and back after loading; the hash move must be validated as pseudo-legal before playing it (key collisions happen); in PV nodes many engines do not cut off from the table so the reported line is complete. Dependencies: everything in Tier 2 and up reads `tt_move`, `tt_depth`, `tt_bound` and `tt_score` [4]. Read: Reckless `src/transposition.rs` [4]; Viridithas `src/transpositiontable.rs`.

## Move ordering: hash move, captures, killers, history

Problem: alpha-beta only pays off if the cutoff move is tried early; in Stockfish about 75 percent of cutoffs at nodes with a hash move come from the hash move [27]. Idea: a staged move picker generates and yields moves in tiers so that the search often never generates the later tiers. Stockfish's stages are: transposition table move, then captures split into good and bad by a static exchange evaluation threshold, then quiets ordered by history, then the bad captures, then bad quiets [2]. Capture scoring inside a stage is capture history plus the value of the captured piece; MVV-LVA (most valuable victim, least valuable attacker) is the beginner version of the same ranking [2]. Killer moves are quiet moves that caused a cutoff at the same ply in a sibling node. The history heuristic (Schaeffer 1983) is a table indexed by side and from-to square, incremented when a quiet move causes a cutoff and decremented (a "malus") for the quiet moves tried before it; modern engines scale the update as `bonus - entry * |bonus| / limit` ("history gravity") so the table saturates gracefully [28][3]. Modern forms: continuation history indexes the current move's piece and destination by the previous move's piece and destination, one, two, four and six plies back, which subsumes the counter-move heuristic [3][28]; capture history is indexed by piece, destination and captured piece type [3]; pawn history is keyed by the pawn structure hash [3][5]; Reckless also keys quiet history by whether the from and to squares are attacked ("threats") [5]. Measured worth: Ethereal without any history lost about 759 Elo, essentially every game [9]; Stockfish's 2022 annotation put main history at about 11 Elo and continuation history at about 63 Elo on top of everything else [10].

Pitfalls: history values feed LMR and pruning decisions, so their scale must be stable; update killers and history only for quiet moves. Read: Stockfish `src/movepick.cpp`, `MovePicker::score` and `next_move`, `src/history.h` for the table types [2][3]; Reckless `src/history.rs`, `QuietHistory`, `NoisyHistory`, `ContinuationHistory` [5]; Viridithas `src/history.rs`, `update_quiet_history` [7].

## Principal variation search

Problem: once the first move at a node has set alpha, later moves rarely beat it, yet a full window search of each costs as much as the first. Idea (Marsland and Campbell 1982, equivalent to Reinefeld's NegaScout): search the first move with the full window, then every other move with a null window `(alpha, alpha + 1)`, which only answers "is this better than alpha?" and is cheaper. If the answer is yes and we are in a PV node, re-search with the full window to get the exact score [29]. Worth about 10 percent of nodes with good ordering [29]; it is the frame LMR hangs on, since LMR is a null-window search at reduced depth followed by the same re-search. Read: Reckless `src/search.rs`, comment "Principal Variation Search" [4].

## Quiescence search

Problem: the horizon effect. If the evaluation is called in the middle of an exchange, the score is nonsense (a queen is hanging). Idea: at depth zero do not stop; search only captures (and sometimes checks) until the position is quiet, using the static evaluation as a "stand pat" lower bound so the side to move can decline to capture [1]. Stockfish annotates it as worth about 155 Elo [10]. Pruning inside quiescence: delta or futility pruning skips a capture when static eval plus the captured piece's value cannot reach alpha; SEE pruning skips captures that lose material; Stockfish also limits non-check captures to the first two moves when futility applies [1]. Reckless adds a small transposition table probe and late move pruning in quiescence [4].

Pitfalls: quiescence can explode without a capture ordering and SEE; when in check all evasions must be searched; depth bookkeeping must not let quiescence recurse forever on checks. Read: Stockfish `src/search.cpp`, `Search::Worker::qsearch`, Step 6 "Pruning" [1]; Weiss `src/search.c`, `Quiescence` [8].

## Static exchange evaluation

Problem: "is this capture good?" needs the whole exchange sequence on one square, not just victim minus attacker. Idea: simulate the sequence of captures on the target square, least valuable attacker first, and return the net gain; most engines implement a threshold form `see_ge(move, threshold)` that answers "does this move gain at least threshold?" without computing the exact value [1]. Uses: split good from bad captures in move ordering [2]; prune losing captures in quiescence and, with a depth-scaled margin, in the main search (Stockfish prunes quiets failing `see_ge(-23 * lmrDepth * lmrDepth)` and captures failing `-177 * depth` adjusted by capture history) [1]. Ethereal measured SEE pruning at about 42 Elo [9]. Read: Viridithas `src/search.rs`, `static_exchange_eval` [6]; Stormphrax `src/see.cpp`.

## Check extensions

Problem: a forcing check sequence can push a tactic just past the horizon. Idea: when a move gives check, search the reply one ply deeper. This was standard for decades; Ethereal and Reckless have both simplified or moved their versions and the gain is now small, since singular extensions and quiescence cover most of the same ground [9][34]. Ethereal's combined "Extensions" removal cost about 60 Elo, most of which is singular [9]. Pitfall: without a cap, perpetual checks make the search depth unbounded. Read: Weiss `src/search.c`, `AlphaBeta` [8].

## Null-move pruning

Problem: many nodes are so good for the side to move that no search is needed to confirm a cutoff. Idea (Beal 1989, Donninger 1993, Heinz 1999 adaptive): give the opponent a free move (pass) and search the result at reduced depth with a null window around beta. If even after passing the score is still at least beta, assume the real best move is better still and return [24]. Stockfish's reduction is `R = 7 + depth / 3 + max((staticEval - beta) / 256, 0)` in units of plies, and it runs only at cut nodes with static eval at or above beta, with a verification search at depth 16 and above where null move is disabled for a few plies to catch zugzwang [1]. Reckless is similar and additionally refuses when a singular extension is likely [4]. Measured: minus 93 Elo when removed from Ethereal [9].

Pitfalls: never when in check or when the side to move has only pawns and king (zugzwang, a position where every move worsens things); never two null moves in a row; do not return unproven mate scores. Dependencies: a static evaluation and the improving flag. Read: Reckless `src/search.rs`, "Null Move Pruning (NMP)" [4]; Viridithas `alpha_beta`, comment "null-move pruning" and the verification search block [6].

## Reverse futility pruning and razoring

Reverse futility pruning (also "static null move pruning" or "beta pruning"): at low depth, if the static evaluation minus a depth-scaled margin is still at least beta, return without searching; the margin in Stockfish is about 45 to 85 per ply, reduced when improving, and the return value is a blend of beta and eval [1]. Reckless scales the margin quadratically in depth and adds a term from the size of the correction history value [4]. Ethereal measured it at about 32 Elo [9]. Razoring is the mirror image: if eval is far below alpha (Stockfish: `alpha - 482 * depth`), drop straight into quiescence [1]. Its value is marginal; Ethereal found zero effect in 2016 [22], Stockfish removed it in 2020 as ineffective [13] and later reinstated a simpler form [1]. Dependencies: a static evaluation that is called at every interior node and cached on the search stack. Read: Reckless `src/search.rs`, "Razoring" and "Reverse Futility Pruning (RFP)" [4]; Viridithas `rfp_margin` [6].

## Late move reductions

Problem: with good ordering the cutoff move is nearly always among the first few, so searching the 20th quiet move at full depth is wasted. Idea (Fruit and Glaurung 2005): search late moves at reduced depth with a null window; if the reduced search beats alpha, re-search at full depth [25]. The base reduction is a table indexed by depth and move number built from a formula like `a + ln(depth) * ln(moveCount) / b` [25]; Stockfish stores reductions in fixed-point units of 1/1024 ply and then adds and subtracts dozens of terms: more reduction at cut nodes, when the hash move is a capture, when the next ply has had many cutoffs, when not improving, at expected all nodes; less reduction for PV and formerly-PV nodes, for the hash move, for moves with good history and continuation history, and by the size of the correction value [1]. After a reduced search fails high, Stockfish and Reckless choose to re-search one ply deeper or shallower depending on how far the score moved [1][4]. Reckless adds a small pseudo-random term to reductions per node [4]. Measured: minus 249 Elo when removed from Ethereal [9], the largest single search feature after history.

Pitfalls: do not reduce below depth 1; reduce the hash move and killers less or not at all; the re-search must use the full window in PV nodes; a wrong sign on a history term is a silent 50 Elo loss. Dependencies: move ordering and history (the reduction is a bet that the ordering is right), the improving flag, cut node flag. Read: Stockfish `src/search.cpp`, Step 18 and `Search::Worker::reduction` [1]; Reckless `src/search.rs`, "Late Move Reductions (LMR)" [4]; Viridithas `lm_reduction` and the "extend/reduce using the stat_score" block [6].

## Late move pruning and history pruning

Late move pruning (move count pruning): at low depth, after `N` quiet moves have been tried (Stockfish: `(3 + depth * depth) / (2 - improving)`), skip the remaining quiets entirely [1]. Ethereal measured about 77 Elo [9]. History pruning: skip a quiet whose continuation history sum is below `-4136 * depth` in Stockfish [1]; Ethereal's counter-move pruning was worth about 8 Elo [9]. Futility pruning at the parent (Heinz 1998 for the extended, two-ply form): skip a quiet when `staticEval + margin(lmrDepth)` cannot reach alpha, Stockfish only when `lmrDepth < 12` [1][26]; Ethereal measured about 3 Elo [9]. All of these run inside the move loop after the LMR depth has been computed, so they share the `lmrDepth` value. Pitfalls: never prune when a mate score is in play, never prune when in check, always search at least one legal move. Read: Stockfish `src/search.cpp`, Step 15 [1]; Reckless `src/search.rs`, "Late Move Pruning (LMP)", "Futility Pruning (FP)", "History Pruning (HP)" [4].

## Aspiration windows

Problem: each iterative deepening iteration starts with an infinite window, but the score rarely moves far from the previous iteration. Idea: search the root with a narrow window around the last score (Stockfish: `delta = 5 + ...` centipawns, widened on each fail); on a fail low, lower alpha; on a fail high, raise beta and slightly reduce depth for the re-search [1]. Stockfish measured about 11 Elo when it was introduced in 2009 [15]; Weiss gained about 7 Elo from tuning the widening [20]. Pitfalls: the window must widen to infinite eventually; mate scores need special handling; it interacts with time management because a fail low is a signal to think longer. Read: Stockfish `src/search.cpp`, `iterative_deepening`, the `failedHighCnt` loop [1]; Reckless `start`, "Aspiration Windows" [4].

## Singular extensions, multi-cut and negative extensions

Problem: when one move is far better than all alternatives (a forced recapture, the only defence) the search should look deeper along it, and when many moves are good the node is safe to cut. Idea (Anantharaman, Campbell and Hsu 1988; Stockfish's cheap form since 2009): if the hash move has a lower bound score at sufficient stored depth, search all other moves at half depth with a null window at `ttValue - margin`, excluding the hash move. If they all fail low, the hash move is singular: extend it by one ply, two or three if the fail was by a large margin [1][30]. If instead the exclusion search fails high above beta, several moves refute the position, so return immediately (multi-cut) [1]. If it fails high but not above beta, and the node is a cut node, reduce the hash move instead (negative extension) [1][4]. Reckless adds a "low depth singular extension" that extends the first move at shallow cut nodes when the estimated score is below alpha [4]. Measured: Weiss gained 12 Elo at fast and 24 Elo at slow time control when adding it [17], plus 11 more from allowing it at lower depths [18]; Stockfish's early tweaks were 3 to 8 Elo each in 2011 and 2012 self-play [38].

Pitfalls: the excluded move must be threaded through the search state and must not be written to the transposition table; no recursive singular searches; the extension budget must be capped so depth cannot run away. Dependencies: a transposition table that stores move, depth and bound; correction history for the double-extension margin in Stockfish. Read: Stockfish `src/search.cpp`, Step 16 [1]; Reckless `src/search.rs`, "Singular Extensions (SE)", "Multi-Cut", "Negative Extensions" [4]; Viridithas `src/search.rs`, `is_forced` [6].

## Internal iterative reductions

Problem: a node with no hash move has poor ordering, so a deep search there is likely wasted. The old answer, internal iterative deepening, ran a shallow search first to find a move. The modern answer is to just reduce depth by one at PV and cut nodes without a hash move at depth 6 and above; Stockfish's comment notes that making it more aggressive scales poorly [1][14]. Viridithas removed internal iterative deepening in 2024 and does IIR when the hash entry is low quality [33]. Worth a few Elo per tweak in Fishtest [14]. Read: Stockfish `src/search.cpp`, Step 11 [1].

## ProbCut

Problem: at a node whose value is far above beta, a single good capture usually proves it, but null move may be off. Idea (Buro 1995, Stockfish form since 2011): with `probCutBeta = beta + margin`, try captures whose SEE gain covers the margin; run quiescence and then a reduced-depth search at `probCutBeta`; if one holds, store it and return [1][16]. Ethereal measured about 9 Elo [9]. Stockfish adds a "small ProbCut": if the hash entry already has a lower bound well above beta at near-depth, return it [1]. Dependencies: SEE-thresholded capture generation and capture history. Read: Stockfish `src/search.cpp`, Steps 12 and 13 [1]; Viridithas "adaptive probcut" block [6].

## The improving heuristic

Problem: pruning margins should be looser when the position is getting worse for us (our static eval is dropping) and tighter when it is getting better. Idea: compare static eval now with static eval two plies ago (or four if the two-ply value is missing), set `improving` when higher; Reckless keeps the signed difference as `improvement` and feeds it into margins directly [4]. Uses: late move pruning threshold, reverse futility margin, null move condition, LMR base reduction, ProbCut beta [1][4]. Ethereal measured single uses at about 5 Elo [9]. Dependencies: a static eval stack; `improving` is false when in check. Read: Reckless `src/search.rs`, `improvement` [4]; Viridithas comment on improving near line 1020 [6].

## Correction history

Problem: static evaluation has systematic errors that depend on features the search can observe: this pawn structure, this piece configuration, this last pair of moves. Idea (Caissa, October 2023, then everyone): after each search, record the difference between the search score and the static eval in tables keyed by pawn hash, non-pawn hash per colour, minor piece hash, and the previous two moves (continuation correction history); before pruning decisions, add a fraction of the stored correction to the static eval [32]. Stockfish introduced the extra keys in September 2024 crediting Sirius and Starzix [11], later removed the major-piece table as not pulling its weight [12], and in December 2025 shares the tables between threads [35]. Reckless also uses the absolute size of the correction as a "how uncertain is the eval" signal that widens reverse futility margins and shrinks LMR [4]. Worth: passed Stockfish's tests as a gainer and scales better at long time control; no single Elo figure in the sources [11][32]. Dependencies: incremental pawn and piece hash keys, a settled static evaluation; it gains most with NNUE. Read: Stockfish `src/search.cpp`, `correction_value` and `update_correction_history` [1]; Reckless `eval_correction` and `update_correction_histories` [4]; Viridithas `src/history.rs`, `update_correction_history` and `correction` [7].

## Other 2024 to 2026 frontier items

These are visible in the three reference searches and are each a few Elo:

- **Cutnode-dependent reductions.** Reduce more at expected cut nodes, and even more when there is no hash move [1][4].
- **TT-move dependent reductions.** Reduce more when the hash move is a capture; reduce less when the hash score beats alpha or the stored depth is at least the current depth [1][4].
- **Cutoff count.** Track how many fail-highs the next ply produced; many means the subtree is easy, so reduce more [1][4].
- **Hindsight reductions.** After the fact, if the parent reduced heavily and the eval swung, add or remove a ply of depth at the child (Reckless) [4].
- **Post-LMR depth adjustment.** After a reduced search fails high, search one ply deeper or shallower depending on the margin [1][4].
- **TT score as eval.** When the hash bound agrees with the window, use the stored score instead of static eval for pruning decisions [4][8].
- **Estimated score blends.** Return `lerp(eval, beta)` rather than raw eval from reverse futility and multi-cut, keeping fail-soft information [1][4].

## Time management and Lazy SMP

Time management: allocate a base time per move from the clock and increment, then scale it by how stable the best move and score have been across iterations, and stop early when the best move has not changed for several iterations; Reckless added stability-based adjustments in 0.7.0 [34]. This is worth tens of Elo in practice but should follow a working search. Lazy SMP: run one search per thread on the same root with a shared transposition table and slightly different depths or ordering; Stockfish adopted it in version 7 (2016) and it scales well to 8 cores and beyond in strength, though not in time-to-depth [31]. Stockfish now shares histories and correction history across threads, with careful non-regression testing [35]. Both are Tier 5.

## Open questions

- Which Elo figures transfer to a fresh Rust engine with a hand-crafted evaluation? All removal data here comes from mature engines; a per-tier self-play measurement plan is needed.
- Bit layout and bucket count for the transposition table in Rust (packed 10 byte entries, 3 or 4 per 32 byte bucket, atomics for Lazy SMP) is a design decision for the TT ticket.
- Where the static evaluation is cached during search (search stack, TT, or both) affects reverse futility, improving and correction history; pick one before Tier 2.
- Whether to keep killers at all: Viridithas still has `insert_killer`; Stockfish removed them in favour of history. Measure.
- SPSA tuning infrastructure (how Stockfish, Reckless and Viridithas set the hundreds of constants) is out of scope here and needs its own note.
- Correction history keys beyond pawn and non-pawn (minor, continuation, threats) have mixed results across engines [12][33]; test each.

## Sources

1. Stockfish `src/search.cpp` at commit 031dfeb4 (master, 2026-09-16). Search steps 1 to 24, `qsearch`, `reduction`, `correction_value`. https://github.com/official-stockfish/Stockfish/blob/031dfeb437fa6b06cdbdf4ef89dfb82f6b83c4d3/src/search.cpp
2. Stockfish `src/movepick.cpp`, same commit. Staged move picker, `MovePicker::score`, `next_move`. https://github.com/official-stockfish/Stockfish/blob/031dfeb437fa6b06cdbdf4ef89dfb82f6b83c4d3/src/movepick.cpp
3. Stockfish `src/history.h`, same commit. History table types (butterfly, continuation, capture, pawn, correction). https://github.com/official-stockfish/Stockfish/blob/031dfeb437fa6b06cdbdf4ef89dfb82f6b83c4d3/src/history.h
4. Reckless `src/search.rs` at commit 31d9cd6f (main, 2026-09-16). Rust reference search with labelled blocks for every technique. https://github.com/codedeliveryservice/Reckless/blob/31d9cd6fd2bea6d9f72eeb35e0bac70daa295fb1/src/search.rs
5. Reckless `src/history.rs`, same commit. Quiet, noisy, pawn, continuation and correction history structs. https://github.com/codedeliveryservice/Reckless/blob/31d9cd6fd2bea6d9f72eeb35e0bac70daa295fb1/src/history.rs
6. Viridithas `src/search.rs` at commit e605537c (master, 2026-09-16). `alpha_beta`, `quiescence`, `static_exchange_eval`, `rfp_margin`, `lm_reduction`, with extensive comments. https://github.com/cosmobobak/viridithas/blob/e605537c8a0ffe78e250d9419924ff6b23e98bad/src/search.rs
7. Viridithas `src/history.rs`, same commit. https://github.com/cosmobobak/viridithas/blob/e605537c8a0ffe78e250d9419924ff6b23e98bad/src/history.rs
8. Weiss `src/search.c` at commit c735b8f3 (master, 2026-09-16). Short C engine meant for reading: `AlphaBeta`, `Quiescence`, `CorrectEval`. https://github.com/TerjeKir/weiss/blob/c735b8f3d2ddb0cdf42b135a5fb42e21c01f7a3d/src/search.c
9. Ethereal commit e755a814 (2020-01-22), "Add elo estimates to search steps". Elo loss from removing each search step at 12s+0.12s, one thread. https://github.com/AndyGrant/Ethereal/commit/e755a8140f
10. Stockfish commit 8fadbcf1 (2022-05-30), "Add info about elo gained from some heuristics": about 11 Elo main history, 63 Elo continuation history, 155 Elo quiescence search. https://github.com/official-stockfish/Stockfish/commit/8fadbcf1b2
11. Stockfish commit 60351b9d (2024-09-12), "Introduce Various Correction histories". https://github.com/official-stockfish/Stockfish/commit/60351b9df9
12. Stockfish commit 831cb01c (2025-01-25), "Remove major corrhist". https://github.com/official-stockfish/Stockfish/commit/831cb01cea
13. Stockfish commit 8ec97d16 (2020-12-26), "Remove razoring". https://github.com/official-stockfish/Stockfish/commit/8ec97d161e
14. Stockfish commit 8dea0705 (2023-06-01), "Move internal iterative reduction before probcut", with Fishtest results. https://github.com/official-stockfish/Stockfish/commit/8dea070538
15. Stockfish commit 4634be8b (2009-04-16), "Merge Joona's new aspiration window search", +11 Elo over 999 games. https://github.com/official-stockfish/Stockfish/commit/4634be8ba6
16. Stockfish commit fca0a2dd (2011-05-21), "New extended probcut implementation", +7 Elo. https://github.com/official-stockfish/Stockfish/commit/fca0a2dd88
17. Weiss commit e668878b (2020-09-26), "Singular extension", +11.5 Elo at 10s and +24 Elo at 60s. https://github.com/TerjeKir/weiss/commit/e668878b94
18. Weiss commit 55edc993 (2023-01-31), "Singular Extension at lower depths", +11.5 Elo at 8s. https://github.com/TerjeKir/weiss/commit/55edc99364
19. Weiss commit 63fd2929 (2020-01-10), "TT replacement scheme". https://github.com/TerjeKir/weiss/commit/63fd2929f1
20. Weiss commit b086028c (2020-02-22), "Tweak aspiration", +6.8 Elo. https://github.com/TerjeKir/weiss/commit/b086028c3d
21. Ethereal commit 85bbcd31 (2017-07-12), "Added a form of Late Move Pruning (Move Count Pruning)". https://github.com/AndyGrant/Ethereal/commit/85bbcd31f7
22. Ethereal commit 9ac301ed (2016-08-24), razoring found to have zero Elo impact. https://github.com/AndyGrant/Ethereal/commit/9ac301ed74
23. Stockfish wiki, "Useful data": Elo versus hash size, MultiPV cost, tablebase gains. https://official-stockfish.github.io/docs/stockfish-wiki/Useful-data.html
24. Chess Programming Wiki, "Null Move Pruning" (Beal 1989, Donninger 1993, Heinz 1999). https://www.chessprogramming.org/Null_Move_Pruning
25. Chess Programming Wiki, "Late Move Reductions". https://www.chessprogramming.org/Late_Move_Reductions
26. Chess Programming Wiki, "Futility Pruning" (Heinz, "Extended Futility Pruning", ICCA Journal 21(2), 1998). https://www.chessprogramming.org/Futility_Pruning
27. Chess Programming Wiki, "Transposition Table" (Zobrist 1970; Thompson and Condon two-tier 1983). https://www.chessprogramming.org/Transposition_Table
28. Chess Programming Wiki, "History Heuristic" (Schaeffer, ICCA Journal 1983). https://www.chessprogramming.org/History_Heuristic
29. Chess Programming Wiki, "Principal Variation Search" (Marsland and Campbell 1982; Reinefeld NegaScout 1983). https://www.chessprogramming.org/Principal_Variation_Search
30. Chess Programming Wiki, "Singular Extensions" (Anantharaman, Campbell and Hsu 1988). https://www.chessprogramming.org/Singular_Extensions
31. Chess Programming Wiki, "Lazy SMP". https://www.chessprogramming.org/Lazy_SMP
32. Chess Programming Wiki, "Static Evaluation Correction History" (Caissa, October 2023). https://www.chessprogramming.org/Static_Evaluation_Correction_History
33. Viridithas GitHub releases v14 to v20, with per-version Elo and changelogs. https://github.com/cosmobobak/viridithas/releases
34. Reckless GitHub releases v0.7.0 to v0.10.0-dev, with per-version Elo and changelogs. https://github.com/codedeliveryservice/Reckless/releases
35. Stockfish commit 1a67ccc7 (2025-12-23), "Share correction history between threads". https://github.com/official-stockfish/Stockfish/commit/1a67ccc72e
36. Stormphrax README, rating table (CCRL 40/15 3605, CCRL Blitz 3747 for 8.0.0). https://github.com/Ciekce/Stormphrax
37. Knuth, D. E. and Moore, R. W., "An Analysis of Alpha-Beta Pruning", Artificial Intelligence 6(4), 1975. Cited via [29].
38. Stockfish commits 23cbb221 (2011-02-17, "Depth dependant singular extension margin", +3 Elo) and 4c91dbc2 (2012-10-02, "Further push singular extension", +8 Elo). https://github.com/official-stockfish/Stockfish/commit/4c91dbc28e
