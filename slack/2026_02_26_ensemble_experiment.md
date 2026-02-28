# Slack Reply — Ensemble Experiment

**To:** Matthew Ho
**Re:** Router vs ensemble approach

---

Two paths forward, working on both:

**1. Ensemble experiment (running now)**

With the data we have (2 seeds each for baseline, concepts, concept-retry):

| Ensemble (K=4, pass 1+2) | Math | LCB |
|---|---|---|
| 2 baseline only | 77 | 41 |
| 2 baseline + 2 concept-retry | **86** | **47** |

The question: does mixing in concept runs beat just running more baselines? We already know 2B+2CR = 86 on math. Would 4 baselines reach that? Running s44 and s45 baselines now to find out. My expectation is no — going from 2→4 baseline seeds should hit diminishing returns (the 3rd/4th seeds need to add 9+ new solves over 2-seed oracle of 77, while seeds 1→2 only added 10).

**2. Router**

Pipeline router infrastructure is built (Protocol → Registry → Wiring). Have four implementations ready: none (pass-through), threshold (rule-based), LLM (semantic), NLI (cross-encoder). Haven't evaluated yet — waiting for the ensemble numbers to establish the ceiling before tuning the router.

Will report back when the 4 baseline runs finish (~1hr).
