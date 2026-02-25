Sorry for the long gap. I had a server shutdown that wiped the codebase and all the experiment artifacts from the runs I reported last time. Spent the first few days rebuilding from the last push.

Things implemented / tested so far:

Switched to qwen-2.5-7b (3b active) as solver with stronger selector models. Rebuilt full concept pipeline: extraction from 200 solved problems, compression, offline selection (4-5 concepts per problem). Math 100-problem L5 subset: concepts 65% vs 63% baseline. LCB 100-problem subset: concepts 33% vs 34% baseline. Variance is high though, 35/100 math problems change outcome between identical baseline runs, so +2% is within noise.

Ran ablation on frequency filtering, cues-only rendering, selection-confidence routing. Nothing reliably beat baseline on either benchmark. Hints help and hurt in roughly equal measure, the harm is semantic — wrong algorithm suggestions, overcomplicated code.

Built an LLM router as new stage between selection and inference. One call per problem, asks which selected concepts are relevant, filters the rest. LCB (qwen3-coder-30b): baseline 33, concepts 33, routed 34, kept 70% of concepts. Math (qwen-2.5-7b as router): baseline 58, concepts 64, routed 57, router too aggressive at 43% keep rate, blanked 22 problems. The 7b model isn't good enough as a router. Ran out of API credit before retrying with 30b.

Next: retry math routing with 30b, try NLI cross-encoder router (local, no API cost, tunable threshold), per-problem case study on router help vs hurt.
