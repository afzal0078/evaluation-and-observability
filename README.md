# Claude AI Engineer — Evaluation and Observability Capstone Submission

This repository contains the complete, verified evidence pack, reflection brief, perturbation log, and execution records for the **Evaluation and Observability** project in the Claude AI Engineer program.

---

## 📂 Repository & Submission Layout

```
.
├── README.md                      # Submission overview & navigation guide
├── reflection-brief.md            # Completed reflection brief (grounded in terminal artifacts)
├── perturbation-log.md            # Three controlled edge-case experiments with run contrasts
├── environment.txt                # Environment details (Python 3.14.6, Windows 11 AMD64, tool versions)
├── capstone-submission.zip        # Bundled submission archive
│
├── 01-policy-pipeline/            # System 1: Validated, Routed Insurance Pipeline
│   ├── tests.txt                  # Full test-suite output (45 passed, 3 skipped)
│   ├── static-checks.txt          # mypy (0 errors in 11 files) + ruff (all passed)
│   ├── routing-tests.txt          # Verbose routing test run (accepted no-key fallback: 9 passed)
│   ├── calibration-report.txt     # Sliced (policy_type × field) calibration report
│   └── perturbation-run.txt       # Perturbation run: excised premium halts immediately (1 API call)
│
├── 02-mortgage-extraction/        # System 2: Schema-Enforced Two-Pass Mortgage Extraction
│   ├── tests.txt                  # Full test-suite output (25 passed)
│   ├── static-checks.txt          # mypy (0 errors in 11 files) + ruff (all passed)
│   ├── extract-run.txt            # Clean extraction & normalization ("about 2,400 sq ft" → 2400)
│   ├── discrepancy-run.txt        # Arithmetic consistency validator catching $1,250 mismatch
│   └── perturbation-run.txt       # Perturbation run: learner-edited stated total ($12,500.00 vs $9,642.17)
│
└── 03-supply-chain/               # System 3: Provenance-Preserving Multi-Source Synthesis
    ├── tests.txt                  # Full test-suite output (34 passed)
    ├── static-checks.txt          # mypy (0 errors in 8 files) + ruff (all passed)
    ├── investigation-run.txt      # Nominal investigation run output
    ├── briefing.md                # 3-section briefing (Well-Established, Contested, Incomplete)
    └── timeout-run.txt            # Fault-isolation run under simulated source timeout
```

---

## 📊 Summary of Systems & Evidence

### System 1: Validated, Routed Insurance Policy Extraction Pipeline
- **Tests & Static Checks:** **45 passed, 3 skipped**; `mypy` (11 files) and `ruff` pass with **0 errors**.
- **Key Artifacts:**
  - [`01-policy-pipeline/routing-tests.txt`](01-policy-pipeline/routing-tests.txt): Verbose run of `tests/test_us04_routing.py` (9 passed), demonstrating deterministic HITL routing rules (`auto_approve`, `human_review`, and stratified spot-checks).
  - [`01-policy-pipeline/calibration-report.txt`](01-policy-pipeline/calibration-report.txt): Slicing by `policy_type × field` reveals that while overall Brier is moderate (`0.291`), the model was wrong 100% of the time on `umbrella / exclusions` (`acc=0.00`) despite claiming `0.93` confidence (`brier=0.865`).
  - [`01-policy-pipeline/perturbation-run.txt`](01-policy-pipeline/perturbation-run.txt): Excising the premium section from `POL-2025-001.txt` triggers an immediate `missing_source` escalation (`premium_amount_absent`), halting after **exactly 1 API call**.
  - **Deterministic Routing Trace:** In `test_ac_04_02_integration_failure_routes_to_human_review`, the integration check failure (`premium_matches_components_sum` with a $50 discrepancy) overrides high extractor confidence (0.95) and forces human review.

### System 2: Schema-Enforced Two-Pass Mortgage Extraction
- **Tests & Static Checks:** **25 passed**; `mypy` (11 files) and `ruff` pass with **0 errors**.
- **Key Artifacts:**
  - [`02-mortgage-extraction/extract-run.txt`](02-mortgage-extraction/extract-run.txt): Demonstrates two-pass classification-then-extraction with unit normalization (`"about 2,400 sq ft"` → `2400`) and missing field handling (`bonus_monthly: null` via nullable schema `float | None`).
  - [`02-mortgage-extraction/discrepancy-run.txt`](02-mortgage-extraction/discrepancy-run.txt): The consistency validator calculates line items ($9,642.17) and catches the -$1,250.00 discrepancy against stated total ($10,892.17), proving that schema validity does not equal mathematical correctness.
  - [`02-mortgage-extraction/perturbation-run.txt`](02-mortgage-extraction/perturbation-run.txt): Learner edits stated monthly total from $10,892.17 to $12,500.00, producing a new delta of -$2,857.83 alongside the original -$1,250.00 delta.

### System 3: Provenance-Preserving Multi-Source Synthesis
- **Tests & Static Checks:** **34 passed**; `mypy` (**8 files**) and `ruff` pass with **0 errors**.
- **Key Artifacts:**
  - [`03-supply-chain/briefing.md`](03-supply-chain/briefing.md): Multi-source synthesis briefing structured into `## Well-Established`, `## Contested`, and `## Incomplete`.
  - **Annotate, Don't Arbitrate:** Preserves the conflicting `on_time_delivery_rate` with full attribution: `95.0%` (supplier audit, 2026-04-10) vs `78.0%` (internal logistics, 2026-04-05) rather than averaging them.
  - [`03-supply-chain/timeout-run.txt`](03-supply-chain/timeout-run.txt): Running with `--simulate-timeout` verifies fault isolation; the coordinator marks logistics as unavailable and dependent metrics under `## Incomplete`, completing the run without crashing.

---

## 📑 Core Evaluated Documents

- **[Reflection Brief](reflection-brief.md):** Complete answers to all prompts (0 through 4c), grounded in direct citations from the evidence pack runs and an analysis of reliability tradeoffs.
- **[Perturbation Log](perturbation-log.md):** Records controlled input perturbations for each system, contrasting observed error handling with unperturbed nominal runs.
- **[Environment Record](environment.txt):** Full technical environment documentation (Python 3.14.6, Windows 11 Enterprise AMD64, tool versions).
