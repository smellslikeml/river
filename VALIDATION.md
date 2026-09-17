# WSTD — validation report

Agent-authored artifact (Remyx Recommendation run, brief mode). Per River's `AGENTS.md`,
the eventual human contributor should rewrite the docstring, PR prose, and commit message
in their own words and disclose agent authorship via a `Co-authored-by:` trailer.

## What was implemented

`river/drift/binary/wstd.py` — `WSTD`, a clean-room implementation of the Wilcoxon Rank
Sum Test drift detector (Barros, Hidalgo & Cabral, *Wilcoxon Rank Sum Test Drift
Detector*, Neurocomputing 275, 2018, doi:10.1016/j.neucom.2017.10.051), subclassing
`river.base.BinaryDriftAndWarningDetector` (the same base as DDM/EDDM/FHDDM/HDDM*),
registered in `river/drift/binary/__init__.py`. `river/drift/__init__.py` does not
re-export individual binary detectors, so it is intentionally untouched.

The paper (paywalled) was not directly readable from this environment. The algorithm was
implemented from standard literature descriptions of WSTD/STEPD. The MOA (GPL-3.0)
source was **not** read, copied, ported, or paraphrased; MOA is cited here as a
behavioral reference only.

Because the input is binary, the pooled rank sums are computed in closed form from the
per-window counts of ones (mid-ranks for ties), so `update` is O(1) — no sorting, no
scipy call in the hot loop. The two-sided p-value is `erfc(|z| / sqrt(2))` in pure
`math`.

## Provenance of constants

| Constant | Value | Source |
|---|---|---|
| recent window size `w` | 30 | Confirmed: secondary source quoting the paper's setup ("w = 30"), matches STEPD |
| `alpha_warning` | 0.05 | Confirmed: secondary sources quoting WSTD's thresholds |
| `alpha_drift` | 0.003 | Confirmed: secondary sources quoting WSTD's thresholds |
| z-statistic | `z = (R − μ_R)/σ_R`, `μ_R = n₁(N+1)/2`, `σ_R = √(n₁n₂(N+1)/12)` | Confirmed formula (normal approximation, **no** tie-correction term in the documented formula) |
| older window size | 120 | **Assumed** — STEPD's blueprint value (WSTD "closely resembles STEPD"). The WSTD-specific bound was not recoverable from accessible sources; benchmark papers tuned it (grids of 1000–4000), and it is exposed as a constructor parameter |
| `min_instances` guard | 30 obs in the older window (recent window full) before testing | **Assumed** — STEPD's minimum-instances convention; the paper's exact guard was not recoverable |
| two-sided p-value | yes | **Assumed** — per the design brief; consistent with taking the rank sum of the smaller sub-window |
| ties | mid-ranks | Forced by binary data (massive ties); the variance keeps the paper's uncorrected form, which overestimates variance under ties and is therefore conservative |

## Proven here (CI-runnable, all executed in this environment)

- `pytest river/drift/ tests/drift/` — 57 passed (includes the new `tests/drift/test_drift_detectors.py::test_wstd*` and the WSTD doctest).
- Behavioral (deterministic, seeded streams):
  - abrupt drift 0.2 → 0.8 (`data_stream_2`): exactly one detection at index 1028, warning at 1017 (warning precedes drift);
  - stationary p=0.2, stationary uniform 0/1, all-ones, all-zeros: **zero false alarms**;
  - two-sidedness: transient 0→1→0 blip stream detects both the rise (210) and the fall (273);
  - warm-up guard: no test (p-value stays 1.0) before the older window reaches `min_instances`;
  - `ValueError` coverage for invalid alphas, window sizes, and `min_instances > older_window_size`.
- Numerical: `WSTD.p_value` matches `scipy.stats.ranksums` (the same tie-uncorrected
  normal approximation, mid-ranks) to `rel_tol=1e-12` at every full-window step over a
  400-sample stream.
- Performance: ~840k `update`/s (DDM baseline ~840k on the same machine) — O(1) per
  update, far above River's 5000 records/s bar.
- Doctest in the class docstring runs and passes.

## Conformance-check caveat (pre-existing, not WSTD-specific)

`river.checks.check_estimator` currently raises `AttributeError` on **every** drift
detector in the repo (verified against `DDM`, `ADWIN`, `KSWIN`, `PageHinkley`, `NoDrift`)
because drift detectors inherit from `base.Base` (not `base.Estimator`), and
`_unit_test_skips` / `_tags` are defined on `Estimator`. Drift detectors are likewise
excluded from `tests/test_estimators.py`'s enumeration (it filters
`issubclass(obj, base.Estimator)`). Fixing that harness gap would require touching
`river/base/` or `river/checks/`, which is outside this brief's allowed scope.

`test_wstd_check_estimator` instead runs the applicable conformance checks directly from
the pre-existing `river.checks.common` module (repr, str, doc, clone idempotence &
independence of signature, default/mutable params, pickling round-trip, repr-roundtrips-
clone, clone-with-new-params, `_get_params` vs signature) — all pass.

## Deferred (honestly not verified)

- Exact numerical parity with the MOA reference implementation and the paper's benchmark
  tables (MEAs, NML, latency). MOA is GPL-3.0 and was not run or read; the paper was not
  directly accessible. The CI bar proven here is "conformant + detects synthetic abrupt
  drift without false-alarming on stationary streams", **not** paper parity.
- Real-dataset drift benchmarks (River marks those `slow`/`datasets` in CI anyway).

## Environment notes

- `uv`, `prek`, `ruff`, `mypy`, and `sklearn` are not installed in this run's
  environment, so `make format` / `uv run mypy` / sklearn-dependent tests could not be
  executed here. Formatting follows the repo's ruff config by inspection (line length
  100, import ordering, `from __future__ import annotations`). `river.drift.*` is in the
  relaxed-mypy override list; the new file is fully annotated regardless.
- On the first combined pytest run, `river/drift/retrain.py`'s doctest (Elec2 dataset,
  unrelated to WSTD) failed once during the dataset's first fetch; 6/6 subsequent runs of
  the identical command pass. Pre-existing environment flake, not touched by this change.

## Self-review summary

- Call site targeted: `river/drift/binary/` (new `wstd.py` + registration in its
  `__init__.py`); tests in the pre-existing `tests/drift/test_drift_detectors.py`, which
  imports `river.drift`, `river.checks`, and `scipy.stats` — no self-only testing.
- Clean-room: implemented from the paper's algorithm description as relayed by
  independent secondary sources; no MOA code was viewed or copied.
- Assumed defaults (older window 120, guard semantics, two-sided conversion) are
  documented above and exposed as tunables.
- Intentionally out of scope: the drift-detector conformance-harness gap in
  `river/base` + `river/checks`, docs/API listings, benchmarks (CodSpeed), other
  detectors.
