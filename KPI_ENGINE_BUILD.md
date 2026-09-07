# KPI Engine — build specification (v2: the generic engine)

## Instructions for the implementing agent

**You are building this in the production environment.** This document is
your specification and your brief; it is self-contained and you are not
expected to find anything outside it.

**Your task.** Implement the KPI engine specified below, one work package
at a time, in the order given in §10. Work on a branch per work package,
commit with clear messages, and keep the return form (§13) updated as you
go — the requester builds the consuming side against that form, not
against this document, so precision there saves a round trip.

**Start here, and stop where it says to stop:**

1. Read §0–§2 and §11. Everything else is reference you will come back to.
2. Run **WP0** (§10): the decision tree in §1.3, the column inventory
   against §6.1, and the **spine check in Appendix A** — does this
   environment already have levelled, dated organisation nodes with
   precomputed paths and per-source link tables? If yes, build on them.
   If no, Appendix A is a prior work package; the requester has three
   reference transforms (~1 100 lines of PySpark) implementing it — ask
   for them rather than writing from scratch.
3. Fill in return-form items 1–4 from what you found.
4. **Report back to the requester** with those items and the open
   questions in §11 — each with your proposed answer where the code lets
   you form one (a candidate pilot KPI against WP3's criteria, the
   milestone codes you can see, the axes the data supports). **Do not
   proceed on assumed answers to §11.** They change what gets built.
5. Then WP1 onward.

**Rules that hold throughout:**

- **Additive only** (§0.3). Every field is optional; nothing existing is
  renamed or redefined; a row written without the new fields is identical
  to today's. If delivering something appears to need a breaking change,
  stop and ask.
- **Build nothing above the engine.** No UI, no dashboard adapter, no
  agent prompts, no Workshop app. The consumers integrate later against
  the rows and functions specified here.
- **Prove one vertical slice before widening** — every primitive, one
  KPI, one axis, gates passing on real data. Generalising before that
  multiplies unproven surface.
- **Where the shapes fit awkwardly against this environment's reality,
  adapt them and record the deviation** in the return form rather than
  forcing the letter of this document. Where a rule is stated as a
  correctness rule — never truncate an unranked set, never reconstruct
  membership, never publish past a failed gate — adapt the shape, not the
  rule.
- **If something here appears to be missing or contradictory, ask the
  requester.** Do not invent a filter language, a severity value, a ramp
  curve or a milestone code that is not in this document.

---

> **What this is.** A self-contained, build-ready specification of a KPI
> engine that can turn **any ontology object type** into a KPI, attribute
> it to **any level of the engineering structure** — programme, project or
> aircraft, work package, supplier, department — judge it against
> **milestone-driven expectations**, and expose the result through a small
> set of **functions that AIP Logic can call as tools**. Object model,
> contracts, algorithms, validation gates, function API and an ordered work
> plan. Hand the whole file to a coding agent running in the **production
> environment** and it can execute without access to any other document.
>
> **Where it sits.** Layer 4 (Measurement) of the engineering operations
> backbone described in `BACKBONE.md`. Its axes are the backbone's spine;
> its `attribute` primitive is the backbone's attribution layer; its
> function API is the measurement family of the backbone's callable
> surface. Nothing here depends on that document — but everything here
> is designed to be one layer of it.
>
> **Relationship to the concept.** A companion concept document argues
> *why* the engine looks like this; everything load-bearing from it is
> restated here as a rule. Where the two disagree, this file wins for
> implementation and the concept wins for intent.
>
> **What changed from v1.** v1 specified one KPI shape well (a scoped count
> with lineage). v2 generalises it into a kernel of five primitives so
> that a new KPI is a configuration object, never a function; scope becomes
> a position on a dated spine rather than a column; judgment resolves
> against milestone objects per scope rather than dates on a config row;
> and the delivery layer is a function API for AIP Logic rather than a
> dashboard adapter. The consuming dashboard is out of scope here on
> purpose — it integrates later, against the same rows and functions.
>
> **How to use it.** §0–§9 are normative — implement exactly, or record a
> deviation. §10 is the ordered work plan; ship the work packages in order,
> each one independently deployable. §11 lists what is settled and what
> still needs a human. §13 is the form to hand back. Do not go looking for
> external references; if something appears missing, ask the requester.

---

## 0. Scope, and the one rule

### 0.1 The generality claim — what "any KPI" has to mean

The engine is finished when each of the following is a configuration
change and not a code change:

| Requirement | Test |
|---|---|
| **Any object type.** | Onboarding a new ontology object type (a test result, a deviation, a supplier delivery, a document) as KPI material = adding one `ObjectTypeProfile` (§5.3). |
| **Any reach.** | An object that is wired to a programme, aircraft, work package, supplier or department *directly or through any chain of links* can be counted against that node. The chain is declared as a path (§5.2), validated against ontology metadata, and never coded. |
| **Any level.** | The same definition yields a number at every level of the structure — fleet, programme, project/aircraft, work package, team — from one build, with each level a stored row, never a query-time rollup. |
| **Any axis.** | The same objects are viewed by organisation *and* by supplier *and* by aircraft without redefining the KPI. Axes are declared; attribution is precomputed per axis. |
| **Any math, from a closed set.** | Count, ratio, sum, weighted sum, age, throughput, percentile, window aggregate, and derived (an expression over other KPIs). New math is a new mechanism — reviewed once, reused by every KPI. |
| **Milestone-driven time.** | "Good" is defined relative to the project's own gates: a ramp from `MG5` to `MG7` resolves to *that project's* dates, from the Milestone objects, by plan or forecast. Fifty projects, one definition. |
| **Honest at every level.** | Objects that reach no node are counted and shown, never dropped. A ramp that moved because a gate slipped says so. A capped list carries its flag. A reorg is not a performance change. |
| **Callable.** | Every answer — the number, the objects, the change, the reason — is a function over stored rows that AIP Logic can invoke and cite. |

### 0.2 The four questions, still

| # | Question | Answered by | Stored as |
|---|---|---|---|
| 1 | What is the number? | measurement | `kpi_value`, `numerator`, `denominator` |
| 2 | Is it good — *for this project, at this date*? | judgment | `severity`, `judgment_context` |
| 3 | Why — which objects? | lineage capture | `contributors`, `exceptions`, `roster` |
| 4 | What changed, and why? | the movers diff | `movers`, `movers_summary`, with a `reason` on every entry |

### 0.3 The one rule: additive only

**Every field this specification adds is optional.** Consumers degrade
gracefully when a field is absent. The spec repository, the pipeline and
every consumer deploy **independently, in any order**. Add no required
field; change no existing field's meaning or type; rename nothing; every
new field's default equals "absent". A row written without the new fields
is byte-identical to today's. If delivering something here appears to
require a breaking change, **stop and ask the requester.**

### 0.4 Definition of done, per KPI

A KPI is *on the engine* when:

1. Its definition is one `KpiDefinition` object (§3) with a signed-off
   population and a validated attribution path per axis.
2. Every cycle writes value + lineage at the aggregation step, at every
   scope on every declared axis, including `unassigned`.
3. Its snapshot carries the severity judged that cycle *and the context it
   was judged in* (resolved ramp dates, policy, config version).
4. Its snapshot carries `movers_summary` at every scope and object-level
   `movers` wherever a predecessor roster exists — every entry with a
   reason.
5. All gates (§8) pass on real data, every cycle.
6. Re-running the cycle from the same inputs reproduces the row, lineage
   included, hash-identical.

---

## 1. Architecture

### 1.1 The layers

```
L0  Sources      ontology objects — any type with a profile
L1  Population   WHICH objects a KPI considers            — declared: anchor → path → target
L1' Attribution  WHERE each object counts                 — declared: target → path → spine node, per axis, dated
L2  Measurement  the number + its lineage                 — mechanism library
L3  Judgment     expectation for THIS scope at THIS date  — policy library, milestone-resolved
L4  Time         snapshots, movers, context changes       — the build
L5  Functions    the callable surface for AIP Logic       — lookups, never computation
```

Two properties hold the stack up. **Each layer's output is stored, not
implied** — a consumer never re-runs a lower layer to interpret an upper
one. **Compute once, read many** — board numbers come from a scheduled
build; query time is lookup time.

### 1.2 The kernel: five primitives

Everything the engine does is a composition of these five. A KPI
definition parameterises them; nothing else in the system computes.

| Primitive | Signature (informal) | Used by |
|---|---|---|
| `resolve(spec, as_of) → ObjectSet` | anchor filter → path traversal → target set, with variables bound to `as_of` | population (L1), drill-down, `traceObject` |
| `attribute(set, axis, as_of) → {object_id → node_id \| null}` | target → path → spine node on one axis, **as the structure stood at `as_of`** | scope fan-out (L1′) |
| `measure(set, mechanism, params) → (value, num, den, contributions)` | one of the closed mechanism set | L2 |
| `judge(value, policy, context) → Judgment` | pure function of value, policy config and a resolved context (milestone dates, child judgments, history) | L3, `judge()` function, replay |
| `diff(M_t, M_{t−1}, classify) → Movers` | set difference over rosters, every entry classified by reason | L4 |

**`resolve` and `attribute` are the same traversal in opposite
directions.** Population walks *from* a qualifying condition *to* the
counted objects; attribution walks *from* a counted object *to* the node it
belongs to. One traversal implementation, driven by ontology link
metadata, serves both — which is what makes "any object, any reach" a
configuration claim instead of a coding claim.

### 1.3 Where the work lands

The kernel spans two repositories at most: the spec repository (schemas,
`judge`, the function API) and the pipeline (`resolve`, `attribute`,
`measure`, `diff`, the build). Determine where the code holds individual
objects in its hands — that is where lineage is written, because one step
after aggregation the membership is gone for good:

| Case | What you find | Capture goes |
|---|---|---|
| 1 | KPI functions filter an ObjectSet of individual objects and aggregate it | the spec repo |
| 2 | The spec repo only defines schema / receives pre-aggregated rows | the pipeline |
| 3 | Mixed | schema everywhere, capture where the objects are |

The movers diff always belongs to the snapshot build: only the build has
cycle *t* and *t−1* side by side.

### 1.4 Cadence and retention — settled

**Weekly.** One build per cycle, one row per KPI × axis × scope. **All
snapshots retained indefinitely.** The storage footprint is cumulative,
which is why the roster (§5.7) exists.

---

## 2. The spine: structure, axes, and dated belonging

This section is the heart of "any level". It builds on the organisation
model the platform already has — one self-referencing node type with
levelled, **dated** parent edges and precomputed paths — and extends it
to every axis a KPI is read along.

### 2.1 Axes and nodes

An **axis** is one hierarchy along which objects are rolled up. Each axis
has a node type, a level list, and dated parent edges:

| Axis | Levels (top → bottom) | Node type | Notes |
|---|---|---|---|
| `org` | programme → project/aircraft → work package → team | `OrgNode` (existing) | Add `programme` and `project` to its `LEVELS`; the design already anticipates this. |
| `supplier` | supplier → contract/work share | `SupplierNode` | Cross-cuts `org`: one supplier serves many programmes. |
| `aircraft` | fleet → programme → MSN | `AircraftNode` | Where the object naturally attaches to an airframe (deliveries, NCs on an MSN). |

Every axis has an implicit root `overall` (no attribution) and an implicit
leaf-side sink **`unassigned`** (§2.4). Axes are declared once, in a small
registry; a KPI lists which axes it is published on.

**Belonging is on the edge, dated — never on the node.** A work package
that moves programmes in September has two edges with `valid_from` /
`valid_to`, and the precomputed path table has one row per node per
period. This is what makes replay honest: last quarter's number stays
last quarter's number under a reorg.

### 2.2 Attribution: from a counted object to a node

For each `(object_type, axis)` pair the KPI is published on, the
definition declares the path:

```json
"attribution": {
  "org":      { "path": ["NonConformity -> WorkPackage"],        "mode": "one" },
  "supplier": { "path": ["NonConformity -> Part -> Supplier"],   "mode": "all" },
  "aircraft": { "path": ["NonConformity -> Aircraft"],           "mode": "one" }
}
```

Attribution is **precomputed per cycle as a link table** — one row per
object × axis × node, with `resolved_at_level` — exactly the pattern of
the existing engineering link tables. A rollup at any level is then a
`GROUP BY` on the path columns. **Nothing walks a graph at query time**,
in an OSDK consumer, an AIP prompt, or a Workshop app.

Attribution reads the structure **as of the cycle date** (`*_at_event`),
never as it stands today, unless a KPI explicitly opts into `current`.
The default is the replayable reading.

### 2.3 Multi-attribution — declare it, then reconcile accordingly

An object can reach more than one node on an axis (a part used on two
programmes). `mode` says what happens, and the reconciliation gate (§8,
gate 8) follows from it:

| `mode` | Object counts in | Σ children vs overall |
|---|---|---|
| `one` | exactly one node — the first by a deterministic rule (declared: nearest, primary link, lowest id) | equal, exact |
| `split` | every reachable node, contribution divided equally (or by a declared weight field) | equal, exact |
| `all` | every reachable node, full contribution each | **not equal**, by design — documented, never "fixed" |

Where the path is many-to-one (the common case), `mode` is moot and
`one` is written for clarity.

### 2.4 `unassigned` and coverage — the most expensive blind spot

An object that resolves to **no node** on an axis does not vanish. It is
attributed to the axis's `unassigned` sink, which is a real scope with a
real row every cycle, and every `overall` row carries

```json
"attribution_coverage": { "org": 0.87, "supplier": 0.64 }
```

The reason this is non-negotiable: an unattached object does not show up
as an error — it silently drops out of the rollup. Fewer objects means
fewer late items means a *better* KPI. A data problem that looks like
good news is the most expensive kind. Coverage is therefore a number that
belongs next to every rolled-up number, with a floor (§8, gate 9), and a
drop between cycles is reported on `movers_summary` as `coverage_delta`.

### 2.5 Configuration inheritance along the axis

Targets, ramps, thresholds and caps are set **at the highest node where
they hold** and inherited downward. A target set on programme `A350`
applies to every project, work package and team under it unless a lower
node overrides. The build resolves the *effective* configuration per
scope and stores `config_resolved_from: "<node_id>"` on the snapshot row,
so a reader can see where a number's expectation came from.

Without this, a catalog of fifty KPIs across five hundred work packages
is twenty-five thousand config rows and the engine is unmaintainable on
day one. With it, most KPIs carry one config row and a handful of
overrides.

---

## 3. The KPI definition — one object that says everything

A KPI is a `KpiDefinition` object. Everything the engine needs to build it
at every level is inside it; there is no code per KPI. This is the shape;
§5 gives the contract of each block.

```json
{
  "kpi_id": "open_nc_count",
  "name": "Open non-conformities",
  "description": "NCs in open or in-review state on aircraft of fleets delivering in the reporting year",
  "owner": "quality.lead@…",
  "source": { "system": "QMS", "steward": "qms.steward@…", "sla_hours": 168 },
  "unit": "count", "display_format": "integer",
  "direction": "down",

  "population": {
    "anchor": { "object_type": "Fleet",
                "filter": { "delivery_date": { "op": "in_year", "value": "$reporting_year" } } },
    "path":   ["Fleet -> Aircraft -> NonConformity"],
    "target": "NonConformity",
    "exclusions": { "msn_type": { "op": "ne", "value": "test" } }
  },
  "date_basis": "forecast",
  "date_anchor": "snapshot",

  "mechanism": "count",
  "mechanism_params": { "bad_condition": { "status": ["open", "in_review"] } },

  "attribution": {
    "org":      { "path": ["NonConformity -> WorkPackage"], "mode": "one" },
    "aircraft": { "path": ["NonConformity -> Aircraft"],    "mode": "one" }
  },
  "axes": ["org", "aircraft"],

  "judgment": {
    "policy": "target_ramp",
    "target": 0,
    "ramp": { "from": "MG5", "to": "MG7", "curve": "s_curve" },
    "warning_band": 0.05,
    "critical_within_days": 30
  },
  "judgment_overrides": [
    { "node": "A350", "target": 40 }
  ],

  "exception_cap": 200,
  "materially_changed": { "status_transition": true, "epsilon": 0 },

  "definition_version": 3,
  "is_active": true
}
```

**Admission** (§9) is the schema of this object: a definition missing
owner, source + steward, population, mechanism, at least one axis with a
validated attribution path, and a judgment policy does not validate and
therefore never computes.

---

## 4. Time semantics

- **Immutability.** Snapshots are never edited. A correction is a new row
  superseding the old; both are retained.
- **As-of replay.** Any answer can be reproduced for a past cycle from
  stored rows alone. This works because everything judgment depends on is
  either versioned (definitions, targets, populations) or stored on the
  row (`judgment_context`).
- **Windows bind to the snapshot** (`date_anchor: "snapshot"`) unless a
  KPI explicitly opts into `today`. Replayable is the default.
- **Structure binds to the snapshot** too: attribution reads the dated
  edges as of the cycle date.
- **`$reporting_year` is the calendar year** of the anchor date — settled.

---

## 5. Contracts

### 5.1 Filter grammar

One machine-executable filter representation. **Do not invent a new
filter language if the codebase has one** — serialise the existing one,
provided it is self-describing and re-executable. Otherwise use this
minimal closed grammar:

```json
{ "field": <predicate>, "field2": <predicate> }        // implicit AND
```

| Predicate | Meaning |
|---|---|
| `"open"` / `42` / `true` | equality |
| `["open", "in_review"]` | IN |
| `{ "op": "ne" \| "lt" \| "lte" \| "gt" \| "gte", "value": v }` | comparison |
| `{ "op": "between", "value": [lo, hi] }` | inclusive range |
| `{ "op": "in_year", "value": "$reporting_year" }` | date within a year |
| `{ "op": "within_days", "value": [-7, 0] }` | relative window around the anchor date |
| `{ "op": "is_null" \| "not_null" }` | nullity |
| `{ "op": "age_gt", "value": 30 }` | (anchor date − field) > N days |

Variables, resolved by `resolve()`, never by the caller:

| Variable | Resolves to |
|---|---|
| `$snapshot_date` | the cycle's date |
| `$today` | build wall-clock — legal **only** with `date_anchor: "today"`; fail the KPI's build otherwise |
| `$reporting_year` | calendar year of the anchor date |

Date fields are named by **role**, not by column: a filter says
`delivery_date`, and the `ObjectTypeProfile` (§5.3) maps that role to the
plan, forecast or baseline column according to `date_basis`.

### 5.2 Paths

A path is a list of link hops, each naming the ontology link type by its
API name:

```json
["Fleet -> Aircraft", "Aircraft -> NonConformity"]      // canonical
["Fleet -> Aircraft -> NonConformity"]                   // shorthand, expanded at admission
```

Rules: every hop must exist in ontology metadata with a known
cardinality; traversal **deduplicates** by target id (an object reachable
by two routes counts once); a path is validated at admission and
re-validated each build (a link type renamed under the engine fails the
build for that KPI, loudly, rather than resolving to zero and reading as
improvement).

### 5.3 `ObjectTypeProfile` — how a type becomes KPI material

One small registry object per ontology type the engine reads. This is the
whole onboarding cost of a new object type:

```json
{
  "object_type": "NonConformity",
  "primary_key": "nc_id",
  "label_field": "title",
  "status_field": "status",
  "date_roles": {
    "created":   { "plan": "created_at" },
    "due":       { "plan": "due_date", "forecast": "forecast_close_date" },
    "closed":    { "plan": "closed_at" }
  },
  "default_bad_condition": { "status": ["open", "in_review"] },
  "links": ["NonConformity -> Aircraft", "NonConformity -> Part", "NonConformity -> WorkPackage"]
}
```

`date_roles` is what makes `date_basis` generic: a KPI says `due`, the
profile says which column that is under `plan` vs `forecast`. A role with
no `forecast` column falls back to `plan` and the fallback is noted on
the recipe.

### 5.4 Population

```json
{
  "anchor": { "object_type": "…", "filter": { … } },
  "path":   [ … ],
  "target": "…",
  "exclusions": { … }
}
```

`anchor.object_type` is the type the qualifying condition **actually sits
on** — the hard populations are the ones where it is not the counted
type. `path` is empty when the condition sits on the counted object.
`exclusions` is the "test articles, cancelled orders" folklore, written
down. Simple and linked cases share one shape and one code path.

Population content is a **business sign-off** by the KPI owner (§11).
Every change lands in `KpiPopulationHistory`.

### 5.5 Mechanisms

Closed set. A KPI picks one and parameterises it.

| `mechanism` | `mechanism_params` | value | per-object `contribution` | ranks? | exceptions are |
|---|---|---|---|---|---|
| `count` | `bad_condition?` | \|S_bad\| | 1 | no | the counted objects |
| `distinct_count` | `field`, `bad_condition?` | distinct values of field | 1 per first object per value | no | one object per value |
| `ratio` | `numerator` (condition **or** a second population), `denominator` = population | num / den | 1 (count space) | no | the failing numerator side |
| `sum` | `field`, `bad_condition?` | Σ field | field value | yes | top-N by contribution |
| `weighted_sum` | `field`, `weight_field` | Σ w·x | w·x | yes | top-N |
| `age` | `date_role`, `agg: mean\|median\|p90`, `unit: days` | agg of (anchor date − date) over S | its age | yes | oldest first |
| `throughput` | `event_date_role`, `window_days`, `agg: count\|sum(field)` | events in window | 1 or field | as `agg` | the events |
| `percentile` | `field`, `p` | p-th percentile | **none** | — | objects at/beyond the boundary |
| `window_aggregate` | `inner`, `window_cycles`, `agg` | agg of inner over N cycles | inherited | inherited | inherited |
| `derived` | `expr`, `inputs: { name: "kpi:<id>" }` | expression over same-scope, same-cycle input KPI values | **none** at object level | — | none; `contributors` = the input rows |

Three notes:

- **`ratio` with two populations** (`numerator` given as a population,
  not a condition) covers "closed this week / opened this week" and every
  other KPI whose top and bottom are different sets. Lineage carries both
  recipes.
- **`derived`** is how composite indices, SPI/CPI, first-time-right and
  every "KPI of KPIs" enter. It reads *stored* input rows for the same
  scope and cycle — never recomputes them — and its `contributors` are
  those rows `(kpi_id, scope, cycle, value)`. An input missing for a
  scope makes the derived row **absent**, not zero. `explain()` recurses
  into inputs.
- **`ranks?`** is a property of the mechanism and decides whether an
  oversized exception set may be truncated (§5.7). Where every object
  contributes 1, it may not.

Reconciliation (gate 3) is **exact** for count-space mechanisms, exact up
to the denominator effect for `ratio`, approximate and documented for
weighted mechanisms, not applicable for `percentile` and `derived`.

### 5.6 Judgment — policies, and milestone-resolved ramps

Judgment is a **pure function of (value, policy config, context)**. The
context is resolved by the build *per scope* and stored on the row, so
any layer can re-run it and get the same answer.

#### 5.6.1 Policies

| `policy` | Needs | Produces severity from |
|---|---|---|
| `target_ramp` | `target`, `ramp {from, to, curve}`, `warning_band`, `critical_within_days` | value vs the expected value on the ramp at the cycle date, for **this scope's** gate dates |
| `thresholds` | `bands: [{ "lte": 0.85, "severity": "critical" }, …]` | absolute bands, direction-aware |
| `rollup` | `agg: worst \| share_critical \| weighted`, optional `threshold` | the judgments of the scope's **children** on the same axis |
| `trend` | `cycles`, `worsening_rate` | the slope over the last N cycles, direction-aware |

The severity enum is exactly **`achieved | on_track | warning | critical
| overdue`** — settled. No policy emits anything else.

#### 5.6.2 Ramps resolve against Milestone objects, per scope

A ramp names milestones **symbolically**. The build resolves them for
each scope from the Milestone objects linked to that scope's node:

```
resolve_ramp(scope, judgment, cycle_date):
    node    = scope.node                          // e.g. project "A350-1041"
    ms_from = milestone(node, code = judgment.ramp.from)   // "MG5"
    ms_to   = milestone(node, code = judgment.ramp.to)     // "MG7"
    // walk up the axis until a node owns milestones of that code
    start   = date_of(ms_from, date_basis)        // baseline | plan (=target) | forecast
    end     = ms_to.achieved_date ?? date_of(ms_to, date_basis)
    return { from: ms_from.id, to: ms_to.id, start, end, basis, achieved: ms_to.achieved_date != null }
```

Milestone objects carry four dates — baseline, target (committed plan),
forecast, achieved. `date_basis` selects `plan` → target or `forecast` →
forecast (and `baseline` is allowed explicitly). An achieved gate ends the
ramp at its achieved date regardless of basis.

This is what lets **one definition serve every project**: "maturity ramps
from MG5 to MG7" is written once; project A resolves to its own dates,
project B to its own. At a scope whose node owns no milestone of that
code (a programme with many projects, each with its own MG7), `target_ramp`
cannot resolve and the definition must either **override to `rollup`**
at that level or carry an explicit override with dates. Admission checks
this per scope and reports which scopes resolved how.

#### 5.6.3 The `target_ramp` algorithm — reference implementation

```
judge_target_ramp(value, cfg, ctx):        // ctx = resolved ramp + cycle_date
    down       = cfg.direction == "down"
    target_met = down ? value <= cfg.target : value >= cfg.target
    if target_met:                                   return "achieved"
    if ctx.ramp == null:                             return "warning"     // no ramp resolvable
    days_to    = ctx.ramp.end − ctx.cycle_date
    if days_to < 0:                                  return "overdue"
    t          = clamp01((ctx.cycle_date − ctx.ramp.start) / (ctx.ramp.end − ctx.ramp.start))
    expected   = cfg.target * CURVE[cfg.ramp.curve ?? "linear"](t)
    on_track   = down ? value <= expected * (1 + cfg.warning_band)
                      : value >= expected * (1 − cfg.warning_band)
    if on_track:                                     return "on_track"
    if days_to <= cfg.critical_within_days (30):     return "critical"
    return "warning"
```

Curves — settled, match them exactly:

| `curve` | `CURVE(t)` |
|---|---|
| `linear` (default) | `t` |
| `s_curve` | `1 / (1 + e^(−12·(t − 0.5)))` |
| `front_loaded` | `√t` |
| `back_loaded` | `t²` |

#### 5.6.4 `judgment_context` — stored, and compared cycle to cycle

Every snapshot row stores what it was judged against:

```json
"judgment_context": {
  "policy": "target_ramp", "config_resolved_from": "A350",
  "definition_version": 3, "target": 40,
  "ramp": { "from": "MS-2004", "to": "MS-2009", "start": "2026-01-15",
            "end": "2026-11-30", "basis": "forecast", "achieved": false },
  "expected_value": 55.2, "gap_to_expected": +3.8
}
```

And the build compares it with the predecessor's. **A ramp whose resolved
end date moved** — because MG7's forecast slipped six weeks — lowers the
expected value at today and can turn a KPI green with no work done. That
is the same integrity problem as a silent target re-baseline, one layer
down. So the row carries

```json
"context_changed": { "ramp_end": { "from": "2026-10-15", "to": "2026-11-30" },
                     "target": null, "config_resolved_from": null }
```

whenever anything in the context differs from the previous cycle, and
`why_changed()` reports it *before* the movers: "expected value fell
because MG7 forecast moved +46 days" is the first line of that answer,
not a footnote.

### 5.7 Lineage: `contributors`, `exceptions`, `roster`

Written **at the aggregation step, in one pass**, while the contributing
set is in hand — never reconstructed afterwards.

**`contributors`** — the resolved recipe, plus the count:

```json
{ "object_type": "NonConformity", "filter": { … resolved, variables substituted … },
  "path": [ … ], "count": 4127,
  "date_basis_fallbacks": ["due: no forecast column, used plan"] }
```

**`exceptions`** — the materialised bad side:

```json
[ { "id": "NC-4412", "label": "Bracket torque out of spec", "contribution": 1,
    "nodes": { "org": "WP-3.1", "aircraft": "MSN-0421" } } ]
```

Each entry carries its node per axis, so a reader at the overall level
can see *where* an exception sits without a second query.

**Exception sets scale with scope — assume thousands at the top.** The
same KPI has a bad side in the thousands at `overall`, hundreds per
programme, tens at a leaf. The materialisation decision is therefore made
**per row**:

| Bad set at this scope | Write |
|---|---|
| ≤ `exception_cap` | full `exceptions` |
| > cap, mechanism **ranks** | top-N `exceptions` sorted `(contribution DESC, id ASC)` + `exceptions_truncated: true` + `roster` |
| > cap, mechanism does **not** rank | **no `exceptions`** + `roster` |

The third row is a correctness rule. When every object contributes 1,
"top 200" is an arbitrary 200 presenting as a curated worst-200, and with
4 000 equal sort keys it is not even a stable 200 — the determinism gate
fails, legitimately. Truncate only where contribution discriminates;
otherwise omit, and let the reader descend the axis, which is what they
wanted anyway. The id tiebreak on ranked truncation is mandatory.

**`roster`** — ids only:

```json
"roster": ["NC-4412", "NC-4418", …]
```

Written whenever `exceptions` is truncated or omitted. Its only purpose
is to make the next cycle's diff exact: membership at *t−1* is
unrecoverable once the cycle ends, and without a roster a large scope
would have no diff or a wrong one. ~12 bytes per id against ~100 for a
full entry; a 4 000-object scope costs ~50 KB per row instead of
~400 KB. The roster is also what makes `traceObject()` (§7) an index
lookup instead of a scan.

`exception_cap` defaults to 200 — settled as the *starting* value; WP3
reports the bad-set size at three scope levels so it is sized against
reality.

### 5.8 Movers — with a reason on every entry, both directions

```json
"movers": {
  "entered": [ { "id": "R-1188", "delta": "+1.4", "reason": "new" },
               { "id": "NC-5102", "delta": "+1",  "reason": "population_change" },
               { "id": "NC-4977", "delta": "+1",  "reason": "scope_change", "from_node": "WP-2.4" } ],
  "left":    [ { "id": "R-0902",  "delta": "-0.3", "reason": "resolved" },
               { "id": "NC-4418", "delta": "-1",   "reason": "population_change" },
               { "id": "NC-4501", "delta": "-1",   "reason": "scope_change", "to_node": "WP-3.2" } ],
  "changed": [ { "id": "R-1041", "from": "warning", "to": "critical", "delta": "+2.1" } ],
  "denominator_effect": 0.0,
  "movers_truncated": false,
  "reconciles": true
}
```

The reason enum is closed and **symmetric** — entering because the
population grew is no more a deterioration than leaving because it
shrank is an improvement:

| `reason` | On `left` means | On `entered` means |
|---|---|---|
| `resolved` / `new` | the condition cleared — **real improvement** | the condition newly holds — **real deterioration** |
| `population_change` | the object left the population (a fleet slipped delivery into next year and took its NCs with it) — **not** improvement | the object joined the population — **not** deterioration |
| `scope_change` | still in the population, still failing, **now attributed to another node** (a reorg moved the work package) — **not** improvement here, not deterioration there | the mirror |

`scope_change` is the reason the dated edges exist: a departmental KPI
that jumps eight points has to say whether performance changed or the
boundary did. The build classifies it by re-evaluating the object against
the current population (still in? → not `population_change`) and the
current attribution (same node? → not `scope_change`); what remains is
`resolved` / `new`.

**Movers are bounded by change, not by population.** 4 000 open NCs still
see tens enter and leave in a week — so object-level movers are
materialised even at scopes that store no `exceptions`, computed from the
rosters. Cap them at `exception_cap` for the pathological case, sorted
`(|delta| DESC, id ASC)`, with `movers_truncated: true`. The summary below
stays exact regardless.

`denominator_effect` — for `ratio`, the cycle delta decomposes into a
numerator term the movers explain exactly and a term from the denominator
changing: `Δ = (N_t − N_{t−1})/D_t + N_{t−1}·(1/D_t − 1/D_{t−1})`. The
second term is a real effect of the population growing or shrinking and
is **reported in its own field**, never absorbed.

#### 5.8.1 `movers_summary` — the answer that always fits

```json
"movers_summary": {
  "entered": 142, "entered_by_reason": { "new": 118, "population_change": 20, "scope_change": 4 },
  "left":    89,  "left_by_reason":    { "resolved": 61, "population_change": 24, "scope_change": 4 },
  "changed": 17, "net": 53,
  "coverage_delta": { "org": -0.03 },
  "context_changed": true
}
```

**Written at every scope, every cycle, without exception.** Counts are
cheap at any size and come from the rosters, so they are exact even where
no list was stored. This is a complete, honest answer to "why did the
fleet-wide number move": *53 net — 118 genuinely new, 61 genuinely
resolved; the rest is the population and the org moving, and the ramp
end shifted*. Gate 3 reconciles against the summary, so the sharpest
validation the engine has holds at every scope, including the ones too
large to list.

---

## 6. Data model

| Object | One row per | Purpose |
|---|---|---|
| `KpiDefinition` | KPI | the §3 object |
| `KpiConfigOverride` | KPI × node | judgment / cap overrides below the definition's default (§2.5) |
| `ObjectTypeProfile` | ontology type | §5.3 |
| `AxisDefinition` | axis | node type, levels, edge dataset |
| `KpiAttribution` | object × axis × cycle | the precomputed attribution link table (§2.2) |
| `KeyPerformanceIndicator` (result row) | KPI × axis × scope | current value + lineage |
| `KeyPerformanceSnapshot` | KPI × axis × scope × cycle | the immutable record |
| `KpiTargetHistory` / `KpiPopulationHistory` / `KpiDefinitionHistory` | change | audit trails |
| `AlertRule` | rule | evaluated server-side each cycle |
| `KpiDataHealth` | source × cycle | freshness vs SLA, validation results |

### 6.1 Columns to add — result row and snapshot

All optional; existing columns (`result_id`, `scope_id`, `kpi_value`,
`numerator`, `denominator`, `snapshot_date`, …) untouched. Where the
existing schema has `aggregation_dimension` / `aggregation_key`, map
`axis` / `node_id` onto them and record the mapping in the return form.

| Column | Type | Default | Meaning |
|---|---|---|---|
| `axis` | string | — | which axis this row is on (`org`, `supplier`, …); absent = overall |
| `node_id` | string | — | the spine node; `unassigned` for the sink |
| `node_level` | string | — | the node's level on its axis |
| `severity` | string enum | — | **frozen at build time** (§5.6) |
| `judgment_context` | json | — | §5.6.4 |
| `context_changed` | json | — | §5.6.4 |
| `config_resolved_from` | string | — | §2.5 |
| `definition_version` | int | — | pins the definition the row was computed under |
| `contributors` | json | — | §5.7 |
| `exceptions` | json list | — | §5.7 |
| `exceptions_truncated` | boolean | false | mandatory whenever the cap was hit |
| `roster` | json list | — | §5.7 |
| `movers` | json | — | §5.8 (snapshot only) |
| `movers_summary` | json | — | §5.8.1 (snapshot only, **always**) |
| `attribution_coverage` | json | — | §2.4 (overall rows) |
| `stale` | boolean | false | published against a source that missed its SLA |
| `low_coverage` | boolean | false | coverage below the KPI's floor (§8, gate 9) |
| `annotation_text` / `_author` / `_date` | | — | owner's explanation for this cycle |

Snapshot **every scope on every axis**, `unassigned` included. Never
derive one scope by scaling another.

---

## 7. The function API — what AIP Logic calls

The delivery layer is a small set of typed functions over stored rows.
They are the **only** interface AIP Logic, agents, Workshop and any
dashboard use. **None of them computes a KPI.** A question that needs
math not present in the rows is a request for a new snapshot column, not
a new function. Every function returns ids the caller can cite.

| Function | Returns | Backed by |
|---|---|---|
| `getKpi(kpi_id, axis?, node_id?, as_of?)` | one snapshot row (overall when no axis) | lookup |
| `getSeries(kpi_id, axis?, node_id?, from, to)` | rows over cycles | lookup |
| `explain(kpi_id, axis?, node_id?, as_of?)` | `{ value, recipe, exceptions \| "roster only (n)", judgment_context, coverage }`; for `derived`, recurses one level into inputs | lookup |
| `whyChanged(kpi_id, axis?, node_id?, as_of?)` | `{ context_changed, movers_summary, movers, annotation }` — **context first**, then movers | lookup |
| `rankScopes(kpi_id, axis, level, by: severity \| delta \| value \| coverage, as_of?, n?)` | the worst / most-moved N nodes at a level | lookup + sort |
| `compareScopes(kpi_id, axis, node_a, node_b, as_of?)` | both rows side by side, both contexts | lookup |
| `traceObject(object_type, object_id, as_of?)` | every `(kpi_id, axis, node_id, role: exception \| member, since_cycle)` the object appears in — **inverted lineage** | roster index |
| `listKpis({ object_type?, axis?, level?, owner?, mechanism?, policy? })` | catalog entries; "which KPIs count NonConformity by supplier?" | catalog |
| `expectedAt(kpi_id, axis?, node_id?, date)` | the expected value on the resolved ramp at a date | `judge` context |
| `judge(value, kpi_id, axis?, node_id?, as_of?)` | a severity for a hypothetical value under that scope's stored context — "what would 38 have been?" | pure `judge` |
| `resolveSet(kpi_id, axis?, node_id?, as_of?, page?)` | the *current* objects the stored recipe resolves to — the drill-down; executes the recipe, which is not reconstruction | `resolve` |
| `coverage(kpi_id, axis, as_of?)` | coverage per level + the `unassigned` row | lookup |
| `dataHealth(kpi_id, as_of?)` | source freshness and gate results for the row | lookup |

Design rules for the functions:

- **Scope arguments are uniform** across all functions: `(axis?, node_id?, as_of?)`. Absent axis = overall; absent `as_of` = latest published cycle.
- **Every response carries provenance**: `cycle`, `definition_version`, `config_resolved_from`, and `stale` / `low_coverage` / `exceptions_truncated` / `movers_truncated` flags verbatim. An agent that drops a flag is misquoting the engine.
- **Absent means absent.** A function never fabricates: no lineage → `"no lineage recorded"`; no predecessor → no movers; input missing → derived row absent.
- **Paginate the big ones** (`resolveSet`, `exceptions` over a roster) rather than cap silently.
- **Tool descriptions ship with the functions.** Each carries a one-paragraph description written for an agent — what question it answers, what it never does — so registering them as AIP tools is a copy, not an authoring task.

---

## 8. The build and its gates

### 8.1 Stages

```
load definitions       →  validate paths/profiles against ontology metadata (gate 7)
resolve populations    →  per KPI: recipe + set, variables bound to the cycle date
attribute              →  per axis: object → node as of cycle date; coverage; unassigned
fan out                →  per KPI × axis × node (incl. overall, unassigned): the scoped set
compute + capture      →  value, num, den, contributors, exceptions | roster
resolve context        →  effective config (inheritance), ramp from milestones, children
judge                  →  severity + judgment_context
snapshot               →  append rows (immutable)
diff                   →  movers from rosters, reasons classified, summary; context_changed
validate               →  gates
publish                →  visible to L5; alerts evaluated
```

A stage that fails **gates the publish for the affected rows**; the rest
publish. The engine prefers an honest gap over a wrong number.

### 8.2 Gates

Run on real data every cycle, not only in CI.

| # | Gate | Assertion | On failure |
|---|---|---|---|
| 1 | Recipe ↔ number | `contributors.count == denominator` (or the documented population size), same scope and cycle | Block the row — they come from the same arguments; a mismatch is a broken build. |
| 2 | Movers ↔ sets | Against the rosters: `entered = M_t \ M_{t−1}`, `left = M_{t−1} \ M_t`, exactly; every entry has a reason; summary counts match the lists where lists exist | Block the row. |
| 3 | Movers ↔ delta | Against `movers_summary`: `entered − left` (+ `denominator_effect`) equals the cycle delta. **Exact** for count-space mechanisms at every scope | Block the row; never "rounding". Weighted mechanisms document the tolerance; `percentile` / `derived` exempt. |
| 4 | Cap honesty | `len(exceptions) ≤ cap`; `truncated` iff the set was larger; a truncated list is sorted `(contribution DESC, id ASC)` and its mechanism ranks; an oversized unranked set has **no** list and a roster | Block the row — a truncated unranked list is what this gate exists to catch. |
| 5 | Determinism | Re-running the cycle from the same inputs hashes identically, lineage and context included | Investigate before publishing; non-determinism invalidates replay and every other gate. |
| 6 | Freshness | Every source met its SLA (`KpiDataHealth`) | Publish with `stale: true` **and** notify the steward. |
| 7 | Path validity | Every hop in every population and attribution path exists in ontology metadata with known cardinality; every date role in use exists on its profile | Block the KPI. A renamed link resolving to zero would read as improvement. |
| 8 | Attribution ↔ overall | Under `mode: one` / `split`: Σ children == overall, exact for count-space; `unassigned` included in the sum | Block the axis for that KPI. Under `all`: not asserted, by declaration. |
| 9 | Coverage floor | `attribution_coverage[axis] ≥ floor` (default 0.8, per KPI) and `coverage_delta ≥ −0.05` | Publish with `low_coverage: true`, notify the owner; a coverage *drop* is reported on the summary regardless. |
| 10 | Ramp resolvable | For `target_ramp` scopes: both milestones resolved, `start < end` | The scope falls back to `warning` with `ramp: null` in its context and the scope is listed in the build report. |
| 11 | Derived inputs | Every input row exists for the scope and cycle | The derived row is absent, never zero. |
| 12 | Context honesty | `context_changed` is set iff the stored context differs from the predecessor's | Block the row. |

Gate 3 on a count pilot remains the sharpest single test the engine has;
gate 8 is its structural twin across the axis. Together they catch the
two ways a rollup lies: by mis-diffing and by mis-attributing.

---

## 9. Governance, made mechanical

- **Admission is schema.** A `KpiDefinition` missing owner, source +
  steward, population, mechanism, at least one axis with a valid path, or
  a judgment policy does not validate and never computes. The council's
  checklist is the required-field set.
- **One write path to semantics.** Definition, target, population and
  override changes are diffs to configuration, reviewed like code, logged
  as decisions, versioned in the histories.
- **Duplicate resistance.** `listKpis` searches by population, mechanism
  and axis, so "two KPIs counting the same set" is a query, not
  archaeology.
- **The integrity flags are consumers' rights** — `exceptions_truncated`,
  `movers_truncated`, `population_change` and `scope_change` reasons,
  `context_changed`, `stale`, `low_coverage`, `unassigned` rows — asserted
  by gates, never left to vigilance.

---

## 10. Work plan

Ordered; each package independently deployable. **Prove the engine
vertically first — every primitive, one KPI, one axis — then widen.**

### WP0 — Preflight (no code)
Run the §1.3 decision tree. Inventory existing columns against §6.1 and
the existing org link tables against §2.2. Confirm the `org` axis can be
extended with `programme` / `project` levels. Get §11's open items
answered.
**Done when** return form items 1–3 can be filled in.

### WP1 — Schema, additive
Every column in §6.1; `KpiDefinition`, `ObjectTypeProfile`,
`AxisDefinition`, `KpiConfigOverride`, the histories.
**Done when** all existing tests pass untouched and a row without new
fields is byte-identical.

### WP2 — Kernel: `resolve` + `attribute` on the `org` axis
Filter grammar (or the existing one), paths validated against ontology
metadata, profiles with date roles, traversal with dedupe, variable
binding; attribution as a precomputed dated link table with
`unassigned` and coverage, reusing the existing org path machinery.
**Done when** for the pilot KPI, `resolve()` reproduces the denominator
the current pipeline computes, and Σ over `org` nodes + `unassigned`
equals it (gate 8).

### WP3 — Capture, pilot KPI, `org` axis
A plain `count` over one object type, simple population, bounded leaf
sets — confirm the choice with the requester. Materialisation per row
(§5.7), rosters where needed.
**Done when** gates 1, 4, 7, 8 pass at every `org` scope on real data,
and the bad-set size at overall / widest programme / a leaf is reported.

### WP4 — Judgment with milestone-resolved ramps
`judge` as a pure function; the four policies; ramp resolution from
Milestone objects per scope with axis walk-up; config inheritance;
`judgment_context` stored; `rollup` at scopes that own no gate.
**Done when** every scope of the pilot has a stored severity and context,
gate 10 reports which scopes resolved a ramp and which rolled up, and a
replay of the previous cycle reproduces its severities from stored rows.

### WP5 — Snapshot build + movers
Diff from rosters; the three-way reason classification including
`scope_change` from the dated edges; `movers_summary` everywhere;
`context_changed`; `denominator_effect`.
**Done when** gates 2, 3, 12 pass across two consecutive real cycles, and
the first `population_change` and first `scope_change` observed are
verified by hand against the object that moved.

### WP6 — Gates and determinism harness
All twelve gates as build stages with row-level blocking; the replay
harness for gate 5.
**Done when** a deliberately corrupted input trips exactly the expected
gate on exactly the affected rows.

### WP7 — Function API
Every function in §7, with tool descriptions, over stored rows;
`traceObject` on a roster index; `resolveSet` paginated.
**Done when** an AIP Logic flow can answer "why is open NCs red for
A350 and what changed?" by calling `getKpi` → `explain` → `whyChanged`
and citing ids, with no computation outside the functions.

### WP8 — Second axis, second pilot
The `supplier` axis for the pilot KPI (`mode: all` — gate 8 not
asserted, coverage asserted); then a traversal-population KPI (anchor ≠
target).

### WP9 — `derived`, `ratio` with two populations, `age`, `throughput`
One KPI each, on top of proven primitives. Gate 11 for derived.

### WP10 — Breadth
Migrate the catalog one mechanism at a time. **Do not backfill lineage or
rosters** — they start when capture starts.

### The demo that ends every stage
Ask: "why is this red for this project?" → the objects, with nodes, and
the ramp it was judged on. "Why did it change?" → context changes first,
then movers with reasons. "Show me last month" → same answers from stored
rows. If the demo needs an engineer to explain it, the stage is not done.

---

## 11. Settled, and still open

### Settled — do not re-open

| Decision | Value | Where |
|---|---|---|
| Cadence | weekly | §1.4 |
| Retention | all snapshots, indefinitely | §1.4 |
| `$reporting_year` | calendar year | §4 |
| Curves | `linear`, `s_curve`, `front_loaded`, `back_loaded`, formulas as given | §5.6.3 |
| Severity enum | the five values, exactly | §5.6.1 |
| Exception cap | start at 200; scale handled by rosters, not the cap; unranked oversized sets are omitted, never truncated | §5.7 |
| Time binding | snapshot-anchored windows and snapshot-dated structure by default | §4 |

### Open — needs a human

| # | Question | Why it matters |
|---|---|---|
| Q1 | **Pilot KPI** (to be named). | Propose one against WP3's criteria and confirm. |
| Q2 | **"Materially changed"** for the pilot — which transitions, what epsilon. | Defines the `changed` bucket. |
| Q3 | **Population sign-off owner** per KPI. | No sign-off, no population; the owner answers the eight questions below. |
| Q4 | **Axis registry**: confirm `org` / `supplier` / `aircraft`, their levels, and which existing node objects back them. | Attribution paths are written against these. |
| Q5 | **Milestone code vocabulary** (`MG5`, `MG7`, …) and which node level owns them (project? aircraft? both?). | Ramp resolution walks up to the owning level. |
| Q6 | **Coverage floor** default (0.8 proposed) and per-axis exceptions. | Gate 9. |
| Q7 | **`mode` for shared parts** on the `aircraft` / `supplier` axes: `all` (honest, non-additive) or `split`? | Gate 8 follows from it. |

The population content itself is a **business sign-off, not an
engineering decision**. Per KPI, the owner answers: what is counted;
what qualifies it and *on which object that condition sits*; the link
path; plan or forecast dates; today- or snapshot-anchored windows; known
exclusions; the owner and sign-off date. The answers become `population`,
`date_basis` and `date_anchor` verbatim. Vague answers become vague KPIs.

---

## 12. Anti-goals

- **Not real-time.** Cycle-based snapshots are the product.
- **Not a query-time aggregator.** Board numbers are precomputed or they
  are not board numbers. No function in §7 aggregates.
- **Not a formula workbench.** New math is a new mechanism through
  review; that friction *is* the one-definition-per-number guarantee.
- **Not a reconstruction engine.** Nothing answers "which objects?" by
  re-deriving membership after the fact. `resolveSet` executes the stored
  recipe against *current* objects and says so.
- **Not a graph walker at read time.** Attribution is precomputed and
  dated. If a consumer needs a path the table lacks, that is a new
  attribution declaration, not a traversal in a prompt.
- **Not a people-measurement tool.** Axes bottom out at team level; no
  person-level node exists, by schema.

Out of scope for this build: dashboard integration, agent prompts and
configuration, backfilling history, making any field mandatory, and
populations for KPIs beyond the pilots.

---

## 13. Return form

```
1.  Aggregation location (case 1 / 2 / 3), with transform/function names
    where the ObjectSet is held: ...
2.  Fields implemented, FINAL names, objects/tables they live on; every
    rename vs. this document (incl. axis/node_id ↔ aggregation_* mapping): ...
3.  Filter grammar used (§5.1 or existing — which): ...
4.  Axes registered, levels per axis, node objects backing them; how the
    existing org path/link machinery was reused or extended: ...
5.  Pilot KPI, why it meets WP3's criteria; contributors/exceptions/roster/
    movers/summary all emitted? Paste one real snapshot row at a leaf
    scope AND one at overall, with lineage and judgment_context: ...
6.  Bad-set size at overall / widest programme / a leaf; which scopes
    materialise exceptions, which are roster-only; cap chosen: ...
7.  Attribution coverage per axis on the pilot; size of `unassigned`;
    the top three `unmapped` reasons: ...
8.  Ramp resolution: which scopes resolved MG-codes to milestone objects,
    which rolled up, which fell back to warning — and why: ...
9.  Exact checks used for the three left/entered reasons: ...
10. "Materially changed" as implemented for the pilot: ...
11. Gates implemented (of 12) and what each does on failure; one real
    gate-failure log line: ...
12. Functions implemented (of §7), signatures as deployed, and the tool
    descriptions as registered: ...
13. Earliest cycle carrying movers; earliest carrying rosters: ...
14. Anything you could not implement as described, and why: ...
```

---

## Appendix A — The spine: prerequisite datasets and their contracts

The engine's `attribute` primitive (§1.2) and every axis in §2 assume an
organisation structure that exists **as dated, levelled nodes with
precomputed paths**, and per-source link tables that attach engineering
rows to those nodes. If this environment has them, verify the contracts
below and reuse. If not, build them first — as their own work package,
before WP2 — following these contracts. They are deliberately simple:
one self-referencing node type, dated parent edges, one walk per build.

### A.1 `org_nodes` — one node type, levelled

| Column | Type | Meaning |
|---|---|---|
| `node_id` | string, pk | e.g. `DEP-CABIN`, `WP-3.1`, `PRG-A350` |
| `level` | string enum | from `LEVELS`, top → bottom: `engineering`, `programme`, `project`, `department`, `workpackage`, `team` |
| `name` | string | display |
| `columns` | array<string> **or** array<struct{alias, source_object?, namespace?}> | the alias list: every spelling of this node in every source; an entry with `source_object` applies only there |

**One node type, not one per level.** Department, work package, programme
are the same thing as far as data is concerned: a drawer engineering rows
get sorted into. Adding a level is an edit to `LEVELS` plus a backfill of
`level` on affected nodes; no consumer that ignores the level changes.
**Belonging is not on the node** — it is on the edge.

### A.2 `org_edges` — dated parent edges

| Column | Type | Meaning |
|---|---|---|
| `child_id` | string | |
| `parent_id` | string | |
| `valid_from` | date | |
| `valid_to` | date | `9999-12-31` for open |

Rebuilt each cycle by comparing `org_nodes` against the edges on record
(slowly-changing-dimension handling): close what moved, open what is new.
**The build cadence is the resolution**: without a real effective date
from the source, a move is dated to the build that first noticed it —
supply a real date wherever the source has one. An append-only
`org_changes` dataset records every movement independently, so history
is reconstructable if `org_edges` is ever flattened, and so a KPI that
jumps eight points can say whether performance or the boundary moved.

### A.3 `org_paths` — the walk, written down, per period

| Column | Type | Meaning |
|---|---|---|
| `node_id` | string | |
| `valid_from` / `valid_to` | date | one row per node **per period** |
| `<level>_id` for each level in `LEVELS` | string, nullable | the ancestor at that level; **null below the node's own level** — which is how a consumer tells the grain a row resolved at |
| `path` | string | `PRG-A350/DEP-CABIN/WP-3.1` |

Skipping a level is legal (a node with no counterpart at some level);
**inverting** one is a data error (a department under a work package)
and fails the build, as do orphaned parents, cycles, duplicate ids,
overlapping periods and levels not in `LEVELS`. A structure problem that
does not fail the build drops a node out of every rollup and reads as
good news.

### A.4 `<source>_links` — one link table per engineering source

Built by resolving each source row's assignment column against the alias
index; one ontology link type per (source object type → OrgNode), backed
by a filtered view. Each row carries the path **twice**:

| Column family | Means | Use for |
|---|---|---|
| `*_at_event` | where the node sat on the row's own date | anything with a time axis — the default |
| `*_current` | where it sits today | "my whole portfolio, history included" |

A row whose source has no date cannot have `*_at_event` and is flagged
`event_date_missing` — never silently given today's answer and called
history. Objects **born on the platform** (a risk raised in Workshop)
carry a real `node_id` chosen by their author and get a direct link at
creation; alias resolution repairs uncontrolled strings and must not
touch data that never had that defect. A manual `org_link_overrides`
table beats the alias index, with an audit of `applied` / `redundant` /
`orphaned` so stale overrides do not fire forever.

### A.5 The honesty outputs — publish all four

| Dataset | Read it for |
|---|---|
| `<source>_unmapped` | every source row that resolved to nothing, with `reason` ∈ `missing_value` / `ambiguous_alias` / `no_match` — each one is a row missing from every rollup |
| `<source>_alias_conflicts` | an alias claimed by two nodes; **fails the build** by default — otherwise assignment depends on join order and the wrong rollup looks entirely plausible |
| `<source>_coverage` | `coverage_pct` and `dated_pct` per source — a silent coverage drop looks identical to good news |
| `<source>_override_audit` | what each manual override did |

The engine's `unassigned` scope (§2.4) is `<source>_unmapped` seen from
the KPI's side; its `attribution_coverage` is `coverage_pct`. Build them
once here and the KPI engine inherits them.

### A.6 Extending to a second axis

`supplier` and `aircraft` are the same pattern with their own node type,
`LEVELS`, edges, paths and link tables. Nothing in A.1–A.5 is
organisation-specific except the level names.

**Reference implementation.** The requester holds three transforms —
`org_structure.py`, `org_edges.py`, `workpackage_link.py` (~1 100 lines,
PySpark) — that implement A.1–A.5 exactly as described. Ask for them
rather than writing from scratch.
