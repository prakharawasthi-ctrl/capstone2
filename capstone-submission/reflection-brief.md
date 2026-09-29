# Reflection Brief — Evaluation and Observability Capstone

**Name:** Prakhar
**Date:** 2026-09-29

> All values below are pasted from my own artifacts in `capstone-submission/`. Reviewer: open the named file and search for the quoted string.

---

## 0. Environment

| Field | Value |
|---|---|
| OS & version | Microsoft Windows 11 Home Single Language, 10.0.26200 Build 26200 (see `environment.txt`) |
| Python version | Python 3.12.4 (see `environment.txt`) |
| Date run | 2026-09-28 |
| Ran any system live? (which) | None fully live. Policy live pipeline attempted with no key and failed auth (see `01-policy-pipeline/pipeline-run.txt` TypeError); fallback offline routing run used. Mortgage `--mode replay`, supply chain `--offline`. |

---

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 45 passed, 3 skipped (`01-policy-pipeline/tests.txt`: `======================== 45 passed, 3 skipped in 0.44s ========================`; skips are 3 `@pytest.mark.live` tests needing a key) |
| Routing output file | `01-policy-pipeline/routing_decisions.json` (offline-generated via `write_routing_decisions`; live run impossible without key — see `01-policy-pipeline/pipeline-run.txt` auth TypeError + offline fallback `9 passed`) |
| auto_approve / human_review / spot_check counts | 1 / 1 / 0 over the 2-record offline file (`routing_decisions.json`: POL-A `auto_approve`, POL-B `human_review`); routing-shape test `test_ac_04_05` covers the same writer |

**1a. Retry boundary.** From your perturbation run (a required field removed), paste the escalation
record. How many API calls did the system make, and why is retrying a futile case worse than
escalating it?

> Artifact `01-policy-pipeline/perturbation-retry.txt` (my edit: `capstone-submission/01-policy-pipeline/POL-2025-001-perturbed.txt` deleted `  $ 1,847.62` from the `Total Policy Premium` line; baseline kept). Faithful-model replay through production `extract_with_retry`: baseline (`premium_amount=1847.62`) → `validate_extraction -> None`, `outcome kind: PolicyExtraction`, `client.call_count: 1`; perturbed (`premium_amount=null`) → `field=premium_amount category=missing_source pattern=premium_amount_absent observed=null`, `outcome kind: RetryFutileEscalation`, `client.call_count: 1`, `reason: "Field 'premium_amount' returned null — the source document does not contain this information. Retry is futile; escalate to human review."` So exactly **1 API call**, no retry. Live `policy-extractor extract` on both files failed before any model call (`TypeError: "Could not resolve authentication method..."`, no key in this environment). Retrying is worse because a missing source is not a malformed guess the model can fix on re-prompt (contrast `test_ac_01_03_format_failure_retries_with_error_appended` which does retry): each extra call burns latency/money, risks hallucinating the premium to satisfy the schema, and delays the correct action — human escalation.

**1b. Reading the router.** Pick one `human_review` record from your routing output. Which of the
three signals (confidence, reviewer, integration) sent it to a human? If you had trusted the model's
confidence alone, what would have happened?

> Main answer: `test_ac_04_06_high_confidence_plus_reviewer_disagreement_still_routes_to_human_review` (`Build a Validated, Routed Insurance Policy Extraction Pipeline/04-hitl-routing/solution/tests/test_us04_routing.py:150-171`, PASSED in `01-policy-pipeline/tests.txt`). Input: every field at 0.99 (`high_conf = {"policy_type": 0.99, "premium_amount": 0.99, ...}`), reviewer disagrees on one field (`agreements[1] = _disagree("premium_amount")`), integration all pass (`_integration_pass_all()`). Result: `decision == "human_review"`, `reviewer_disagreements == ["premium_amount"]`, `fields_below_threshold == []`. Driving signal: the **independent reviewer** — confidence and integration were clean, only the reviewer fired. A confidence-only router (threshold 0.90) would have **auto-approved** this record because every field was above threshold; only the reviewer stopped it. This is the confident-but-wrong trap the question asks about. Second example from my routing file (`01-policy-pipeline/routing_decisions.json`, record POL-B from the `test_ac_04_05` writer shape): `"decision": "human_review"`, `"reason": "fields_below_threshold=['premium_amount']"`, `"premium_amount": 0.5` — there the driver is **confidence**, and a confidence-only router would agree (low confidence → human), so POL-B alone cannot demonstrate the confidence-alone failure.

**1c. Where the aggregate lies.** Run the calibration snippet. Quote the one cell whose accuracy lags
its confidence, plus the overall figure. What does slicing by `policy_type × field` catch that a
single number hides?

> Artifact `01-policy-pipeline/calibration-report.txt`: `umbrella  exclusions      n=2 conf=0.93 acc=0.00 brier=0.865` vs `OVERALL brier=0.291` (other cells: `auto premium_amount n=3 conf=0.95 acc=1.00 brier=0.003`; `home deductible n=1 conf=0.90 acc=1.00 brier=0.010`). Slicing catches a systematically miscalibrated pocket — umbrella/exclusions is wrong every time despite 0.93 confidence — that the moderate overall (0.291) averages away. Without the slice I would ship thinking "healthy overall" while one policy-type × field combination silently fails.

---

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 passed (`02-mortgage-extraction/tests.txt`: `============================= 25 passed in 1.10s ==============================`) |
| Document run | `fixtures/documents/appraisal_informal_sqft.txt --mode replay --verbose` → `02-mortgage-extraction/extract-run.txt` |
| Classified type | `type=appraisal`, `tool=extract_appraisal` (first stderr line of `extract-run.txt`: `mortgage_extractor.pipeline: classify: model=claude-haiku-4-5 in=2169 out=108 type=appraisal`) |

**2a. Two guarantees.** Paste your discrepancy-run output. Tool use already forces valid JSON, yet the
validator still catches a bad sum. Why are these two different guarantees? Name one error each cannot
catch.

> Artifact `02-mortgage-extraction/discrepancy-run.txt`: `"validation": {"consistent": false, "discrepancies": [{"field": "total_monthly_income", "calculated": 9642.17, "stated": 10892.17, "delta": -1250.0}]}`, exit 1. Tool choice / JSON schema (`extract_income_verification`) guarantees *shape*: right fields, right types, no missing keys — it cannot know 5416.67+1250.0+2140.0+385.5+450.0 = 9642.17 ≠ stated 10892.17 because both numbers are individually well-typed. The arithmetic validator guarantees *content consistency*: recomputed sum vs stated total within tolerance ($1.00) — it cannot catch a wrong-but-consistent file (e.g., every line item misread by the same offset so the sum still matches) or a mistyped name/property-type, which are schema/prompt problems, not math problems.

**2b. Refusing to fabricate.** Run on a document missing a field. Paste that field's output. Why null
instead of an invented value? Point to the schema choice that allows it.

> Artifact `02-mortgage-extraction/missing-field-run.txt` (paystub `income_missing_bonus.txt`, `type=income_verification`): `"income": {"base_monthly": 5673.08, "bonus_monthly": null, "bonus_ytd": null, "commission_monthly": null, "overtime_monthly": null, "other_monthly": null, "stated_monthly_total": null}` with `"validation": {"consistent": true, "discrepancies": []}`. Null because the source lists only Regular Salary 5,673.08 and no bonus/commission/overtime lines — inventing a bonus would corrupt underwriting. Schema choice enabling honesty: nullable unions — `mortgage_extractor/schema.py:104` `"bonus_monthly": {"type": ["number", "null"]}` and `mortgage_extractor/models.py:58` `bonus_monthly: float | None = None` — plus prompt rule "null for unstated fields" (covered by `test_ac_03_03_missing_bonus_returns_none`).

**2c. Normalization.** Quote one field where the source text and extracted value differ in format
("about 2,400 sq ft" → `2400`). Why normalize at extraction time rather than downstream?

> Source `fixtures/documents/appraisal_informal_sqft.txt`: `Gross Living Area:   approximately 2,400 sq ft (above-grade finished)` → artifact `02-mortgage-extraction/extract-run.txt`: `"gross_living_area_sqft": 2400` (integer; also `"appraised_value": 410000.0`). Rule lives in `mortgage_extractor/prompts.py:17-22` (`NORMALIZATION_RULES`, `"about 2,400 sq ft" → 2400`), exercised by `test_ac_03_04_informal_sqft_normalized_to_integer`. Normalize at extraction because every downstream consumer (validator, comparisons, JSON schema type integer) needs one canonical comparable form; pushing free-text variants downstream forces N parsers to re-handle "approximately/~ /about/comma/sq ft" and guarantees divergent math.

---

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 passed (`03-supply-chain/tests.txt`: `============================= 34 passed in 27.91s ==============================`) |
| Briefing file | `03-supply-chain/briefing.md` (saved stdout of `supply-chain-investigate meridian --offline`; same bytes as `investigation-run.txt`) |
| Section the conflict landed in | `## Contested` |

**3a. Annotate, don't arbitrate.** Quote one conflicting-metric pair from your briefing — both values,
sources, dates. Give one way a reader is better served by the preserved conflict than by a single
reconciled number.

> Artifact `03-supply-chain/briefing.md` / `investigation-run.txt`: `### on_time_delivery_rate  _[2 sources, conflicting]_  ⚠️ ESCALATE` / `- escalation: high-impact metric is contested across sources` / `- 95.0 percent — supplier_audit (as of 2026-04-10)` / `- 78.0 percent — logistics (as of 2026-04-05)`. Keeping both beats a reconciled ~86.5% because the gap itself is the signal: audit self-report vs operational logistics differ by 17 points 5 days apart — a reader can escalate, weight the fresher/independent logistics feed, or ask why the audit is inflated, instead of acting on a fictional average that neither source stated (covered by `test_contested_preserves_both_values`).

**3b. Source goes dark.** Run with `--simulate-timeout`. Paste the part of the briefing showing the
failed source. How is "unreachable" handled differently from "nothing to report," and why does the run
still finish?

> Artifact `03-supply-chain/timeout-run.txt` / `briefing-timeout.md`: banner `> Sources unavailable: logistics unavailable (timeout)`; `## Contested` → `_none_`; `## Incomplete`: `### late_shipment_count  _[missing source: timeout reading logistics]_` + `- missing source: timeout reading logistics`. "Unreachable" (timeout reading logistics) is an access failure annotated with the cause, distinct from "nothing to report" (`### production_capacity_utilization  _[missing source: no source reported this metric]_` — present in both runs). The run still finishes (exit 0) because the coordinator catches per-source failure, vectorizes only successful claims, and marks the gap (`test_single_failure_does_not_abort`, `test_access_failure_distinct_from_empty_result`) — aborting would discard the surviving audit/quality/news evidence.

**3c. Dates as a guardrail.** Quote two claims about the same supplier with different dates. How does
requiring a date stop a time difference from reading as a contradiction?

> Artifact `03-supply-chain/briefing.md`: `average_lead_time_days`: `- 12.0 days — supplier_audit (as of 2026-04-10)` vs `- 12.0 days — logistics (as of 2026-04-05)`; and `defect_rate_ppm`: `- 180.0 ppm — supplier_audit (as of 2026-04-10)` vs `- 190.0 ppm — internal_quality (as of 2026-04-08)` (schema enforces it: `test_claim_requires_source_date`). Required dates let the synthesizer/readers treat 5-April vs 10-April (or 8-April vs 10-April) as a time series — same metric moving or sampled differently over time — rather than a logical contradiction, and lets Contested vs corroborated classification account for staleness instead of flagging every delta as conflict.

---

## 4. Synthesis

**4a. One principle.** Name the single moment in your runs (system + artifact) where *evaluate the
output, don't trust the model's word* most clearly caught something a trusting design would have
shipped.

> Mortgage `02-mortgage-extraction/discrepancy-run.txt`: the model produced perfectly valid typed JSON (`tool=extract_income_verification`) whose numbers don't add up (`calculated 9642.17 vs stated 10892.17, delta -1250.0, consistent: false, exit 1`). A trusting design that stops at "valid JSON from the tool" ships a $1,250/month income overstatement to underwriting; the independent arithmetic check — which recomputes rather than believing the stated total — is the moment evaluation caught what parsing could not.

**4b. Confidence ≠ correctness.** Pick the system where this mattered most, and explain why using
something you observed.

> Policy pipeline. `01-policy-pipeline/calibration-report.txt` cell `umbrella exclusions n=2 conf=0.93 acc=0.00 brier=0.865`: the extractor is highly confident (0.93) and perfectly wrong (0% accurate) on that slice, while `OVERALL brier=0.291` looks acceptable. Combined with `routing_decisions.json` POL-B (`premium_amount` confidence 0.5 → human_review) and the mirror test `test_ac_04_06_high_confidence_plus_reviewer_disagreement_still_routes_to_human_review`, the system proves confidence must be gated by reviewer/integration signals and sliced calibration — trusting the raw number ships the confident-but-wrong umbrella exclusions.

**4c. Apply it.** Describe a real workflow where an LLM pulls structured results from messy input.
Which pattern — validated retry with escalation, independent review with deterministic routing, or
provenance-preserving conflict annotation — would you reach for first, and what would you instrument
to know when it broke?

> Workflow: LLM parsing vendor invoices (line items, totals, tax, due dates) into AP postings. First reach: **validated retry with escalation** (policy System 1 pattern) + mortgage-style math check: schema requires totals; validator recomputes line-item sum vs stated total and tax arithmetic; format/consistency failures retry once with appended error, missing-source (no line items at all) escalates immediately without retry. Instrument: per-field confidence histogram + sliced calibration (`vendor_type × field`, especially `tax`/`total`), retry-count vs escalation-rate by failure pattern (`format`/`consistency`/`missing_source`), validator discrepancy rate and delta distribution, and spot-check sample pass rate on auto-approved postings — alert when any slice's accuracy diverges from confidence or discrepancy rate spikes.
