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
│   ├── pipeline-run.txt           # Pipeline routing terminal capture & offline verification
│   ├── routing_decisions.json     # Generated routing decisions with stratified spot-checks
│   ├── calibration-report.txt     # Sliced (policy_type × field) calibration report
│   └── screenshots/
│       └── pipeline_routing_terminal.svg
│
├── 02-mortgage-extraction/        # System 2: Schema-Enforced Two-Pass Mortgage Extraction
│   ├── tests.txt                  # Full test-suite output (25 passed)
│   ├── static-checks.txt          # mypy (0 errors in 11 files) + ruff (all passed)
│   ├── extract-run.txt            # Clean extraction & normalization ("about 2,400 sq ft" → 2400)
│   ├── discrepancy-run.txt        # Arithmetic consistency validator catching $1,250 mismatch
│   └── screenshots/
│       └── mortgage_discrepancy_terminal.svg
│
└── 03-supply-chain/               # System 3: Provenance-Preserving Multi-Source Synthesis
    ├── tests.txt                  # Full test-suite output (34 passed)
    ├── static-checks.txt          # mypy (0 errors in 7 files) + ruff (all passed)
    ├── investigation-run.txt      # Nominal investigation run output
    ├── briefing.md                # 3-section briefing (Well-Established, Contested, Incomplete)
    ├── timeout-run.txt            # Fault-isolation run under simulated source timeout
    └── screenshots/
        └── supply_chain_briefing_terminal.svg
```

---

## 📊 Summary of Systems & Evidence

### System 1: Validated, Routed Insurance Policy Extraction Pipeline
- **Command:** `policy-extractor pipeline data/policies/ --routing-out routing_decisions.json --seed 42`
- **Tests & Static Checks:** **45 passed, 3 skipped**; `mypy` and `ruff` pass with **0 errors**.
- **Key Artifacts:**
  - [`01-policy-pipeline/routing_decisions.json`](01-policy-pipeline/routing_decisions.json): Documents 10 policies processed with deterministic HITL routing and stratified drift-detection sampling (`auto_approve`: 4, `human_review`: 1, `spot_check`: 4, `escalations`: 1).
  - [`01-policy-pipeline/calibration-report.txt`](01-policy-pipeline/calibration-report.txt): Slicing by `policy_type × field` reveals that while overall Brier is moderate (`0.291`), the model was wrong 100% of the time on `umbrella / exclusions` (`acc=0.00`) despite claiming `0.93` confidence (`brier=0.865`).
  - **Deterministic Routing:** Policy `POL-2025-010` had high confidence (0.95), but was routed to human review due to the integration check (`premium_matches_components_sum` failed by $50).

### System 2: Schema-Enforced Two-Pass Mortgage Extraction
- **Command:** `mortgage-extract <document> --mode replay`
- **Tests & Static Checks:** **25 passed**; `mypy` and `ruff` pass with **0 errors**.
- **Key Artifacts:**
  - [`02-mortgage-extraction/extract-run.txt`](02-mortgage-extraction/extract-run.txt): Demonstrates two-pass classification-then-extraction with unit normalization (`"about 2,400 sq ft"` → `2400`) and missing field handling (`bonus_monthly: null` via nullable schema `float | None`).
  - [`02-mortgage-extraction/discrepancy-run.txt`](02-mortgage-extraction/discrepancy-run.txt): The consistency validator calculates line items ($9,642.17) and catches the -$1,250.00 discrepancy against stated total ($10,892.17), proving that schema validity does not equal mathematical correctness.

### System 3: Provenance-Preserving Multi-Source Synthesis
- **Command:** `supply-chain-investigate meridian --offline`
- **Tests & Static Checks:** **34 passed**; `mypy` and `ruff` pass with **0 errors**.
- **Key Artifacts:**
  - [`03-supply-chain/briefing.md`](03-supply-chain/briefing.md): Multi-source synthesis briefing structured into `## Well-Established`, `## Contested`, and `## Incomplete`.
  - **Annotate, Don't Arbitrate:** Preserves the conflicting `on_time_delivery_rate` with full attribution: `95.0%` (supplier audit, 2026-04-10) vs `78.0%` (internal logistics, 2026-04-05) rather than averaging them.
  - [`03-supply-chain/timeout-run.txt`](03-supply-chain/timeout-run.txt): Running with `--simulate-timeout` verifies fault isolation; the coordinator marks logistics as unavailable and dependent metrics under `## Incomplete`, completing the run without crashing.

---

## 📑 Core Evaluated Documents

- **[Reflection Brief](reflection-brief.md):** Complete answers to all prompts (0 through 4c), grounded in direct citations from the evidence pack runs and an analysis of reliability tradeoffs.
- **[Perturbation Log](perturbation-log.md):** Records controlled input perturbations for each system, contrasting observed error handling with unperturbed nominal runs.
- **[Environment Record](environment.txt):** Full technical environment documentation (Python 3.14.6, Windows 11 Enterprise AMD64, tool versions).
