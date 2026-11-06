# 03 — Engine algorithms

This is the technical heart of WhyNot. Everything here is deterministic, pure TypeScript.

## Public API

```ts
evaluate(snapshot: ClinicSnapshot, request: BookingRequest, opts: EvaluateOptions): Evaluation

type EvaluateOptions = {
  now: Instant                      // injected clock (epoch minutes UTC)
  alternatives?: { max: number /*3*/, searchDays: number /*14*/, budgetEvals: number /*3000*/ }
  weights?: Partial<CostWeights>    // alternative ranking weights
  trace?: boolean                   // include matrix + search steps (engine inspector)
}

type Evaluation =
  | { outcome: 'bookable', assignment: Assignment, meta: Meta }
  | { outcome: 'blocked', explanation: Explanation, alternatives: Alternative[], adminFix: AdminFix | null, meta: Meta, trace?: Trace }
  | { outcome: 'invalid_request', errors: ValidationError[], meta: Meta }

Meta = { engineVersion, policyVersionId, evaluatedAt /* = now */, timingsMs }
```

Also exported: `isBookable(snapshot, request, now)` (fast path), `projectForAudience(evaluation, audience)` (doc 04), and the rule catalog.

## Step 1 — Validate the request

Reject before any rule runs: unknown patient/service/practitioner, `start` not on the slot grid, start outside the snapshot window. Output `invalid_request`. These are input errors, not reasons.

## Step 2 — Enumerate candidates

A candidate is `(practitioner p, room r, equipmentUnit e | null)` at the requested interval
`[start, start + duration)`.

- Practitioners: **all** practitioners in the snapshot (unqualified ones produce `practitioner-skill` facts — needed so explanations can say "only X and Y offer this").
- Rooms: all rooms. Equipment: all units if the service needs equipment, else the single value `null`.
- Demo size: ≤ 6 × 6 × 2 = 72 candidates. Ordered canonically by IDs.

## Step 3 — Build the violation matrix

For each candidate `a`, compute `V(a)` = the set of violated **facts** (grounded rule instances, doc 02).

Factorisation (implementation detail, also a big speed-up):

```
V(a) = G ∪ P(p) ∪ R(r) ∪ E(e)
  G    = global/patient facts: clinic-closed-day, clinic-hours, min-notice, max-advance,
         patient-eligibility-*, patient-daily-limit
  P(p) = practitioner facts: skill, hours, break, time-off, busy, buffer, daily-cap, request-practitioner
  R(r) = room facts: capability, maintenance, busy, turnover
  E(e) = equipment facts: type, maintenance, busy
```

Each family is computed once per resource, not per candidate. Occupancy checks use per-resource interval lists sorted by start (binary search for overlaps/adjacency).

**Bookable ⇔ ∃a: V(a) = ∅.** If bookable, the chosen assignment is the first empty-`V` candidate in a preference order: requested practitioner first, then lowest practitioner load that day, then canonical ID order (documented, deterministic).

### Buffer semantics (precise)

For the requested service S at `[s, e)` and an existing practitioner booking B at `[bs, be)` of service SB:

- `practitioner-busy(p, B)` if the intervals overlap.
- Otherwise `practitioner-buffer(p, B)` if `bs ≤ s < be + SB.bufferAfter` (we start inside B's buffer) **or** `s ≤ bs < e + S.bufferAfter` (our buffer runs into B).
- Room turnover is the same with `roomTurnoverMin` on room bookings.

## Step 4 — Explanation = minimum-weight hitting set

### Why a hitting set

If we keep only a set of facts `X` active (treat every other rule as relaxed), the request is blocked iff **every** candidate still violates something in `X`, i.e. `X ∩ V(a) ≠ ∅` for all `a`. So:

- **Sufficient explanation** = a hitting set of the family `F = { V(a) }`.
- **Minimal (irredundant) explanation** = a hitting set where every element is necessary: for each `x ∈ X` there is a candidate with `V(a) ∩ X = {x}`. Removing any reason would make the request bookable — this is exactly a *minimal unsatisfiable subset* in constraint-solving terms.
- **Best explanation** = the minimal hitting set with **minimum total weight** (catalog weights, doc 02). Low-weight facts (closed day, notice, eligibility) are preferred over high-weight ones (room capability), so the engine says the simplest true thing.

Example (H1): `F` after reduction is `{ {buffer(Rao,bk142)}, {timeoff(Mehta,to7)}, {skill(Okafor)}, {skill(Iyer)}, {skill(Harris)}, {skill(Chen)} }` → explanation = all six; the presenter groups the four skill facts into one context line. If `min-notice` were violated, it would be in every `V(a)` and the minimum explanation becomes just `{min-notice}` — the single simplest sufficient reason.

### Structure note (important for scenario design)

With v1's rules, every fact belongs to exactly one dimension — global (`G`), one practitioner, one room, or one equipment unit — so `V(a)` factorises. Consequence: a blocked request is blocked because **a global fact holds, or every practitioner is blocked, or every suitable room is blocked, or every suitable equipment unit is blocked**. A minimal explanation therefore never mixes dimensions (e.g. "room A under maintenance *and* nurse B booked" cannot both be necessary). Multi-reason explanations happen **within** a dimension: "Dr. Rao is in a buffer *and* Dr. Mehta is on leave"; "Treatment Room under maintenance *and* Consult 3 in use".

The general hitting-set algorithm is kept anyway: it is simple, exact, already proven by the properties, and it stays correct when coupled rules arrive (e.g. equipment fixed to a room, a practitioner bound to their own room). A per-dimension shortcut may be added as an optimisation, but its output must equal the hitting set's (tested).

### Algorithm

1. **Reduce:** dedupe sets; drop any set that is a superset of another (hitting the subset hits the superset). Represent facts as indices → sets as bitsets (`Uint32Array`).
2. **Forced facts:** a singleton set `{x}` forces `x` into the answer.
3. **Exact branch-and-bound:** choose the un-hit set with the fewest elements; branch on each of its facts in ascending (weight, factId) order; prune when `currentWeight + lowerBound ≥ bestWeight`. Lower bound: for disjoint un-hit sets picked greedily, sum of their minimum fact weights.
4. **Budget:** cap at 50,000 search nodes. If exceeded: greedy (pick the fact with best `weight / setsHit` ratio, ties by ID), then **prune to irredundant** (remove any fact whose removal still leaves a hitting set, in descending weight order). Mark `explanation.optimal = false`.
5. **Verify (always, cheap):** assert hitting property and irredundancy before returning. A failure is a bug → throw in tests, log + still return in production.

Also returned: `explanation.perCandidateSummary` — for each practitioner, the facts that block *them* (from their best candidate) — used by the staff view and the inspector.

### Tie-breaking and determinism

All iteration orders are by canonical IDs; ties in weight are broken by fact ID string order. Inputs are canonicalised (sorted) on entry, so shuffled arrays produce identical output (tested).

## Step 5 — Alternatives (counterfactual search)

Goal: up to `max` (default 3) **bookable** modified requests that change as little as possible, and that are meaningfully different from each other.

### Variation generators (lazy, in cost order)

| Category | Variation |
| --- | --- |
| `same-day` | Same day, same practitioner constraint, other start times on the grid (alternate later/earlier: +15, −15, +30, −30, …) |
| `other-practitioner` | Same start, a different practitioner (relaxes `request-practitioner` if the patient asked for one) |
| `other-day` | Same time on the next/previous days within `searchDays`, then nearest time on those days |

### Cost model (configurable, documented in UI)

```
cost = 1.0 × (|Δ minutes| / 15)           // per 15-min shift on the same day
     + 6.0 × |Δ days|                      // per day moved
     + 8.0 × [requested practitioner changed]
     + 0.5 × [any practitioner changed when none was requested]
     + 0.1 × [earlier than requested]      // slight preference for later over earlier on ties
```

Weights live in `CostWeights` with defaults above; the UI explains them in plain words ("we prefer keeping your doctor over keeping your time").

### Search

- Generate variations in ascending cost (a merged lazy stream from each category), evaluate each with the fast `isBookable` path (matrix with early exit), stop at `budgetEvals`.
- **Diversity selection:** take the cheapest bookable alternative; then fill remaining slots with the cheapest from categories not yet represented; then cheapest remaining. No two alternatives with identical (practitioner, start).
- Every returned alternative includes its assignment and `changes: [{ field, from, to }]` and its `cost`.
- Never suggest times before `now + minNotice` (they would fail anyway — but generators skip them for speed).
- If none found within budget: `alternatives = []` and the UI shows "No nearby openings in the next 14 days".

## Step 6 — Admin fix (minimal policy change)

Staff-only answer to "what policy change would have allowed this exact request?"

- Consider candidates whose `V(a)` contains **only** admin-relaxable facts (catalog column). Each such `V(a)` is a *correction set* — relaxing all of it makes the request bookable via `a`.
- Choose the one with minimum total admin-cost (catalog weight), ties by candidate order.
- Express as concrete changes where possible: `min-notice 120 → 60 min`, `Tom Harris daily cap 6 → 7`, `procedure buffer 15 → 0 min` (the exact value needed is computed: e.g. the largest buffer that would still fit).
- If no candidate qualifies: `adminFix = null` ("No policy change alone would allow this — it's blocked by bookings or availability").

## Step 7 — Trace (optional)

With `trace: true`, include the candidate list, the full matrix (`V(a)` per candidate), reduced family, search node count and the alternatives evaluated. Used only by the engine inspector page; capped in size.

## Performance plan

| Operation | Target p95 (demo dataset, Node, laptop) | Budget mechanism |
| --- | --- | --- |
| `isBookable` | ≤ 2 ms | Early exit on first empty `V(a)` |
| `evaluate` without alternatives | ≤ 20 ms | Factorised matrix + bitsets |
| `evaluate` with alternatives | ≤ 400 ms | `budgetEvals`, lazy cost-ordered generation |
| Availability grid (one service, one day, all slots) | ≤ 100 ms | Repeated `isBookable` with shared per-day indexes |

Benchmarks (`pnpm bench`) run three dataset sizes: **S** (demo clinic), **M** (3× resources, 2× bookings), **L** (10× resources). Targets apply to S; M/L are reported to show scaling. If S misses targets in Phase 1, optimise before building UI.

## Extension notes (not in v1)

- **Multi-resource services** (two practitioners): candidates become longer tuples; hitting-set logic is unchanged.
- **Series bookings:** a series is bookable iff every occurrence is; explanation = union of per-occurrence explanations with grouping.
- **Larger clinics:** if candidate products explode, switch Step 4 to an incremental MUS algorithm (QuickXplain) over a CP-SAT model — the `Explanation` contract would not change.
