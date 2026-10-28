# Reflection Brief — Evaluation and Observability Capstone

**Name:** Dylan Burkey
**Date:** 2026-10-27

> Ground every answer in your own run. When a question asks for a number, file name, or line, paste
> it from your artifacts — a reviewer should be able to find it. Answers that are correct in the
> abstract but cite nothing do not meet the bar. Keep it short and specific.

Paths below are relative to this `capstone-submission/` folder unless they start with a course
project name. Code paths are relative to each project's final `solution/` directory:
`[P1]` = `Build a Validated, Routed Insurance Policy Extraction Pipeline/04-hitl-routing/solution/`,
`[P2]` = `Build a Resilient Mortgage Document Extraction System/04-validate-mathematical-consistency/solution/`,
`[P3]` = `Investigate Supply Chain Risk with Multi-Source Synthesis/03-resilient-coordinator/solution/`.

---

## 0. Environment

| Field | Value |
|---|---|
| OS & version | macOS 26.5.2 (build 25F84), arm64 — `environment.txt` |
| Python version | 3.12.15 in each project venv (system Python is 3.15.1; 3.12 used for chromadb/torch wheels) — `environment.txt` |
| Date run | 2026-10-27 |
| Screenshots | Taken on 2026-10-28 from a separate, later run, not the runs cited in this brief. Two numbers differ because of that: System 1 tests show 48 passed (`ANTHROPIC_API_KEY` was set, so the 3 live tests ran instead of skipping), and the live pipeline run shows 0 / 9 / 0 routing counts instead of 0 / 8 / 1, because model output varies between runs. The text captures (`tests.txt`, `pipeline-run.txt`, `routing_decisions.json`) are the cited evidence. |
| Ran any system live? (which) | Yes — System 1 (`pipeline-run.txt`, `perturbation-run.txt`) and System 2 perturbations + live control (`perturbation-run*.txt`, `live-unperturbed-control.txt`), via the course Anthropic endpoint with `claude-haiku-4-5-20251001`. System 2 baseline runs used `--mode replay`; System 3 ran `--offline`. |

---

## 1. Validated, routed pipeline

| Evidence | Value |
|---|---|
| Passing test count | 45 passed, 3 skipped — `01-policy-pipeline/tests.txt` |
| Routing output file | `01-policy-pipeline/routing_decisions.json` (9 decisions) |
| auto_approve / human_review / spot_check counts | 0 / 8 / 1 (plus 1 escalation, POL-2025-009) — `01-policy-pipeline/pipeline-run.txt` |

**1a. Retry boundary.** From your perturbation run (a required field removed), paste the escalation
record. How many API calls did the system make, and why is retrying a futile case worse than
escalating it?

I deleted the Exclusions block from a copy of POL-2025-003 (`perturbation-inputs/POL-2025-003.txt`).
From `01-policy-pipeline/perturbation-run.txt`:
```
INFO HTTP Request: POST https://claude.vocareum.com/v1/messages "HTTP/1.1 200 OK"
INFO validation_failed
{ "category": "missing_source", "detected_pattern": "exclusions_absent", "field": "exclusions",
  "kind": "escalation", "policy_id": "POL-2025-003",
  "reason": "Field 'exclusions' returned null — the source document does not contain this information. Retry is futile; escalate to human review." }
```
**One API call**, even though `--max-retries` defaults to 3. The short-circuit is
`[P1] policy_extractor/retry.py:79-86` (`if error.category == "missing_source": return
RetryFutileEscalation(...)`), triggered by `_check_required_present` in
`[P1] policy_extractor/validator.py:37-53`. Retrying can't add information the document doesn't
contain. Each retry costs another call and more latency, and it pressures the model through the
"your previous attempt was rejected" feedback (`[P1] policy_extractor/extractor.py:167-181`) to
produce *something* non-null. The likely result is a plausible, invented exclusions list that passes
validation and ships. Escalating keeps the honest null and sends it to the person who can get the
real form (PA-100). Format and consistency errors, which *are* fixable, still get the retry loop
(`retry.py:88-96`).

**1b. Reading the router.** Pick one `human_review` record from your routing output. Which of the
three signals (confidence, reviewer, integration) sent it to a human? If you had trusted the model's
confidence alone, what would have happened?

POL-2025-003 in `01-policy-pipeline/routing_decisions.json`:
`"decision": "human_review", "reason": "reviewer_disagreement=['coverage_limit']"`,
`fields_below_threshold: []`, `integration_failures: []`, lowest field confidence 0.90.
Only the **reviewer** signal fired. The confidence check is `c < threshold` with
`DEFAULT_CONFIDENCE_THRESHOLD = 0.90` (`[P1] policy_extractor/routing.py:21,49-51`), so 0.90 passes,
and `routing.py:59-72` routes to a human if *any* of the three signals fires. With confidence alone it
would have been `auto_approve`d with a disputed coverage limit. The coverage schedule reads
"Bodily Injury Liability ....... 50/100 (in thousands)", exactly the kind of split-limit notation a
confident model can mis-normalise. Across the whole run, 7 of the 8 `human_review` records
(002–006, 008, 010) had **no** field below threshold. Confidence alone would have auto-approved
8 of 9 policies rather than 0.

**1c. Where the aggregate lies.** Run the calibration snippet. Quote the one cell whose accuracy lags
its confidence, plus the overall figure. What does slicing by `policy_type × field` catch that a
single number hides?

From `01-policy-pipeline/calibration-report.txt`:
```
umbrella  exclusions      n=2 conf=0.93 acc=0.00 brier=0.865
OVERALL brier=0.291
```
The other cells look healthy (auto/premium_amount brier 0.003, home/deductible 0.010), and 4 of the
6 labels are correct, so the aggregate reads as "mostly fine." Slicing (`[P1]
policy_extractor/routing.py:171-205`, `calibration_report`) shows one cell that is *always wrong
at 0.93 confidence*. That cell clears the 0.90 routing threshold, so on confidence alone every
umbrella exclusions extraction would auto-approve wrong. A single number averages a systematic,
high-confidence failure in a small stratum together with many easy successes. The slice tells you
exactly which threshold or prompt to fix, and the stratified spot-check
(`apply_stratified_spot_check`, `routing.py:87`) keeps small strata like umbrella from being
sampled away.

---

## 2. Schema-enforced two-pass extraction

| Evidence | Value |
|---|---|
| Passing test count | 25 passed — `02-mortgage-extraction/tests.txt` (the project README says 28; the suite contains 25 tests) |
| Document run | `fixtures/documents/appraisal_informal_sqft.txt` — `02-mortgage-extraction/extract-run.txt` (exit 0) |
| Classified type | `appraisal` → tool `extract_appraisal` — `02-mortgage-extraction/classification-log.txt` |

**2a. Two guarantees.** Paste your discrepancy-run output. Tool use already forces valid JSON, yet the
validator still catches a bad sum. Why are these two different guarantees? Name one error each cannot
catch.

From `02-mortgage-extraction/discrepancy-run.txt` (`income_sum_mismatch.txt`, exit 1):
```
"base_monthly": 5416.67, "bonus_monthly": 1250.0, "commission_monthly": 2140.0,
"overtime_monthly": 385.5, "other_monthly": 450.0, "stated_monthly_total": 10892.17
"validation": { "consistent": false, "discrepancies": [ { "field": "total_monthly_income",
  "calculated": 9642.17, "stated": 10892.17, "delta": -1250.0 } ] }
```
Tool use (`tool_choice={"type":"any"}`, `[P2] mortgage_extractor/pipeline.py:103`, against the
JSON Schema in `schema.py`) guarantees **shape**: right keys, right types, nullable where allowed.
The validator (`[P2] mortgage_extractor/validator.py:39-48`) guarantees **arithmetic agreement**.
It sums the components in Python (`models.py:66`, `calculated_monthly_total`) and compares them with
the verbatim stated total, using a $1 tolerance. `10892.17` is a perfectly valid number, so the
schema can't know it's wrong. *Schema can't catch:* a well-typed but inconsistent total, as above.
*Validator can't catch:* a value with nothing to compare against. In my perturbation
(`perturbation-log.md`, System 2; `02-mortgage-extraction/perturbation-run.txt`) I changed a
bi-weekly stub's GROSS PAY to 6,073.08 while the line items still summed to 5,673.08. The model left
`stated_monthly_total: null` (there is no monthly total on a bi-weekly stub), so `validator.py:41`
(`if calculated is not None and stated is not None`) skipped the check and reported
`"consistent": true`. Neither guarantee caught it. A third gap: a `loan` object with a null
`amount` matches what the model emitted but fails the Pydantic model, and that error is uncaught at
`pipeline.py:129` (`perturbation-run-2.txt`).

**2b. Refusing to fabricate.** Run on a document missing a field. Paste that field's output. Why null
instead of an invented value? Point to the schema choice that allows it.

`02-mortgage-extraction/missing-field-run.txt` (`income_missing_bonus.txt`, a stub with only
Regular Salary, Holiday Pay and PTO lines): `"bonus_monthly": null` (also `commission_monthly`,
`overtime_monthly`: null), exit 0. The schema declares
`"bonus_monthly": {"type": ["number", "null"]}` (`[P2] mortgage_extractor/schema.py:104`), the
union-type idiom described at `schema.py:7`, so null is a *legal* answer and the model isn't forced
to invent a number to satisfy the type. The prompt reinforces it: "Return null for any field not
explicitly stated in the document. Do not infer, default, or fabricate" (`prompts.py:26`), with
the `missing-bonus-returns-null` few-shot (`prompts.py:51-65`). Null ≠ 0: "no bonus stated" and
"bonus of $0" are different facts for an underwriter. A fabricated 0 would also corrupt
`calculated_monthly_total`, which uses None to mean "no components stated" (`models.py:69-71`).

**2c. Normalization.** Quote one field where the source text and extracted value differ in format
("about 2,400 sq ft" → `2400`). Why normalize at extraction time rather than downstream?

Source (`[P2] fixtures/documents/appraisal_informal_sqft.txt:37`):
`Gross Living Area:   approximately 2,400 sq ft (above-grade finished)` → extracted
`"gross_living_area_sqft": 2400` (`02-mortgage-extraction/extract-run.txt`). The rule lives in one
constant, `NORMALIZATION_RULES` rule 1 (`prompts.py:17-19`), and the schema enforces the target type
(`"gross_living_area_sqft": {"type": ["integer", "null"]}`, `schema.py:76`). Normalizing at
extraction time means the model reads "approximately 2,400" in context (the same page also has
"~900 sq ft" basement, "441 sq ft" garage, "20,037 sq ft" lot), so it picks the right number. The
schema can then type-check it, and the validator and every downstream consumer can do arithmetic
without each one re-parsing free text, which would drift. The counter-example in my run: pay
frequency is *not* in the normalization rules, and the live run turned bi-weekly 5,673.08 into
`"base_monthly": 11346.16` (×2 instead of ×26/12), while the recorded run kept 5,673.08 unconverted
(`perturbation-run.txt` vs `missing-field-run.txt`). An unstated rule gives an unstable result.

---

## 3. Multi-source synthesis

| Evidence | Value |
|---|---|
| Passing test count | 34 passed — `03-supply-chain/tests.txt` |
| Briefing file | `03-supply-chain/briefing.md` (from `supply-chain-investigate meridian --offline`) |
| Section the conflict landed in | Contested (`on_time_delivery_rate`, flagged ⚠️ ESCALATE) |

**3a. Annotate, don't arbitrate.** Quote one conflicting-metric pair from your briefing — both values,
sources, dates. Give one way a reader is better served by the preserved conflict than by a single
reconciled number.

From `03-supply-chain/briefing.md`:
```
## Contested
### on_time_delivery_rate  _[2 sources, conflicting]_  ⚠️ ESCALATE
- escalation: high-impact metric is contested across sources
    - 95.0 percent — supplier_audit (as of 2026-04-10)
    - 78.0 percent — logistics (as of 2026-04-05)
```
The gap is (95−78)/95 ≈ 18%, above `REL_TOL = 0.10` (`[P3] supply_chain_risk/synthesis.py:26`,
`_disagree` / `_classify` at `synthesis.py:134-148`). The system shows both values with provenance
rather than averaging to ~86.5%. That tells the reader *who* says what: the 95% is the supplier's
self-reported audit ("consistent with our internal service-level commitments"), and the 78% is
derived from our own shipment records, which line up with the Long Beach port strike in the same
briefing. A buyer can act on that by challenging the supplier's number, or by weighting our own data.
A blended 86.5% is a number nobody reported, and it would hide the possibility that the supplier
is misstating performance. My perturbation shows the flip side: setting the audit to 82% made the
metric "corroborated across 2 sources" and dropped the escalation (`03-supply-chain/perturbation-run.txt`).
A single hard threshold decides whether the reader ever sees the disagreement at all.

**3b. Source goes dark.** Run with `--simulate-timeout`. Paste the part of the briefing showing the
failed source. How is "unreachable" handled differently from "nothing to report," and why does the run
still finish?

From `03-supply-chain/timeout-run.txt` (exit 0):
```
> Sources unavailable: logistics unavailable (timeout)
...
## Incomplete
### late_shipment_count  _[missing source: timeout reading logistics]_
### production_capacity_utilization  _[missing source: no source reported this metric]_  ⚠️ ESCALATE
```
The two Incomplete entries show the distinction. `late_shipment_count` is
*unreachable*: logistics is its only source and it timed out, so the coordinator records
`unavailable[metric_id] = f"{r.error.failure_type} reading {r.source}"` and a banner-level
`unavailable_sources` entry (`[P3] supply_chain_risk/coordinator.py:64-73`; the reader returns a
structured `FailureContext` with `failure_type="timeout"`, `readers.py:134`).
`production_capacity_utilization` is *nothing to report*: every source responded and none had it,
so it falls back to `"no source reported this metric"` (`synthesis.py:275-284`). The first means
"retry or check the feed"; the second means "this data doesn't exist, go get it." The run finishes
because each reader returns a result object (`ok` / `error`) instead of raising. The coordinator
builds the briefing from `ok_claims` only (`coordinator.py:61`) and annotates the gaps. Related
metrics degrade visibly too: `on_time_delivery_rate` drops from Contested to "single source only"
(supplier_audit 95%), and the briefing labels it that way instead of claiming corroboration.

**3c. Dates as a guardrail.** Quote two claims about the same supplier with different dates. How does
requiring a date stop a time difference from reading as a contradiction?

From `03-supply-chain/briefing.md`:
```
- 180.0 ppm — supplier_audit (as of 2026-04-10)
- 190.0 ppm — internal_quality (as of 2026-04-08)
```
and the on-time pair above (95.0% as of 2026-04-10 vs 78.0% as of 2026-04-05). `Claim.source_date: date`
is a required field (`[P3] supply_chain_risk/models.py:31`), and every value is rendered with it
(`synthesis.py:217`, `"(as of {c.source_date})"`), so no claim can enter the briefing undated. With
the date attached, a reader can tell "two sources disagree about the same moment" apart from "the
metric moved between two measurements." The defect pair is two days apart and within tolerance,
so it reads as corroboration. The news claims (recall 2026-02-19, port strike 2026-03-17, distress
2026-03-09) are anchored too. So a March strike explains a lower logistics on-time rate
collected 2026-04-05 instead of looking like an unexplained conflict, and a reader can judge
whether a 95% figure published 2026-04-10 covers the strike window. Without dates, an older and a
newer value would just look like two sources contradicting each other.

---

## 4. Synthesis

**4a. One principle.** Name the single moment in your runs (system + artifact) where *evaluate the
output, don't trust the model's word* most clearly caught something a trusting design would have
shipped.

System 1, `01-policy-pipeline/routing_decisions.json` + `pipeline-run.txt`: the model's own
confidence put **8 of 9** policies at or above the 0.90 threshold on every field. The independent
reviewer disagreed on at least one field in all 8 (e.g. POL-2025-004:
`reviewer_disagreement=['coverage_limit', 'deductible', 'exclusions']`, min confidence 0.90), and
the deterministic router (`routing.py:59-72`) sent them to `human_review`. A trusting design would
have written 8 `auto_approve` records, with disputed deductibles and coverage limits, straight into
the policy system. The run also escalated POL-2025-009 (`endorsements_absent`) and flagged
POL-2025-010's `premium_does_not_match_components` (`pattern_summary` in `pipeline-run.txt`), two
more problems the model reported with no sign of trouble.

**4b. Confidence ≠ correctness.** Pick the system where this mattered most, and explain why using
something you observed.

System 1. Every one of the 9 routed extractions had a minimum field confidence between 0.85 and
0.95, yet reviewer agreement told a different story. POL-2025-006 (umbrella) had min confidence
**0.95** and the reviewer still disagreed on `deductible`. POL-2025-008 reported 0.95 while the
reviewer disputed the `policy_type` itself, which is the field that decides which downstream rules
apply. The calibration slice measures the same thing: `umbrella exclusions conf=0.93 acc=0.00
brier=0.865` (`calibration-report.txt`). Confidence there is high, consistent, and uninformative.
It's what the model *says* about itself, and in these runs it didn't separate right answers from
disputed ones. It mattered most in System 1 because that pipeline's alternative to a human is
`auto_approve`, so a miscalibrated score turns directly into an unreviewed write.

**4c. Apply it.** Describe a real workflow where an LLM pulls structured results from messy input.
Which pattern — validated retry with escalation, independent review with deterministic routing, or
provenance-preserving conflict annotation — would you reach for first, and what would you instrument
to know when it broke?

Accounts-payable intake: an LLM extracts vendor, invoice number, line items, tax, and total from
emailed PDF/scanned invoices before they're queued for payment. I'd reach first for **validated
retry with escalation**, combined with System 2's arithmetic validator. Invoices carry their own
checksum (line items + tax = total, PO amount ≥ invoice), so a deterministic Python check catches the
costliest error, a wrong payable amount, without trusting the model. Format failures (bad dates,
currency strings) get a bounded retry with the validator's error fed back. A missing field (no
invoice number, unreadable total) escalates after one call, as in 1a, rather than being guessed.
Paying money is irreversible, so I'd add deterministic routing (System 1) on top: auto-approve only
when validation passes, the vendor matches the vendor master, and the amount is under a threshold.
**Instrumentation:** (1) escalation rate and first-pass validation-success rate by vendor and
template (a spike means a new invoice layout broke extraction); (2) retries-per-document and the
`detected_pattern` histogram, like `pattern_summary` in `pipeline-run.txt`; (3) a sliced calibration
report (vendor × field) built from AP clerks' corrections, to catch a high-confidence wrong cell
like umbrella/exclusions; (4) a weekly stratified spot-check of auto-approved invoices, so small
vendors aren't sampled away; and (5) a counter for uncaught schema-shape exceptions, the failure my
System 2 follow-up exposed (`perturbation-run-2.txt`), which should never be zero-visibility.
