# ADR-0001: MadeUpTextileAdvisor ⊣ Made-Up Textile Articles Operations Governor architecture

## Status

Accepted. `cloud-itonami-isic-1392` promoted from `:spec` to
`:implemented` in the `kotoba-lang/industry` registry, following the
verified fresh-scaffold protocol established by prior actors in this
fleet.

## Context

`cloud-itonami-isic-1392` publishes an OSS blueprint for made-up-
textile-articles-plant **operations coordination**
(cutting/sewing/hemming/quilting production-batch output-quality data
logging -- quality-grade and defect-rate readings -- cutting/sewing-
line-equipment maintenance scheduling, equipment-safety/quality-
defect concern flagging, and outbound finished-goods shipment
coordination). Like every actor in this fleet, the blueprint alone is
not an implementation: this ADR records the governed-actor
architecture that promotes it to real, tested code, following the
same langgraph StateGraph + independent Governor + Phase 0->3 rollout
pattern established across the cloud-itonami fleet.

The closest domain analog is `cloud-itonami-isic-1313` (Finishing of
textiles): both are back-office plant-operations-coordination actors
with real physical-safety-relevant equipment (cutting/sewing/hemming/
quilting lines vs. dyeing/printing/bleaching/mercerizing lines),
production-batch tracking with quality/grade data, and equipment
maintenance scheduling with a permanent block on directly operating
that equipment. This build mirrors 1313's module shape (advisor ⊣
governor ⊣ phase ⊣ store, four ops, `MemStore`-only backend) closely,
substituting made-up-textile-articles-specific ground truth:
`quality-grade` in place of `colorfastness-grade`, `defect-rate-
percent` in place of `shrinkage-percent`, `units`/`units-produced`/
`shipped-units` in place of yardage, and a permanent `:direct-
operate?` block in place of 1313's own `:direct-operate?` block --
both are a proposal-level field that, if set true, attempts to bypass
"propose/schedule a DRAFT" and reach actual equipment operation.

This vertical is distinct from 1313 in a structural way this ADR
calls out explicitly: **1392 is finished-goods manufacture** (cutting
and sewing finished fabric into bedding, curtains, towels, and table
linens -- ISIC Rev.5 wording "made-up textile articles, except
apparel"), not contract processing of customer-supplied greige
fabric. This vertical also has ONE FEWER permanent scope boundary
than 1313: 1313's dyeing/printing/bleaching/mercerizing lines
generate chemical process wastewater subject to effluent-discharge-
permit regimes, requiring a dedicated permanent
`effluent-permit-decision-blocked` governor check; cutting and sewing
finished fabric involves no chemical discharge, so this vertical has
no analogous check. Its own permanent scope boundary is narrower and
singular: "any proposal touching cutting/sewing-line-equipment
control is a hard, permanent block" -- implemented as the same two-
layer combination 1313 itself uses for its own equipment-control
boundary (a closed proposal-effect allowlist PLUS a dedicated
`:direct-operate? true` block on `:schedule-maintenance`), just
without the second, chemical-discharge-specific boundary 1313 also
carries.

This vertical has NO pre-existing `kotoba-lang/made-up-textiles`-style
capability library to wrap (verified: no such repo exists). This
build therefore uses self-contained domain logic -- pure functions in
`madeuptextileops.registry` (equipment/batch verification, shipment-
unit recompute, quality-grade validation, defect-rate plausibility
validation) are re-verified independently by the governor, the same
"ground truth, not self-report" discipline established across prior
actors (most directly `cloud-itonami-isic-1313`'s
`finishingops.registry`).

This blueprint's own `:itonami.blueprint/governor` keyword,
`:made-up-textile-articles-operations-governor`, is grep-verified
UNIQUE fleet-wide (`gh search code
"made-up-textile-articles-operations-governor"`, zero hits before
this repo was created). The namespace prefix `madeuptextileops` is
similarly grep-verified unique (`gh search code "madeuptextileops"`,
zero hits).

## Decision

### Decision 1: Self-contained domain logic (no external made-up-textile-articles capability library to wrap)

Unlike actors that delegate to pre-existing domain libraries, this
made-up-textile-articles vertical has NO pre-existing capability
library to wrap. The equipment/batch-verification / shipment-unit /
quality-grade / defect-rate validation functions live as pure
functions in `madeuptextileops.registry` and are re-verified
independently by `madeuptextileops.governor` -- the same "ground
truth, not self-report" discipline established across prior actors
(most directly `cloud-itonami-isic-1313`'s `finishingops.registry`).

### Decision 2: Coordination, not control — scope boundary at the back-office

This actor is **strictly back-office coordination** of made-up-
textile-articles-plant operations. It does NOT:
- Control cutting, sewing, hemming, or quilting-line equipment directly
- Make plant-safety, labor-safety, or equipment-safety decisions (exclusive to the human plant supervisor)
- Directly operate cutting/sewing-line equipment under any proposal

All proposals are `:effect :propose` only. The advisor proposes; the
governor validates; escalation paths funnel to human plant-supervisor
approval. This is not a replacement for the supervisor's authority —
it is a proposal-screening and documentation layer.

**CRITICAL SAFETY BOUNDARY**: made-up-textile-articles manufacturing
carries real physical-safety dimensions (blade/cutting-tool exposure
risk on cutting lines, moving-fabric and needle entanglement risk on
sewing/hemming/quilting lines). Safety-concern flagging NEVER
auto-commits. All safety concerns escalate immediately to human
review.

### Decision 3: Safety-concern escalation — always human sign-off

`:flag-safety-concern` (equipment-safety or quality-defect concern)
ALWAYS escalates, never auto-commits. This is not a "low-stakes
proposal" — it is a circuit-breaker that must reach human authority,
deliberately broad enough to cover both an equipment-safety finding
and a quality-defect finding simultaneously.

### Decision 4: Two independent verified/registered gates (equipment AND batch), not one

Unlike a single-entity-gated vertical, this vertical has TWO entity
kinds each gating a different op: `:schedule-maintenance`
independently verifies the referenced **equipment** unit's own
`:verified?`/`:registered?` fields; `:coordinate-shipment`
independently verifies the referenced **batch**'s own
`:verified?`/`:registered?` fields. Both are the same "plant/batch
record must be independently verified/registered before any action"
HARD invariant applied to the two distinct record kinds this domain
actually has. `:coordinate-shipment` additionally independently
recomputes whether a batch's own recorded shipped-to-date unit count
plus the proposal's own claimed unit count would exceed the batch's
own recorded finished-unit count -- never taken on the advisor's
self-report.

### Decision 5: HARD invariants (no override)

Four HARD governor invariants (elaborated into ten concrete checks in
`madeuptextileops.governor`, mirroring `cloud-itonami-isic-1313`'s own
elaboration of its HARD invariants into concrete checks, minus the
one effluent-discharge-specific check this vertical has no analog
for) block proposals and cannot be overridden by human approval:
1. Plant/batch record (equipment for maintenance, batch for shipment) must be independently verified/registered before any action is taken against it, and a shipment's units must independently recompute within the batch's own logged finished-unit count
2. Proposals must be `:effect :propose` only (never direct equipment control)
3. Any proposal touching cutting/sewing-line-equipment control is a hard, permanent block -- implemented as a closed proposal-effect allowlist PLUS a dedicated `:direct-operate? true` block on `:schedule-maintenance`
4. The op allowlist is closed — `:log-production-batch`/`:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` only

## Consequences

(+) Made-up-textile-articles-plant operations back-office now has a
documented, governed, auditable coordination layer that funnels all
decisions through independent validation before human approval.

(+) The "coordination, not control" boundary is explicit in code: all
`:effect :propose`, all real-world actuation requires human plant-
supervisor sign-off.

(+) Scope is bounded and verifiable: four HARD invariants (elaborated
into ten concrete governor checks) protect against scope creep into
unauthorized equipment operation. Safety concerns are a circuit-
breaker, not a threshold.

(+) Safety-critical discipline is explicit: safety-concern flagging
cannot be rate-limited, suppressed, or auto-decided by phase gate.

(-) Still a simulation/proposal layer, not a real plant-operations
control system. Equipment actuation remains human-controlled via
external channels.

(-) No integration with real plant-management databases (equipment
telemetry, batch tracking, freight dispatch) — this is a standalone
coordinator blueprint.

## Verification

- `cloud-itonami-isic-1392`: `clojure -M:test` green (all tests pass;
  see the superproject ADR and `kotoba-lang/industry` registry entry
  for the exact `Ran N tests containing M assertions, 0 failures, 0
  errors` output, verified from an independent fresh clone), `clojure
  -M:lint` clean, `clojure -M:dev:run` demo narrative exercises
  proposal submission, escalation, and every HARD-hold scenario
  directly (not-propose-effect, unknown-op, equipment-not-verified,
  batch-not-verified, shipment-units-exceeded, direct-operate-
  blocked, already-scheduled, invalid-grade, invalid-defect-rate).
- All source is `.cljc` (portable ClojureScript / JVM / nbb) — no
  JVM-only interop; the actor graph is invoked exclusively via
  `langgraph.graph/run*` (not `.invoke`, which is not cljs-portable).
- Audit ledger is append-only, all decisions are traced; every settled
  request (commit or hold) leaves exactly one ledger fact.
- `deps.edn` pins `io.github.kotoba-lang/langgraph` and
  `io.github.kotoba-lang/langchain` via `:local/root` directly in the
  top-level `:deps` (not only under a `:dev` alias), so a bare
  `clojure -M:test` resolves offline inside the monorepo checkout.
