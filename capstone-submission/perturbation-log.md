# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run
it, and record what actually happened. See the starters in the Instructions, or design your own (your
own experiment earns more credit).

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):**
  In `data/policies/POL-2025-009.txt` (and verified via `tests/test_us01_retry.py::test_ac_01_04_missing_source_halts_immediately`), the document explicitly omitted the referenced endorsement schedule ("Schedule A referenced in policy text is not attached"), leaving the mandatory `endorsements` field completely missing from the ground truth text.
- **Command I ran:**
  `pytest tests/test_us01_retry.py::test_ac_01_04_missing_source_halts_immediately -v`
- **What I predicted:**
  The validator will flag this as a `missing_source` failure. Because re-prompting an LLM cannot recover information that is physically absent from the source document, the system's fail-fast boundary should immediately classify retry as futile, halt after exactly one API call (`client.call_count == 1`), and output a `RetryFutileEscalation` instead of making repeated calls that might tempt the model into hallucinating endorsements.
- **What actually happened (paste the key output line):**
  ```python
  RetryFutileEscalation(
      policy_id='POL-2025-009',
      field='endorsements',
      category='missing_source',
      detected_pattern='endorsements_absent',
      reason='Missing source document data for endorsements cannot be resolved by retry.'
  )
  assert client.call_count == 1  # no further API call
  ```
- **How this differs from the unperturbed run:**
  On unperturbed policies like `POL-2025-001.txt`, all required fields exist in the source document, allowing the extractor to succeed cleanly on the first pass (or retry formatting errors up to 3 times with specific feedback). When a required source field is genuinely missing, the system short-circuits further API calls immediately.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):**
  In `fixtures/documents/income_sum_mismatch.txt`, the line items for income (base $5,416.67, bonus $1,250.00, commission $2,140.00, overtime $385.50, and other $450.00) sum to $9,642.17. The stated monthly income figure was edited to `$10,892.17` (creating a $1,250.00 artificial discrepancy).
- **Command I ran:**
  `mortgage-extract fixtures/documents/income_sum_mismatch.txt --mode replay`
- **What I predicted:**
  The LLM will produce syntactically valid JSON conforming strictly to the Pydantic schema (valid positive floats, matching keys). However, the post-extraction deterministic validator will calculate the line-item sum, compare it against the stated total, detect that the -$1,250.00 difference exceeds the $1.00 tolerance, and flag `consistent: false`.
- **What actually happened (paste the key output line):**
  ```json
  "validation": {
    "consistent": false,
    "discrepancies": [
      {
        "field": "total_monthly_income",
        "calculated": 9642.17,
        "stated": 10892.17,
        "delta": -1250.0
      }
    ]
  }
  ```
- **How this differs from the unperturbed run:**
  On an unperturbed clean document like `fixtures/documents/appraisal_informal_sqft.txt`, the validator confirms mathematical consistency, outputting `"consistent": true` and `"discrepancies": []`. The perturbation demonstrates that tool calling and schema validation only enforce syntax; programmatic business logic is needed to catch arithmetic contradictions.

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
