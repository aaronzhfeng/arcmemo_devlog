# Devlog 24 — Config Validation Gap: pipeline vs components

**Date**: 2026-02-23
**Prerequisite**: Devlog 22 (audit), Devlog 23 (coupling fixes)

## The Gap

The config has two sections that must agree but are not validated against each other:

```yaml
pipeline:
  inference_engine: python_transform_retry    # selects the CLASS

components:
  inference_engine:                           # provides CONSTRUCTOR ARGS
    model: qwen/qwen3-30b
    gen_cfg: {n: 1, temperature: 0.0}
    prompt_options:                            # only valid for python_transform_retry
      include_hint: true
```

`wiring.py` does:
```python
cls = INFERENCE_ENGINES[pipe["inference_engine"]]   # lookup class by name
kwargs = comp_cfg.get("inference_engine", {})        # grab kwargs dict
return cls(**kwargs)                                 # pray they match
```

There is no check that `kwargs` are valid for `cls`.

---

## How This Fails

### Silent absorption — wrong params ignored

```yaml
pipeline:
  inference_engine: math_ps_solve

components:
  inference_engine:
    model: qwen/qwen3-30b
    prompt_options:            # <-- math_ps_solve doesn't have this
      include_hint: true       #     but accepts **kwargs, so it vanishes
```

`MathPsSolveInferenceEngine.__init__` has `**kwargs` to absorb unknown keys.
The user thinks `include_hint` is active. It's not. No error, no warning.

### Missing required params — cryptic TypeError

```yaml
pipeline:
  benchmark: livecodebench

components:
  benchmark:
    data_root: data/arc_agi/training    # <-- ARC data path, wrong for LCB
    limit: 5
```

This technically "works" (both constructors accept `data_root`), but loads ARC files
as LCB problems, producing garbage at evaluation time.

### Stale params after pipeline swap

User changes `pipeline.inference_engine` from `python_transform_retry` to
`math_ps_solve` but forgets to update `components.inference_engine`. The old
`prompt_options`, `system_prompt_key`, etc. are silently absorbed or cause
`TypeError` depending on whether the new class has `**kwargs`.

---

## Scope of the Problem

Every component has this gap. The 10 `pipeline.*` keys each have a corresponding
`components.*` block:

| pipeline key | Classes | Param overlap | Risk |
|-------------|---------|---------------|------|
| `inference_engine` | 3 classes, different params | `prompt_options` ARC-only, `**kwargs` absorbs rest | HIGH |
| `benchmark` | 3 classes, different params | `data_root` shared, `split`/`types`/`levels` domain-specific | MEDIUM |
| `evaluator` | 3 classes | `timeout_s` shared, `require_all_tests` ARC-only | LOW |
| `feedback_engine` | 3 classes | `positive_msg`/`negative_msg` shared | LOW |
| `memory_builder` | 3 classes | completely different params | MEDIUM |
| `memory_retriever` | 4 classes | completely different params | HIGH |
| `provider` | multiple profiles | `profile_name` routing, params vary | MEDIUM |
| `task_adapter` | 3 classes | `task_name` only | LOW |
| `trajectory_policy` | 1 class | `retry_paths` only | LOW |
| `artifact_sink` | 1 class | none | LOW |

The highest-risk components are `inference_engine` and `memory_retriever` because:
- They have the most classes with the most divergent params
- They use `**kwargs` to absorb unknown params silently
- Config inheritance (`_base_:`) makes it easy to carry stale params

---

## Fix Options

### Option A: Constructor-level validation (minimal)

Remove `**kwargs` from constructors. Let `TypeError` propagate on unknown params.

**Pros**: Zero new infrastructure. Immediate feedback on wrong params.
**Cons**: Breaks config inheritance where base sets params the child class doesn't
use. Every class must explicitly list every param it accepts.

```python
# Before:
class MathPsSolveInferenceEngine:
    def __init__(self, model="", gen_cfg=None, **kwargs):  # absorbs anything
        ...

# After:
class MathPsSolveInferenceEngine:
    def __init__(self, model="", gen_cfg=None,             # no **kwargs
                 include_reselected_lessons=False, ...):
        ...
```

### Option B: Schema declaration per class

Each class declares its accepted params. Wiring validates before construction.

**Pros**: Clear documentation of what each class accepts. Can generate help text.
**Cons**: Schema must stay in sync with constructor. Duplication.

```python
class MathPsSolveInferenceEngine:
    ACCEPTED_PARAMS = {"model", "gen_cfg", "include_reselected_lessons",
                       "error_feedback", "num_feedback_passes", "include_past_outcomes"}
```

```python
# In wiring.py:
def _build_component(registry, key, cfg):
    cls = registry[key]
    accepted = getattr(cls, "ACCEPTED_PARAMS", None)
    if accepted is not None:
        unknown = set(cfg.keys()) - accepted
        if unknown:
            raise ConfigurationError(
                f"Unknown params for '{key}': {unknown}. Accepted: {sorted(accepted)}"
            )
    return cls(**cfg)
```

### Option C: Introspect constructor signature (automatic)

Use `inspect.signature(cls.__init__)` to extract accepted params. Warn or error on
unknown keys.

**Pros**: No manual declaration needed. Always in sync with constructor.
**Cons**: `**kwargs` defeats it. Must remove `**kwargs` first (back to option A).

```python
import inspect

def _build_component(registry, key, cfg):
    cls = registry[key]
    sig = inspect.signature(cls.__init__)
    has_var_keyword = any(
        p.kind == inspect.Parameter.VAR_KEYWORD for p in sig.parameters.values()
    )
    if not has_var_keyword:
        accepted = {name for name, p in sig.parameters.items()
                    if name != "self" and p.kind in (
                        inspect.Parameter.POSITIONAL_OR_KEYWORD,
                        inspect.Parameter.KEYWORD_ONLY,
                    )}
        unknown = set(cfg.keys()) - accepted
        if unknown:
            raise ConfigurationError(
                f"Unknown params for '{key}': {unknown}. Accepted: {sorted(accepted)}"
            )
    return cls(**cfg)
```

### Option D: Warn but don't error (gradual migration)

Same as option C but log warnings instead of raising. Lets existing configs work
while flagging issues.

---

## Recommended Approach

**Option A + C combined**:

1. Remove `**kwargs` from all component constructors. Each class explicitly lists
   every param it accepts (with defaults for params it doesn't use).
2. Add `inspect.signature` check in `_build_component`. Since no class has `**kwargs`,
   this catches all unknown params automatically.
3. Unknown params -> `ConfigurationError` with a clear message listing accepted params.

This is the cleanest path because:
- No manual schema maintenance (unlike option B)
- Constructor IS the schema — single source of truth
- Removing `**kwargs` forces each class to be explicit about what it accepts
- Config inheritance still works if child class lists all inherited params with defaults

### Migration path

For each class that currently uses `**kwargs`:
1. Check what keys are actually passed via configs
2. Add those as explicit params with defaults
3. Remove `**kwargs`
4. Run all tests + smoke tests to verify

Classes currently using `**kwargs`:
- `MathPsSolveInferenceEngine` (absorbs `prompt_options` from shared config)
- `LcbSolveInferenceEngine` (same)
- `MathPsExecutionEvaluator` (absorbs `require_all_tests` from shared config)
- `LcbExecutionEvaluator` (absorbs `require_all_tests` from shared config)

---

## Scope

This is a config-layer fix. No prompt changes, no logic changes. The guiding principle
("structure changes, logic doesn't") applies. Parity tests must pass before and after.
