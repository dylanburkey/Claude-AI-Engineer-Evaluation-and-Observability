# Perturbation Log

For each system, make one deliberate change to an input or configuration, predict the outcome, run
it, and record what actually happened. See the starters in the Instructions, or design your own (your
own experiment earns more credit).

All perturbed inputs are copies (originals untouched) and are saved in `perturbation-inputs/`.
Systems 1 and 2 ran live against the API (`claude-haiku-4-5-20251001` via the course endpoint);
System 3 ran `--offline`.

---

### System 1 — validated, routed pipeline

- **Change I made (file + what I changed):** Copied `data/policies/POL-2025-003.txt` and deleted the
  entire "Exclusions (partial list…)" block (5 lines) so `exclusions` — one of the validator's
  `_REQUIRED_FIELDS` — is genuinely absent. Copy: `perturbation-inputs/POL-2025-003.txt`.
- **Command I ran:**
  `policy-extractor extract perturbation-inputs/POL-2025-003.txt --policy-id POL-2025-003`
  (default `--max-retries 3`)
- **What I predicted:** The model returns `exclusions: null` rather than inventing exclusions; the
  validator classifies it as `missing_source` and the system escalates immediately after **one** API
  call, without spending any of its 3 retries.
- **What actually happened (paste the key output line):** Exactly one
  `POST https://claude.vocareum.com/v1/messages "HTTP/1.1 200 OK"` followed by `validation_failed`, then:
  ```
  "category": "missing_source", "detected_pattern": "exclusions_absent", "field": "exclusions",
  "kind": "escalation", "policy_id": "POL-2025-003",
  "reason": "Field 'exclusions' returned null — the source document does not contain this information. Retry is futile; escalate to human review."
  ```
  Exit code 1. Evidence: `01-policy-pipeline/perturbation-run.txt`.
- **How this differs from the unperturbed run:** In the unperturbed pipeline run
  (`01-policy-pipeline/pipeline-run.txt`) POL-2025-003 extracted successfully and was routed to
  `human_review` only because of a reviewer disagreement on `coverage_limit`. With exclusions removed
  it never reaches routing: it becomes a `RetryFutileEscalation` after a single call (no retry loop,
  no fabricated exclusion list) — the same path POL-2025-009 (`endorsements_absent`) took in the
  baseline run.

---

### System 2 — schema-enforced two-pass extraction

- **Change I made (file + what I changed):** Copied `fixtures/documents/income_missing_bonus.txt`
  and changed the stated `GROSS PAY` current-period total from `5,673.08` to `6,073.08`, leaving the
  line items (Regular Salary 5,673.08, Holiday 0.00, PTO 0.00) unchanged — a $400 mismatch.
  Copy: `perturbation-inputs/income_gross_mismatch.txt`.
- **Command I ran:** `mortgage-extract perturbation-inputs/income_gross_mismatch.txt --mode record`
  (live; the recorded fixture cache was left unchanged)
- **What I predicted:** The extractor captures the stated total, the consistency validator computes
  5,673.08 vs 6,073.08 and reports a discrepancy (delta ≈ −400), exit code 1.
- **What actually happened (paste the key output line):** My prediction was **wrong**:
  ```
  "base_monthly": 11346.16, ... "stated_monthly_total": null
  "validation": { "consistent": true, "discrepancies": [] }
  ```
  Exit code 0. Evidence: `02-mortgage-extraction/perturbation-run.txt`. The paystub is
  **bi-weekly** and never states a *monthly* total, so the model (correctly, per the "null rather than
  invent" rule) left `stated_monthly_total` null. With only one side of the comparison present, the
  validator had nothing to check and passed the document. Side observation: it normalised the
  bi-weekly pay to monthly as 5,673.08 × 2 = 11,346.16 (the accurate conversion is × 26/12 ≈
  12,291.67), and the recorded unperturbed response reported 5,673.08 unconverted — so the
  normalisation of pay frequency is not stable.
- **Follow-up (to exercise the validator directly):** Copied `income_sum_mismatch.txt` (a *monthly*
  stub whose stated total 10,892.17 already exceeds its line items 9,642.17) and edited both
  `TOTAL MONTHLY EARNINGS` and `Gross Earnings (Monthly)` to `9,642.17` so they *do* match
  (`perturbation-inputs/income_total_fixed.txt`). Prediction: `consistent: true`, exit 0.
  Result: the run **crashed twice** before reaching the validator, with two different
  schema-shape errors from the live model on the `loan` block (the paystub has no loan, so `loan`
  should be null):
  ```
  attempt 1: loan  Input should be a valid dictionary ... input_value='{\n  "amount": <UNKNOWN>\n}'
  attempt 2: loan.amount  Input should be a valid number ... input_value=None
  ```
  (`02-mortgage-extraction/perturbation-run-2-attempt1-crash.txt`, `perturbation-run-2.txt`).
  As a control, the **unperturbed** `income_sum_mismatch.txt` run live in the same session
  succeeded and flagged `calculated 9642.17, stated 10892.17, delta -1250.0`, exit 1
  (`02-mortgage-extraction/live-unperturbed-control.txt`).
- **How this differs from the unperturbed run:** Unperturbed, `discrepancy-run.txt` flags the −1,250
  income mismatch and `missing-field-run.txt` returns `bonus_monthly: null` with exit 0. The
  perturbations show two limits: (1) the validator only checks what was extracted, so a mismatch
  against a total the schema has no slot for (a bi-weekly gross) goes undetected — tool use made the
  output valid JSON, but that does not make it complete; (2) the pipeline does not handle a tool-use
  result that violates the Pydantic model (`MortgageExtraction.model_validate` raises an uncaught
  exception at `pipeline.py:129`), so one odd model response crashes the CLI instead of retrying or
  escalating.

---

### System 3 — multi-source synthesis

- **Change I made (file + what I changed):** Copied the whole `data/` root and edited
  `meridian/audit.json`: supplier-audit `on_time_delivery_rate` `value` 95 → 82 (and the matching
  "95%" in its `statement`). Logistics still derives 78%. 82 vs 78 is a 4.9% relative gap, inside
  `REL_TOL = 0.10` in `synthesis.py`. Copy: `perturbation-inputs/audit.json`.
- **Command I ran:** `supply-chain-investigate meridian --offline --data-root <perturbed copy>/data`
- **What I predicted:** The conflict disappears: `on_time_delivery_rate` moves from Contested to
  Well-Established "corroborated across 2 sources", its ⚠️ ESCALATE flag is dropped (escalation is
  only for *contested* or *incomplete* high-impact metrics), and Contested becomes `_none_`. Everything
  else stays the same.
- **What actually happened (paste the key output line):**
  ```
  ### on_time_delivery_rate  _[corroborated across 2 sources]_
      - 82.0 percent — supplier_audit (as of 2026-04-10)
      - 78.0 percent — logistics (as of 2026-04-05)
  ## Contested
  _none_
  ```
  Exit 0, no escalation on that metric. Evidence: `03-supply-chain/perturbation-run.txt`.
- **How this differs from the unperturbed run:** Unperturbed (`investigation-run.txt`):
  `on_time_delivery_rate _[2 sources, conflicting]_ ⚠️ ESCALATE — high-impact metric is contested
  across sources` (95 vs 78). Moving one source by 13 points flips it to "corroborated" — while the
  two sources still disagree by 4 points. The Contested/Well-Established boundary (and therefore
  human escalation) is a single hard 10% threshold, so a metric can be silently resolved even though
  the sources do not actually agree. Also run for reference: the suggested
  `--simulate-timeout` perturbation (`timeout-run.txt`) — logistics is marked
  `unavailable (timeout)`, `late_shipment_count` moves to Incomplete
  (`missing source: timeout reading logistics`), and the run still exits 0.
