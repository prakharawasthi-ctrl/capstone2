# Perturbation Log

For each system, one deliberate change to an input or configuration, prediction, command, and observed result.

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):** Copied `Build a Validated, Routed Insurance Policy Extraction Pipeline/04-hitl-routing/solution/data/policies/POL-2025-001.txt` to `capstone-submission/01-policy-pipeline/POL-2025-001-perturbed.txt` and deleted the required value from the copy: `Total Policy Premium ........................  $ 1,847.62` → `Total Policy Premium ........................` (one line; original kept as baseline; `diff -u` in `01-policy-pipeline/perturbation-retry.txt`).
- **Command I ran:** From `Build a Validated, Routed Insurance Policy Extraction Pipeline/04-hitl-routing/solution/`: offline replay through the production code path (`validate_extraction` + `extract_with_retry` with `RecordedClient` faithful outputs) for baseline (`premium_amount=1847.62`) vs perturbed (`premium_amount=null` per system-prompt null-for-unstated rule); plus live attempts `.venv/Scripts/policy-extractor extract data/policies/POL-2025-001.txt --policy-id POL-2025-001` and `.venv/Scripts/policy-extractor extract <perturbed-copy> --policy-id POL-2025-001` (all captured in `01-policy-pipeline/perturbation-retry.txt`).
- **What I predicted:** Baseline validates clean; perturbed flags `missing_source` and escalates immediately with exactly 1 API call and no retry, since no re-prompt can recover text absent from the document.
- **What actually happened (paste the key output line):** Baseline: `validate_extraction -> None`, `outcome kind: PolicyExtraction`, `client.call_count: 1`. Perturbed: `field=premium_amount category=missing_source pattern=premium_amount_absent observed=null`, `outcome kind: RetryFutileEscalation`, `client.call_count: 1`, `reason: "Field 'premium_amount' returned null — the source document does not contain this information. Retry is futile; escalate to human review."` Live-CLI attempts on both files failed before any model call with `TypeError: "Could not resolve authentication method..."` (no ANTHROPIC_API_KEY/Vocareum key in this environment).
- **How this differs from the unperturbed run:** Same call count (1), opposite outcome: baseline succeeds with no escalation, perturbed halts as `premium_amount_absent` with no retry. Recoverable failures (`test_ac_01_03_format_failure_retries_with_error_appended`, `test_ac_01_03_consistency_failure_also_retries`) make 2+ calls with error feedback; this missing-source case makes exactly 1.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):** No file edited. Contrasted two bundled fixtures: clean `fixtures/documents/appraisal_informal_sqft.txt` vs perturbed-total `fixtures/documents/income_sum_mismatch.txt` (stated total 10892.17 does not equal line-item sum 9642.17). Equivalent to editing a stated total so it no longer matches line items.
- **Command I ran:** From `Build a Resilient Mortgage Document Extraction System/04-validate-mathematical-consistency/solution/`: `.venv/Scripts/mortgage-extract fixtures/documents/appraisal_informal_sqft.txt --mode replay --verbose` and `.venv/Scripts/mortgage-extract fixtures/documents/income_sum_mismatch.txt --mode replay --verbose` (captured in `02-mortgage-extraction/extract-run.txt` and `discrepancy-run.txt`).
- **What I predicted:** The clean appraisal validates `consistent: true`; the mismatched paystub is flagged `consistent: false` with calculated vs stated vs delta.
- **What actually happened (paste the key output line):** Clean: `"gross_living_area_sqft": 2400`, `"validation": {"consistent": true, "discrepancies": []}` (exit 0). Mismatch: `"validation": {"consistent": false, "discrepancies": [{"field": "total_monthly_income", "calculated": 9642.17, "stated": 10892.17, "delta": -1250.0}]}` (exit 1).
- **How this differs from the unperturbed run:** Same pipeline/classifier shape (`type=income_verification`, `tool=extract_income_verification` in both income runs); only the validator verdict flips from clean-true to flagged-false.

---

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):** No file edited. Used the built-in source-failure switch `--simulate-timeout`, which forces the logistics source to time out.
- **Command I ran:** From `Investigate Supply Chain Risk with Multi-Source Synthesis/03-resilient-coordinator/solution/`: `supply-chain-investigate meridian --offline` vs `supply-chain-investigate meridian --offline --simulate-timeout` (captured in `03-supply-chain/investigation-run.txt` / `briefing.md` and `03-supply-chain/timeout-run.txt` / `briefing-timeout.md`).
- **What I predicted:** The run still completes; logistics-backed claims move out of Contested/Well-Established into Incomplete with a timeout annotation.
- **What actually happened (paste the key output line):** Banner `> Sources unavailable: logistics unavailable (timeout)`; `### late_shipment_count  _[missing source: timeout reading logistics]_` + `- missing source: timeout reading logistics`; `## Contested` collapses to `_none_`; `on_time_delivery_rate` drops to single-source `95.0 percent — supplier_audit (as of 2026-04-10)` under Well-Established.
- **How this differs from the unperturbed run:** Healthy run has `## Contested` = `### on_time_delivery_rate  _[2 sources, conflicting]_` with `95.0 percent — supplier_audit (as of 2026-04-10)` vs `78.0 percent — logistics (as of 2026-04-05)`, and `late_shipment_count` Well-Established single-source `11.0 shipments — logistics (as of 2026-04-05)`. Timeout run exits 0 (no abort) with the failed source annotated, not reconciled.
