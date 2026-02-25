Honest assessment: I think minor changes alone are unlikely to get reliably positive results on math/code with the current concept format + model combination. Here's why:

The core problem from the ablation is that concepts help and hurt in equal measure. On math, qwen-2.5-7b already knows "modular arithmetic" and "dynamic programming" — our concepts are restating knowledge it already has. On LCB, concepts push toward overcomplication when brute force works. This isn't a prompting issue, it's a content issue. No amount of filtering, routing, or rendering changes the fact that the concepts aren't adding information the model lacks.

That said, there are a few things worth trying before concluding we need a method revision:

1. Ensembling (your suggestion): run with and without concepts, pick the better answer. Zero method change, just eval strategy. This gives the ceiling for "concepts can only help" and would directly test whether concept-helped problems are recoverable. If oracle best-of shows a meaningful gain, a model-driven answer selector becomes the path forward.

2. Concepts only on retry: pass 1 no concepts, pass 2 with concepts for failures only. Also minimal change. Tests whether concepts help specifically when the model is stuck rather than as uniform guidance.

3. Hard-only subsets: lower baseline = more headroom. On math our L5 subset already has 58-63% baseline. On LCB we could filter to hard problems where the baseline is maybe 15-20%. Concepts might matter more when the model genuinely struggles.

4. We also just found and fixed a ground-truth leakage bug in the math and LCB feedback engines — on retry, the math engine was telling the model the expected answer, and the LCB engine was leaking private test results. All pass 2 scores were inflated. Need to re-run baselines with the fix before drawing any conclusions about pass 2.

For the resubmission vs next paper split: I think (1) and (3) are the safest bets for showing positive results with minimal method change. If ensembling shows a clear ceiling and hard-only subsets show lift, that's a legitimate finding for the resubmission. The deeper method changes (DC-style synthesis, RLAD-style diverse abstractions, problem-archetype concepts) are next-paper territory.

What I'd want to run next in order:
- Fix the feedback leak and re-run math + LCB baselines to get clean pass 2 numbers
- Oracle ensembling (with vs without concepts, best-of)
- Concepts-on-retry only
- Hard LCB subset
