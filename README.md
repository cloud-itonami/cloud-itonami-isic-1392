# cloud-itonami-isic-1392: Manufacture of made-up textile articles, except apparel

Open Business Blueprint for **ISIC Rev.5 1392**: manufacture of made-up textile articles, except apparel — an autonomous "actor" (LLM advisor behind an independent Governor, langgraph-clj StateGraph, append-only audit ledger) that coordinates back-office made-up-textile-articles-plant **operations**: production-batch data logging (cutting/sewing/hemming/quilting output-quality data — quality grade and defect-rate readings), cutting/sewing-line-equipment maintenance scheduling, equipment-safety/quality-defect concern flagging, and outbound finished-goods shipment coordination.

This repository designs a forkable OSS business for made-up-textile-
articles manufacturing — a plant that cuts and sews finished fabric
into bedding, curtains, towels, and table linens (ISIC Rev.5 1392,
distinct from wearing apparel [1410] and from carpets and rugs
[1393]) — run by a qualified operator so a made-up-textile-articles
plant keeps its own operating records instead of renting a closed
SaaS.

## What this actor does

Proposes **plant operations coordination**, not machine operation:
- `:log-production-batch` — cutting/sewing/hemming/quilting batch, output-quality data logging (administrative, not an operational decision)
- `:schedule-maintenance` — cutting/sewing-line-equipment maintenance scheduling proposal
- `:flag-safety-concern` — surface an equipment-safety/quality-defect concern (always escalates)
- `:coordinate-shipment` — outbound finished-goods shipment coordination proposal

## What this actor does NOT do

**CRITICAL SCOPE BOUNDARY** (cutting lines, sewing lines, hemming lines, quilting lines):

- Does NOT control cutting, sewing, hemming, or quilting-line equipment directly
- Does NOT make plant-safety, labor-safety, or equipment-safety decisions (that's the plant supervisor's exclusive human authority)
- Does NOT directly operate cutting/sewing-line equipment under any proposal (permanently blocked, see Architecture)
- ONLY proposes/coordinates operations back-office; all actuation requires explicit human approval
- Safety-concern flagging ALWAYS escalates — never auto-decided, no confidence threshold or phase below escalation

## Architecture

Classic governed-actor pattern (`madeuptextileops.operation/build`, a langgraph-clj StateGraph):
1. **`madeuptextileops.advisor`** (sealed intelligence node, `MadeUpTextileAdvisor`): proposes decisions only, never commits
2. **`madeuptextileops.governor`** (independent, `Made-Up Textile Articles Operations Governor`): validates against domain rules, re-derived from `madeuptextileops.registry`'s pure functions and `madeuptextileops.store`'s SSoT -- never trusts the advisor's own self-report
   - HARD invariants (always `:hold`, no override):
     - Plant/batch record must be independently verified/registered (`:verified?` AND `:registered?`) before any action is taken against it (equipment before maintenance scheduling, batch before shipment coordination)
     - The request's own `:effect` must be `:propose` (never a direct-write bypass)
     - `:op` must be in the closed four-op allowlist
     - The proposal's own `:effect` must be one of the four propose-shaped effects (no direct cutting/sewing-line-equipment control)
     - Directly operating cutting/sewing-line equipment (`:direct-operate? true`) is a PERMANENT, unconditional block
     - A shipment may not push a batch's own recorded shipped units past its own logged finished-unit count (independently recomputed)
     - No double-scheduling the same maintenance record
     - No fabricated `:quality-grade` value on a production-batch patch
     - No physically implausible `:defect-rate-percent` value on a production-batch patch
   - ESCALATE (always human sign-off, overridable by a human):
     - `:flag-safety-concern` always escalates, regardless of confidence
     - Low-confidence proposals
3. **`madeuptextileops.phase`** (Phase 0->3 rollout): `:schedule-maintenance`/`:flag-safety-concern`/`:coordinate-shipment` are NEVER in any phase's `:auto` set (permanent, matching the governor's own posture); only `:log-production-batch` may auto-commit at phase 3 when clean
4. **`madeuptextileops.store`** (append-only audit ledger + SSoT): a single `MemStore` backend behind a `Store` protocol (see ns docstring for why a second Datomic-backed backend is out of scope for this build)

## Development

```bash
# Run tests (top-level deps.edn already pins langgraph+langchain local/root)
clojure -M:test

# Run tests via the workspace :dev override alias (equivalent, kept for sibling-repo parity)
clojure -M:dev:test

# Run the demo
clojure -M:dev:run

# Lint
clojure -M:lint
```

## Status

`:implemented` — `governor.cljc`/`store.cljc`/`advisor.cljc`/`registry.cljc` + `deps.edn` complete the module set; tests green, demo runnable, langgraph-clj integration verified.

## License

AGPL-3.0-or-later
