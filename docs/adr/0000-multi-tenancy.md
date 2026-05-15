# ADR-0000: `store_id` as Location Stamp, Not Multi-Tenancy

**Status:** Accepted, 2026-05-15
**Filename:** Kept as `0000-multi-tenancy.md` for continuity with the design doc reference. The decision below explicitly rejects multi-tenancy.
**Source decision:** Office-hours session on 2026-05-13, design doc `~/.gstack/projects/smoxhakim-smox-lab/smoxWork-main-design-20260513-195608.md`, decision D10.
**Supersedes:** Resolves Open Question #9 in the source design doc.

---

## Context

The Smox Lab PRD's earlier wording was misleading. It described "3 SaaS pilot stores within 12 months," a "Brand / Retailer (SaaS Customer)" persona, and "100 stores via SaaS" scalability. Read in isolation, these phrases suggested Smox Lab is a multi-tenant SaaS platform serving independent retailers.

**It is not.** The clarified business model:

- Smox Lab is **one consumer brand**, operated by **one company** (the founder's).
- Multiple Smox Lab locations may exist over time. The first is the family-connected venue in Morocco. Additional Smox Lab locations, including the "3 stores within 12 months" in the PRD, are **company-owned, brand-uniform Smox Lab locations**, not licensee retailers.
- The exit path, if it materializes, is **selling the entire Smox Lab business as a single going concern** — analogous to selling a restaurant chain. Not licensing the platform to third-party retailers.
- The **creator marketplace is a core network-effect feature**, not a SaaS perk. A Moroccan creator publishing a design brings their followers into the Smox Lab ecosystem, who may then visit any Smox Lab location nearest them. This requires a **unified, brand-wide creator marketplace** that spans all Smox Lab locations.

What is left to decide is how to represent the fact that a particular order, kiosk session, printed garment, or AR photo happened at a specific physical Smox Lab location. That is a location-tagging question, not a tenancy question.

## Decision

`store_id` is a foreign-key column added to **operational tables only**. It identifies which Smox Lab location a particular operational event happened at. It is **not** a tenant identifier and is **not** used to partition global business data.

A new `stores` table is added in Phase 2 with one row per company-owned Smox Lab location.

### Tables with `store_id NOT NULL`

Operational tables — they record events that happen at a specific physical location.

| Table | Why |
|---|---|
| `orders` | An order is placed at a specific kiosk in a specific store |
| `order_status_history` | Denormalized for analytics query performance |
| `kiosk_sessions` | A kiosk session is intrinsically per-location |
| `printer_jobs` | A print job runs on a printer at a specific location |
| `printers` | Physical hardware lives at one store |
| `ar_photos` | Captured at a specific kiosk; location is part of metadata |
| `pause_history` | Prayer pause events differ per store (city-specific) |
| `operator_profiles` | Operators are staff at a specific store (single-store assumption for v1) |
| `audit_log` | Nullable — has `store_id` for store-context actions, NULL for global admin actions |

### Tables WITHOUT `store_id` (global)

Global business entities. Adding `store_id` here would actively break the brand model.

| Table | Why |
|---|---|
| `users` | One user account works at every Smox Lab location |
| `customers` | Customer profile data — global account |
| `creators` | Creators publish to the unified marketplace, not to one location |
| `published_designs` | Marketplace listings visible at every kiosk + web |
| `marketplace_purchases` | Unlock valid at any Smox Lab kiosk |
| `creator_earnings` | Aggregated across all locations |
| `payouts` | Per creator, not per location |
| `templates` | Brand-wide template library |
| `designs` | Saved/library entries; nullable `store_id` is allowed when origin was a kiosk session (see special case below) |
| `design_versions` | Inherits scope via FK to `designs` |
| `products` | Brand-wide catalog (every Smox Lab store sells the same products) |
| `product_variants` | Same reasoning as `products` |
| `prayer_schedules` | Folded into the `stores` table as columns (`city`, `calculation_method`, `prayer_duration_minutes`, `prayers_to_pause` jsonb) |
| `stores` | The parent table itself |

### Special case: `designs`

The `designs` table has a **nullable `store_id`** column:
- Populated when a design was created during a kiosk session at a specific location.
- NULL when the design was created on the web app or saved to a user's library.

This preserves the operational origin (useful for analytics like "designs created in-store vs. at home") without partitioning the design's lifecycle. A customer can promote a kiosk-created design into their library; the design's content stays the same, only the metadata interpretation shifts.

### Special case: `operator_stores` (deferred)

For v1, an operator works at exactly one store, captured by `operator_profiles.store_id NOT NULL`. The regional-manager scenario (one operator working at multiple stores) requires an M:N `operator_stores` junction table. This is deferred until the first such operator is hired and is documented as a known future migration.

### Tenant-resolution: there isn't one

There is no tenant boundary, so there is no gateway middleware to resolve a tenant from a request. `store_id` is written to operational rows by the service that originates the operation:

- **Anonymous kiosk sessions:** the kiosk identifies its `store_id` from its installation config (env var or config file on the kiosk hardware). The gateway trusts the kiosk's claim, verified against a `registered_kiosks` table at session start.
- **Authenticated operators:** `store_id` comes from `operator_profiles.store_id`, attached to the session JWT at login.
- **Web flows:** no `store_id` context until the user reaches a kiosk via the QR handoff flow.
- **Admin / global writes:** `store_id` is explicit in the request payload where applicable; NULL where the action is brand-wide.

## Rationale

1. **Multi-tenancy is the wrong abstraction.** Multi-tenancy exists to isolate data between independent customers of a SaaS product. Smox Lab does not have multiple SaaS customers; it has one company operating multiple locations of one brand. Forcing multi-tenancy where the business does not require it would corrupt the data model and add gateway-middleware complexity for no real benefit.

2. **The creator network effect is core, not optional.** A marketplace with 50 creators visible only at one store's kiosk is a worse product than a marketplace with 50 creators visible at every Smox Lab kiosk and on the web. The PRD's marketplace KPI ("50+ active influencer creators at launch") implies a unified pool.

3. **Customer accounts must be global.** A customer who designs a t-shirt on the web app and walks into any Smox Lab location to pick it up expects their account, saved designs, loyalty points, and order history to be there. Per-store accounts would be hostile UX.

4. **`store_id` on operational tables is still useful.** Analytics ("which store has the highest conversion?"), printer routing ("send this job to the printer that lives in the store this order was placed at"), and Prayer Pause ("Casablanca needs to pause now; Rabat is on a different schedule") all require knowing which location an operational event occurred at.

5. **Reversibility is asymmetric.** Adding `store_id` to operational tables and stamping every write with it costs almost nothing today. Removing `store_id` from a table that does not need it is trivial. Adding `store_id` to a global table (e.g., `users`) after a year of data has accumulated is a multi-week migration with downtime risk. The cost asymmetry favors operational-only.

6. **Rejecting SaaS-to-other-retailers is itself a decision.** It is not deferred. A future operator of Smox Lab who wants to revive SaaS will need to write a successor ADR explaining how to partition the marketplace, the creator pool, and customer accounts — and accept that doing so will harm the network-effect product they inherit.

## Consequences

### Positive

- Schema is simpler than a true multi-tenant schema. No tenant-resolution middleware. No tenant-aware caching layer. No tenant-isolation tests.
- The creator marketplace works as designed: one creator, one pool, all Smox Lab locations.
- Customer accounts are portable across locations from day one — supports the "design on web at home, walk into nearest store to pick up" use case explicitly.
- Cross-location order analytics is a simple `GROUP BY store_id`; no cross-tenant federation needed.
- Adding a second Smox Lab location is a row in `stores` plus a printer and kiosk install — no schema migration required.

### Negative

- A future pivot to SaaS-to-other-retailers is locked out without significant rework. **Accepted explicitly.**
- Two Smox Lab locations cannot have meaningfully different product catalogs without an additional layer (e.g., `store_product_overrides`). Deferred — for v1, all locations carry the same catalog.
- A regional-manager scenario needs an `operator_stores` M:N table when it appears. Deferred.
- `audit_log` queries that span store-context and global actions must handle a nullable `store_id`.

### Ongoing concerns

- **Anonymous kiosk session integrity.** The kiosk's `store_id` comes from its install config and is trusted by the gateway. If a kiosk is physically moved between stores, the install config must be updated. **Mitigation:** the kiosk's reported `store_id` is verified against a `registered_kiosks` fingerprint table at session start; mismatches alert operations and refuse the session.
- **Cross-store data leakage risk.** Essentially zero by design, because almost all customer-visible data is global. The only exception is operational tables, where a naive API could expose Store A's orders to Store B's operator. **Mitigation:** operator-scoped endpoints filter by `operator_profiles.store_id`; only the `admin` role can query across stores.
- **Prayer-schedule integrity.** The schedule lives on `stores` rows. A wrong city changes when the system pauses. **Mitigation:** the admin UI for prayer config has a confirmation step showing a 7-day preview of resulting pause times before save.
- **Schema-evolution discipline.** A future migration that adds a new operational table must remember to include `store_id NOT NULL`. **Mitigation:** a custom Prisma migration lint rule (or pre-commit grep) requires every new table to either declare `store_id NOT NULL` or carry a `// @global` comment in the schema file.

## Status

**Accepted, 2026-05-15.** This ADR is binding for Phase 2 of the build plan. Override requires a successor ADR with explicit counter-reasoning, including a plan for how to migrate accumulated data.
