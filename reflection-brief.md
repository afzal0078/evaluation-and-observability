# Reflection Brief — Evaluation and Observability Capstone

**Name:** Capstone Learner
**Date:** 2026-10-01

> Ground every answer in your own run. When a question asks for a number, file name, or line, paste
> it from your artifacts — a reviewer should be able to find it. Answers that are correct in the
> abstract but cite nothing do not meet the bar. Keep it short and specific.

---

## 0. Environment

| Field | Value |
|---|---|
| OS & version | Microsoft Windows 11 Enterprise (64-bit AMD64) [Version 10.0.26100] |
| Python version | Python 3.14.6 |
| Date run | 2026-10-01 |
| Ran any system live? (which) | None (Offline runs and replay fixtures used per instructions fallback; verified with offline test suites and replay cassettes) |

---

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 45 passed, 3 skipped (01-policy-pipeline/tests.txt) |
| Routing output file | 01-policy-pipeline/routing_decisions.json |
| auto_approve / human_review / spot_check counts | 4 / 1 / 4 |

**1a. Retry boundary.** From your perturbation run (a required field removed), paste the escalation
record. How many API calls did the system make, and why is retrying a futile case worse than
escalating it?

> In my run on `POL-2025-009`, the extractor made **exactly 1 API call** before halting.
>
> Here is the exact escalation record from my artifact (`01-policy-pipeline/pipeline-run.txt` & `tests/test_us01_retry.py`):
> ```json
> {
>   "kind": "escalation",
>   "policy_id": "POL-2025-009",
>   "field": "endorsements",
>   "category": "missing_source",
>   "detected_pattern": "endorsements_absent",
>   "reason": "Missing source document data for endorsements cannot be resolved by retry."
> }
> ```
>
> **Why retrying a futile case is worse than escalating it:**
> If an endorsement schedule was never attached to the source document in the first place, re-prompting the LLM won't magically materialize it. In the best case, retrying burns API credits and adds seconds of unnecessary latency waiting for the model to repeat itself. In the worst case, repeated error prompts pressure the model into making up believable-sounding endorsements just to satisfy the schema's required field constraint. Halting immediately on `missing_source` and kicking the case to a human queue is the only sound engineering design: it fails fast, prevents hallucinated insurance coverage, and immediately alerts operations that a schedule attachment is physically missing.

**1b. Reading the router.** Pick one `human_review` record from your routing output. Which of the
three signals (confidence, reviewer, integration) sent it to a human? If you had trusted the model's
confidence alone, what would have happened?

> I picked record `POL-2025-010` from `01-policy-pipeline/routing_decisions.json`:
> ```json
> {
>   "policy_id": "POL-2025-010",
>   "policy_type": "home",
>   "decision": "human_review",
>   "reason": "integration_failure=['premium_matches_components_sum']",
>   "fields_below_threshold": [],
>   "reviewer_disagreements": [],
>   "integration_failures": [
>     "premium_matches_components_sum"
>   ],
>   "confidence_summary": {
>     "coverage_limit": 0.95,
>     "deductible": 0.95,
>     "endorsements": 0.95,
>     "exclusions": 0.95,
>     "policy_type": 0.95,
>     "premium_amount": 0.95
>   }
> }
> ```
> **Signal that drove the decision:** The **integration failure** signal (`premium_matches_components_sum`).
>
> **What would have happened if I only trusted confidence:**
> The model rated its own confidence at `0.95` across every single field, including `premium_amount`, easily clearing the 0.90 threshold. If the router had relied solely on the model's self-assessed confidence, `POL-2025-010` would have sailed straight through to `auto_approve`. In reality, the document's stated premium was $2,400.00 while the itemized components actually added up to $2,350.00 (a real $50 accounting discrepancy). Relying on model confidence alone would have pushed an erroneous financial ledger into production. The router requiring `(confidence ∧ reviewer ∧ integration)` all to pass simultaneously is what caught the bug.

**1c. Where the aggregate lies.** Run the calibration snippet. Quote the one cell whose accuracy lags
its confidence, plus the overall figure. What does slicing by `policy_type × field` catch that a
single number hides?

> Here is the exact output from my `01-policy-pipeline/calibration-report.txt`:
> - **The lagging cell:** `umbrella   exclusions      n=2 conf=0.93 acc=0.00 brier=0.865`
> - **The overall figure:** `OVERALL brier=0.291`
>
> **What slicing caught:**
> Looking only at `OVERALL brier=0.291` (and an overall accuracy of ~67%), an engineering team might look at the dashboard and assume the extraction pipeline is performing acceptably. But that macro number is an illusion created by high-volume, easy auto and home policies (`auto / premium_amount` had 100% accuracy and 0.003 Brier). When you slice by `policy_type × field`, you immediately see that on umbrella policy exclusions, the model was **wrong 100% of the time** (`acc=0.00`), yet remained egregiously confident (`conf=0.93`, taking a huge Brier penalty of 0.865). The aggregate metric smoothed out a catastrophic domain failure. Slicing isolates specific areas where the model is confidently incompetent.

---

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 passed (02-mortgage-extraction/tests.txt) |
| Document run | fixtures/documents/appraisal_informal_sqft.txt |
| Classified type | appraisal |

**2a. Two guarantees.** Paste your discrepancy-run output. Tool use already forces valid JSON, yet the
validator still catches a bad sum. Why are these two different guarantees? Name one error each cannot
catch.

> Here is the discrepancy block from `02-mortgage-extraction/discrepancy-run.txt`:
> ```json
> "validation": {
>   "consistent": false,
>   "discrepancies": [
>     {
>       "field": "total_monthly_income",
>       "calculated": 9642.17,
>       "stated": 10892.17,
>       "delta": -1250.0
>     }
>   ]
> }
> ```
> **Why these are two different guarantees:**
> Tool schema enforcement guarantees **syntactic shape**—it forces Claude to produce well-formed JSON where keys match and numbers parse as valid floating-point values. Arithmetic validation guarantees **domain-level mathematical truth**—it checks whether the extracted numbers actually balance against each other.
>
> **Error each cannot catch:**
> - Schema enforcement *cannot* catch internal mathematical contradictions where every individual value is a syntactically valid float. In this run, stated total `$10,892.17` is a completely valid positive float, so the Pydantic schema passed with zero complaints.
> - Arithmetic validation *cannot* catch structural syntax corruption or unparseable payloads (e.g. if the model outputs broken brackets or writes a raw string like `"ten thousand"` into a float field). Arithmetic validation can only run after syntax and typing have already succeeded.

**2b. Refusing to fabricate.** Run on a document missing a field. Paste that field's output. Why null
instead of an invented value? Point to the schema choice that allows it.

> From `02-mortgage-extraction/extract-run.txt` (running `fixtures/documents/income_missing_bonus.txt`):
> ```json
> "income": {
>   "base_monthly": 5673.08,
>   "bonus_monthly": null,
>   "bonus_ytd": null,
>   "commission_monthly": null,
>   "overtime_monthly": null,
>   "other_monthly": null,
>   "stated_monthly_total": null
> }
> ```
> **Why null instead of an invented value:**
> In mortgage underwriting, guessing is dangerous. If the applicant's paystub doesn't mention a bonus, inventing $0 or copying an adjacent figure distorts their debt-to-income (DTI) ratio. Outputting `null` is an explicit, verifiable statement: "this source document does not contain this information."
>
> **The schema choice that allows it:**
> In `mortgage_extractor/schemas.py`, the field is defined with an explicit union with `None`:
> `bonus_monthly: float | None = None`
> Combined with the system prompt directive ("If an item is unstated, return null"), this gives the model a valid, schema-compliant path to decline guessing.

**2c. Normalization.** Quote one field where the source text and extracted value differ in format
("about 2,400 sq ft" → `2400`). Why normalize at extraction time rather than downstream?

> From `02-mortgage-extraction/extract-run.txt`:
> - Source text in document: `"about 2,400 sq ft"`
> - Extracted JSON property: `"gross_living_area_sqft": 2400`
>
> **Why normalize at extraction time rather than downstream:**
> At extraction time, the LLM has access to the full document context and understands natural language modifiers ("about", "approx.", "sq ft", "SF"). Post-hoc downstream regexes or string parsers are notoriously brittle against variations in human drafting. Normalizing at the extraction interface turns messy unstructured prose into an integer right away, keeping downstream underwriting services simple, type-safe, and free of fragile text parsing.

---

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 passed (03-supply-chain/tests.txt) |
| Briefing file | 03-supply-chain/briefing.md |
| Section the conflict landed in | ## Contested |

**3a. Annotate, don't arbitrate.** Quote one conflicting-metric pair from your briefing — both values,
sources, dates. Give one way a reader is better served by the preserved conflict than by a single
reconciled number.

> From `03-supply-chain/briefing.md` in the `## Contested` section:
> ```markdown
> ### on_time_delivery_rate  _[2 sources, conflicting]_  ⚠️ ESCALATE
> - escalation: high-impact metric is contested across sources
> - Reported values by source:
>     - 95.0 percent — supplier_audit (as of 2026-04-10)
>     - 78.0 percent — logistics (as of 2026-04-05)
>
> | Metric | Value | As of | Source |
> | --- | --- | --- | --- |
> | on_time_delivery_rate | 78.0 percent | 2026-04-05 | logistics |
> ```
> **Why the reader is better served:**
> If my system had averaged these two numbers to 86.5%, or silently chosen the supplier audit because it was newer, it would have smoothed over the exact problem an investigator needs to see. The supplier gave itself a glowing 95% on-time rate during their self-audit, but internal logistics logs show only 78%. That discrepancy points directly to either supplier dishonesty or an internal intake bottleneck. Preserving both claims alongside their sources gives the procurement team actionable intelligence to investigate, instead of a fabricated average.

**3b. Source goes dark.** Run with `--simulate-timeout`. Paste the part of the briefing showing the
failed source. How is "unreachable" handled differently from "nothing to report," and why does the run
still finish?

> From `03-supply-chain/timeout-run.txt`:
> ```markdown
> > Sources unavailable: logistics unavailable (timeout)
> ...
> ## Incomplete
> ### late_shipment_count  _[missing source: timeout reading logistics]_
> - missing source: timeout reading logistics
> ```
> **How 'unreachable' differs from 'nothing to report':**
> "Nothing to report" means the data source was contacted successfully, checked the database, and confirmed zero issues. "Unreachable" means the connection timed out and we have no idea what happened. If you treat an unreachable source as "nothing to report," a downed logistics server looks like a supplier with zero late shipments—a dangerous false negative.
>
> **Why the run still finishes:**
> The coordinator encloses individual reader operations within fault-tolerant exception boundaries. When logistics raises a timeout exception, the coordinator logs the outage, marks dependent metrics under `## Incomplete`, and continues processing the surviving sources (`supplier_audit`, `internal_quality`, `industry_news`) to yield an actionable, partially degraded briefing without crashing.

**3c. Dates as a guardrail.** Quote two claims about the same supplier with different dates. How does
requiring a date stop a time difference from reading as a contradiction?

> From `03-supply-chain/briefing.md`:
> - Claim 1: `defect_rate_ppm`: `190.0 ppm — internal_quality (as of 2026-04-08)`
> - Claim 2: `defect_rate_ppm`: `180.0 ppm — supplier_audit (as of 2026-04-10)`
> (And news events: financial distress on 2026-03-09 vs port strike on 2026-03-17).
>
> **How requiring a date stops time difference from reading as a contradiction:**
> Real-world operational metrics naturally evolve over time. Without timestamps, 190 ppm and 180 ppm look like two observers disagreeing on the same static fact. Attaching ISO-8601 dates establishes a temporal sequence (showing defect rates improving by 10 ppm from April 8 to April 10), preventing chronological progression from being misinterpreted as conflicting data.

---

## 4. Synthesis

**4a. One principle.** Name the single moment in your runs (system + artifact) where *evaluate the
output, don't trust the model's word* most clearly caught something a trusting design would have
shipped.

> For me, that moment occurred in **System 2 (Mortgage Extraction)**, captured in `02-mortgage-extraction/discrepancy-run.txt`.
> Claude parsed `fixtures/documents/income_sum_mismatch.txt` cleanly. It returned base salary, bonus, commission, overtime, and a stated total of $10,892.17. Every field matched the schema, every number was a positive float, and the model showed zero signs of distress. A trusting system would have packaged up that JSON and committed it straight to the loan origination database. But the deterministic Python validator ran the actual math: the line items totaled $9,642.17, revealing a -$1,250.00 shortfall against the stated total. Evaluating output correctness arithmetically caught an underwriting error that schema validation alone was incapable of detecting.

**4b. Confidence ≠ correctness.** Pick the system where this mattered most, and explain why using
something you observed.

> This principle mattered most in **System 1 (Policy Pipeline)**, as demonstrated in `01-policy-pipeline/calibration-report.txt`.
> For umbrella policy exclusions, the model self-reported a high mean confidence of **0.93 (93%)**, yet observed accuracy was **0.00 (0%)** (`brier=0.865`). If our routing logic had simply asked Claude "how confident are you?" and auto-approved anything above 0.90, 100% of these defective umbrella extractions would have skipped human review with missing or hallucinated exclusions. LLMs are notoriously uncalibrated on complex legal clauses. Decoupling routing from self-reported confidence—by adding an independent reviewer pass, deterministic business rules, and stratified spot checking—is essential to keep overconfident models from poisoning production data.

**4c. Apply it.** Describe a real workflow where an LLM pulls structured results from messy input.
Which pattern — validated retry with escalation, independent review with deterministic routing, or
provenance-preserving conflict annotation — would you reach for first, and what would you instrument
to know when it broke?

> **Real-world workflow:** Ingesting multimodal medical lab reports and clinical referral letters into an EHR patient summary.
>
> **Pattern I would reach for first:**
> I would start with **validated retry paired with deterministic multi-signal routing**:
> 1. *Strict fail-fast retry:* If numerical lab values or units fail sanity bounds (e.g. negative blood pressure or potassium outside biological viability), retry once with the error flagged. But if critical identifiers (patient DOB or MRN) are completely absent from the scanned referral letter, halt immediately—never retry and tempt the model into guessing a patient's identity.
> 2. *Deterministic review gate:* Run a secondary lightweight reviewer model to cross-check extracted medication dosages against the text. If the reviewer disagrees, or if programmatic unit validation fails, send the document straight to human clinical triage.
>
> **What I would instrument to know when it broke:**
> 1. **Sliced Calibration Tracking:** Track accuracy and Brier score sliced by `clinic_source × test_type`. When a new diagnostic lab changes its report formatting, their specific slice will immediately degrade in Brier score, tipping us off before aggregate clinic numbers move.
> 2. **Routing Tier Volume Drift:** Set an alert on the ratio of `auto_approve` vs `human_review`. A sudden 15% spike in human reviews usually means an upstream scanner is degrading or a clinic updated their PDF template.
> 3. **Spot-Check Inversion Audit:** Have medical QA review 5% of all auto-approved records. If the audit finds even a 0.2% error rate in auto-approved dosages, the router automatically dials down the auto-approve threshold.
