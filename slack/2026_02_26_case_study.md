Case Study: Concept-Retry Oracle Analysis

Ran full analysis on broader Math L5 dataset (all 7 categories, 200 eval problems). 4 baseline seeds + 2 concept-retry seeds.

5 genuine wins (CR solves, 4 baselines never), 18 genuine harms (4 baselines solve, CR never).

The surprise: "harms" aren't caused by concepts

All 18 harms show False on BOTH CR passes — the model fails pass 1 (no concepts) AND pass 2 (with concepts). These problems are just solved sporadically across baseline seeds (sampling luck), not broken by concept injection. Example:

cmath_11628 (Prealgebra): "A slackrope walker has a rope tied to two 15m high poles 14m apart. Standing 5m from one pole, he is 3m above ground. How long is the rope?" Answer: 28
— Solved by baseline on all 4 seeds' pass 1, but CR pass 1 (same model, different seed trajectory) misses both times. Nothing to do with concepts.

Real concept signal exists

cmath_4005 (Intermediate Algebra): "Let z be a complex number with |z−5−i|=5. Find min of |z−1+2i|²+|z−9−4i|²." Answer: 100
— Failed pass 1 (no concepts) on BOTH CR seeds. Solved pass 2 (with concepts) on BOTH seeds. Concepts selected: coordinate geometry, distance formula, algebraic manipulation. Clear lift.

cmath_9955 (Intermediate Algebra): "Find ordered pairs (a,b) where a is root of x²+ax+b=0 and b is root of x²+ax+b=0." Answer: 3
— Solved pass 1 on both CR seeds, never by any of 4 baselines. Concepts: system of equations, polynomial root analysis, verification through substitution.

Implication: the router framing is wrong. We don't need to gate concepts per-problem (pass 1 is concept-free, so no harm). We need a verifier/ensembler — run both baseline and concept-retry, then pick the right answer. The oracle ceiling is 149/200 (4B+2CR). The question is how close a judge can get.
