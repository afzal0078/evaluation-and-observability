# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run
it, and record what actually happened. See the starters in the Instructions, or design your own (your
own experiment earns more credit).

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):**
  Created a copy of `data/policies/POL-2025-001.txt` as `data/policies/POL-2025-001_no_premium.txt`, and deliberately excised the entire premium section (lines 15-17: `PREMIUM SUMMARY`, `Total Policy Premium ........................  $ 1,847.62`, and `Payment Plan: Semi-Annual`), leaving the mandatory `premium_amount` field unstated in the source text.
- **Command I ran:**
  `python -m scripts.run_perturbation` (in `04-hitl-routing/solution`, capturing output to `01-policy-pipeline/perturbation-run.txt`)
- **What I predicted:**
  Per the system prompt instruction to return null for unstated fields, the extractor will return `premium_amount: null`. The validator will flag this as a `missing_source` failure (`category="missing_source"`, `detected_pattern="premium_amount_absent"`). Because re-prompting cannot recover information physically missing from the source document, the retry engine should recognize retry as futile, halt immediately after exactly 1 API call (`client.call_count == 1`), and return a `RetryFutileEscalation` to human review without wasting retry attempts.
- **What actually happened (paste the key output line):**
  ```text
  Policy ID: POL-2025-001-PERTURBED
  Result Type: RetryFutileEscalation
  Field: premium_amount
  Category: missing_source
  Detected Pattern: premium_amount_absent
  Reason: Field 'premium_amount' returned null — the source document does not contain this information. Retry is futile; escalate to human review.
  API Call Count: 1
  ```
- **How this differs from the unperturbed run:**
  On the unperturbed source document (`data/policies/POL-2025-001.txt`), the total policy premium of $1,847.62 is present and valid; the extraction succeeds on attempt 0 (`retry_count=0`) with `PolicyExtraction(policy_id='POL-2025-001', premium_amount=1847.62)`. On the perturbed document lacking the premium line, the validator immediately halts on attempt 0, making exactly 1 API call and routing to human review instead of re-prompting the model.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):**
  Created a copy of `fixtures/documents/income_sum_mismatch.txt` as `fixtures/documents/income_sum_mismatch_perturbed.txt` and manually edited the stated monthly earnings line from `TOTAL MONTHLY EARNINGS 10,892.17` to `TOTAL MONTHLY EARNINGS 12,500.00`, expanding the arithmetic mismatch against the itemized income sum ($9,642.17).
- **Command I ran:**
  `python -m scripts.run_perturbation` (in `04-validate-mathematical-consistency/solution`, capturing output to `02-mortgage-extraction/perturbation-run.txt`)
- **What I predicted:**
  The schema and JSON typing checks will still pass completely (both component numbers and stated total are valid floats). However, the mathematical consistency validator will compute the sum of components ($5,416.67 + $2,140.00 + $1,250.00 + $385.50 + $450.00 = $9,642.17) and compare it against the newly edited stated total ($12,500.00), flagging `consistent: false` with a new delta of -$2,857.83 (contrasting with the original -$1,250.00 delta).
- **What actually happened (paste the key output line):**
  ```json
  [ORIGINAL BUNDLED FIXTURE RUN]
  {
    "document": "fixtures/documents/income_sum_mismatch.txt",
    "stated_monthly_total": 10892.17,
    "calculated_monthly_total": 9642.17,
    "consistent": false,
    "discrepancy": {
      "field": "total_monthly_income",
      "calculated": 9642.17,
      "stated": 10892.17,
      "delta": -1250.0
    }
  }

  [PERTURBED FIXTURE RUN (LEARNER-EDITED STATED TOTAL: $12,500.00)]
  {
    "document": "fixtures/documents/income_sum_mismatch_perturbed.txt",
    "stated_monthly_total": 12500.0,
    "calculated_monthly_total": 9642.17,
    "consistent": false,
    "discrepancy": {
      "field": "total_monthly_income",
      "calculated": 9642.17,
      "stated": 12500.0,
      "delta": -2857.83
    }
  }
  ```
- **How this differs from the unperturbed run:**
  The original bundled paystub had a stated total of $10,892.17, creating a delta of -$1,250.00 (the bonus amount was double-counted). In my perturbed fixture with the stated total manually edited to $12,500.00, the validator flags the enlarged discrepancy with `delta = -2857.83`. Both contrast with an unperturbed mathematically consistent document (such as `appraisal_informal_sqft.txt`), which returns `consistent: true` and `discrepancies: []`.

---

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):**
  Added the `--simulate-timeout` flag to the CLI command, simulating a network timeout failure when attempting to query the `logistics` data reader (`data/meridian/logistics.csv`).
- **Command I ran:**
  `supply-chain-investigate meridian --offline --simulate-timeout`
- **What I predicted:**
  The coordinator's fault-isolation boundary will catch the reader timeout gracefully without crashing the whole process. The briefing will note the outage in a header banner, route `late_shipment_count` into `## Incomplete`, and reclassify `on_time_delivery_rate` from `## Contested` down to a single-source claim under `## Well-Established`.
- **What actually happened (paste the key output line):**
  ```markdown
  > Sources unavailable: logistics unavailable (timeout)
  ...
  ## Incomplete
  ### late_shipment_count  _[missing source: timeout reading logistics]_
  - missing source: timeout reading logistics
  ```
- **How this differs from the unperturbed run:**
  In the nominal run (`supply-chain-investigate meridian --offline`), `logistics` succeeds and corroborates `average_lead_time_days` (12.0 days across 2 sources), reports 11.0 late shipments, and creates a direct conflict under `## Contested` for `on_time_delivery_rate` (95.0% audit vs 78.0% logistics). Under timeout perturbation, the missing source is explicitly annotated under `Incomplete` rather than crashing the pipeline or being quietly ignored.
