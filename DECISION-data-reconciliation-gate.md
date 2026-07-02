# Decision — Data-Platform reconciliation gate (audit BLOCKER #5)

> **Status:** OPEN — needs owner decision. **Owner:** ibrowe108 / Data-Platform.
> **Date:** 2026-06-17 · Related: `PRODUCTION_READINESS_AUDIT.md` §BLOCKER #5.
> **Scope:** the financial ("money Gold") publish path only. The behavioural /
> 108Zing path is canary-proven and **not** blocked by this.
>
> **Progress (2026-06-17) — design + slice 0 MERGED to Data-Platform `main`:**
> - Build-grounded design: `Data-Platform/docs/DESIGN_reconciliation_gate_implementation.md`
>   ([PR #50](https://github.com/108-Plaza/Data-Platform/pull/50), **merged**). Splits the two
>   existing docs' clashing "Option A" into three named strategies (**B** durability / **A-intra**
>   two-derivation / **A-ext** control-total).
> - **Slice 0 (durability pre-check) SHIPPED** ([PR #51](https://github.com/108-Plaza/Data-Platform/pull/51),
>   **merged**, CI green): `stratum-transform::recon::DurabilityReport` (fail-closed row-count
>   conservation) + `Promotion::{count_rows,durability_check}` + `RECONCILIATION_BREAKS` metric.
>   No owner decision needed (tolerance defaults to exact). **No DB/config/runner yet** (slices 1/3).
> - **Still gated (the decisions below):** the *value* reconciliation (A-intra/A-ext) waits on
>   Q-REC-1..5 **plus** a pinned AccountZing read contract (no client exists today) and the
>   money-type call (money is `Float64` today). BLOCKER #5 is **not** cleared by slice 0 alone.

## Context

The audit's owner-locked invariant:

> **Money Gold must balance against an independent source before publish.**
> No financial domain goes to production until this gate is built. Today there is
> **zero `reconcil*` code** in `Data-Platform`.

A "Gold" dataset (curated, published financial truth) can silently diverge from
the system it summarises (overflow/truncation casts, dropped offsets, partial
loads). Publishing unbalanced Gold = wrong financial numbers downstream. The gate
must **refuse to publish** a Gold partition until it reconciles to an independent
source within tolerance.

## The decisions the owner must make

1. **What is the independent source?** (the thing Gold is checked against)
   - (a) The upstream OLTP subledger totals (e.g. AccountZing settlement / GL
     control totals per tenant/period).
   - (b) The raw ingestion ledger / offset log in `stratum-catalog` (sum of
     committed source rows vs sum landed in Gold).
   - (c) A second independent re-aggregation computed by a different code path.
2. **What is the invariant + grain?** e.g. `sum(amount)` and `count(rows)` per
   `(tenant_id, period, currency)` must match within tolerance `ε` (ε = 0 for
   integer minor units; a small ε only if FX/rounding is in play).
3. **Gate placement & failure mode** — hard gate (block publish, page on-call)
   vs soft gate (publish + alert). Audit says **hard** for financial.
4. **Tolerance & exceptions** — known acceptable deltas (timing cutoffs,
   in-flight) and how they're whitelisted without weakening the gate.

## Options

| Option | What it checks | Pros | Cons | Effort |
|---|---|---|---|---|
| **A. OLTP control-total reconciliation** (recommended) | Gold period totals vs the source system's authoritative control totals (AccountZing GL/settlement) | Catches real divergence end-to-end; matches "independent source" intent | Needs a stable contract to read source control totals | M |
| **B. Offset/row-count durability gate** | rows committed in `stratum-catalog` offsets vs rows landed in Gold | Fully inside Data-Platform; no cross-system contract | Catches *loss*, not *value corruption* (truncating casts still pass) | S |
| **C. Dual independent re-aggregation** | two code paths recompute the same total | Strongest correctness signal | Most code + maintenance; still needs a source of record | L |

**Recommendation:** **A as the gate of record**, with **B as a cheap pre-check**
that runs first (fast fail on row loss before the heavier value reconciliation).
C only if a regulator/audit later demands independent recomputation.

## Implementation sketch (once A is chosen)

- `stratum-*` crate `reconciliation`: `reconcile(period) -> Report { source_total,
  gold_total, delta, within_tolerance }`, persisted to a `reconciliations` table.
- Publish pipeline calls it as a **blocking step**; `delta > ε` ⇒ refuse publish,
  emit a structured alert + a `data_reconciliation_failed` metric.
- CI integration test (now possible — see Media's `db-tests` pattern) against a
  seeded mismatch to prove the gate actually blocks.

## Open questions for the owner
- Which system holds the authoritative control totals, and how does Data read
  them (API? shared read-only view? exported control file)?
- Exact grain + tolerance per currency.
- Who is paged when the gate fails, and what is the manual remediation runbook?
