# Perturbation Log

For each system, one deliberate change to an input or configuration, prediction, command, and observed result.

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):** No file edited. Ran the existing futile-case test that simulates a blanked required field: `tests/test_us01_retry.py::test_ac_01_04_missing_source_halts_immediately` sets `endorsements: None` ("model couldn't find Schedule A") with `endorsements` confidence 0.2.
- **Command I ran:** From `Build a Validated, Routed Insurance Policy Extraction Pipeline/04-hitl-routing/solution/`: `.venv/Scripts/pytest tests/test_us01_retry.py -v` (captured in `01-policy-pipeline/perturbation-retry.txt`). Also attempted live pipeline `.venv/Scripts/policy-extractor pipeline data/policies/ --routing-out routing_decisions.json --seed 42` with no key (captured in `01-policy-pipeline/pipeline-run.txt`).
- **What I predicted:** A null required field should escalate immediately without burning retries, since no re-prompt can conjure Schedule A.
- **What actually happened (paste the key output line):** `tests/test_us01_retry.py::test_ac_01_04_missing_source_halts_immediately PASSED` — asserts `isinstance(result, RetryFutileEscalation)`, `result.field == "endorsements"`, `result.detected_pattern == "endorsements_absent"`, `client.call_count == 1  # no further API call`. Live-pipeline attempt failed before any retry with `TypeError: "Could not resolve authentication method..."`.
- **How this differs from the unperturbed run:** Well-formed or format/consistency failures retry with error feedback (`test_ac_01_03_format_failure_retries_with_error_appended PASSED`, `test_ac_01_03_consistency_failure_also_retries PASSED`); the missing-source case makes exactly 1 API call and halts, while recoverable cases make >1 call.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):** No file edited. Contrasted two bundled fixtures: clean `fixtures/documents/appraisal_informal_sqft.txt` vs perturbed-total `fixtures/documents/income_sum_mismatch.txt` (stated total 10892.17 does not equal line-item sum 9642.17). Equivalent to editing a stated total so it no longer matches line items.
- **Command I ran:** From `Build a Resilient Mortgage Document Extraction System/04-validate-mathematical-consistency/solution/`: `.venv/Scripts/mortgage-extract fixtures/documents/appraisal_informal_sqft.txt --mode replay --verbose` and `.venv/Scripts/mortgage-extract fixtures/documents/income_sum_mismatch.txt --mode replay --verbose` (captured in `02-mortgage-extraction/extract-run.txt` and `discrepancy-run.txt`).
- **What I predicted:** The clean appraisal validates `consistent: true`; the mismatched paystub is flagged `consistent: false` with calculated vs stated vs delta.
- **What actually happened (paste the key output line):** Clean: `"gross_living_area_sqft": 2400`, `"validation": {"consistent": true, "discrepancies": []}` (exit 0). Mismatch: `"validation": {"consistent": false, "discrepancies": [{"field": "total_monthly_income", "calculated": 9642.17, "stated": 10892.17, "delta": -1250.0}]}` (exit 1).
- **How this differs from the unperturbed run:** Same pipeline/classifier shape (`type=income_verification`, `tool=extract_income_verification` in both income runs); only the validator verdict flips from clean-true to flagged-false. Verified at model level too: same components with `stated_monthly_total=9642.17` validates true.

---

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):** No file edited. Used the built-in source-failure switch `--simulate-timeout`, which forces the logistics source to time out.
- **Command I ran:** From `Investigate Supply Chain Risk with Multi-Source Synthesis/03-resilient-coordinator/solution/`: `supply-chain-investigate meridian --offline` vs `supply-chain-investigate meridian --offline --simulate-timeout` (captured in `03-supply-chain/investigation-run.txt` / `briefing.md` and `03-supply-chain/timeout-run.txt` / `briefing-timeout.md`).
- **What I predicted:** The run still completes; logistics-backed claims move out of Contested/Well-Established into Incomplete with a timeout annotation.
- **What actually happened (paste the key output line):** Banner `> Sources unavailable: logistics unavailable (timeout)`; `### late_shipment_count  _[missing source: timeout reading logistics]_` + `- missing source: timeout reading logistics`; `## Contested` collapses to `_none_`; `on_time_delivery_rate` drops to single-source `95.0 percent — supplier_audit (as of 2026-04-10)` under Well-Established.
- **How this differs from the unperturbed run:** Healthy run has `## Contested` = `### on_time_delivery_rate  _[2 sources, conflicting]_` with `95.0 percent — supplier_audit (as of 2026-04-10)` vs `78.0 percent — logistics (as of 2026-04-05)`, and `late_shipment_count` Well-Established single-source `11.0 shipments — logistics (as of 2026-04-05)`. Timeout run exits 0 (no abort) with the failed source annotated, not reconciled.
