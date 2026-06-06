# Wiom Change Spec — Authoring Guide v1.0

Companion to **Wiom_Change_Spec_Template_v1.0.html**.

This guide is the canonical reference for what content belongs in each section of a **Change Spec** — the deliverable for modifying an existing feature's behaviour or logic. Two audiences: human PMs learning the template, and AI agents needing to understand boundaries. It focuses on **substance** — what a section should contain and why — not on HTML rendering.

It is the sibling of the **Product Solution Spec Authoring Guide v2.0**. Where the two overlap (conventions, Mermaid, the engineering scope fence) this guide stays consistent and does not re-derive; where the Change Spec differs (it is a *diff*, not a feature) this guide is the authority.

---

## 0. The spec's purpose

A Change Spec is **the canonical articulation of how an existing behaviour should change** — precise enough that engineering can build and test the change, and that nothing adjacent breaks.

A modification is a **diff**, not a feature. So the spec is built around four moves: *why* the change, *current* behaviour, *desired* behaviour, and the *delta* between them — plus an explicit guard on what must **not** change.

Use a Change Spec when: the behaviour or logic of an existing feature changes, and the change goes PM → Engineering.

Do **not** use it when:

- You are building something new → use the **Product Solution Spec**.
- The change is UI / UX only with **no backend dependency** → it is a design PR raised by design and merged by engineering. No spec.
- The change is UI / UX **with a backend dependency** → use the **Experience–Data Spec** (design authors states, PM authors data & rules).

It is **not**:

- A code-generation contract. Engineering produces its own design doc (API contracts, schemas, concurrency, retry, storage, monitoring) and decides architecture.
- A restatement of the original feature. The baseline lives in the feature's PRD / OS spec; this spec captures only what changes.
- A decision on *whether* to make the change. That validation is upstream; A2 is a light restatement of the problem, not a business case.

The governing objective: **move fast without breaking trust.** Every section earns its place against speed, accountability, or verification. The regression guard (A6) and the OS sign-off gate (A8) are the two places where "don't break things" becomes a hard check rather than a hope.

---

## 1. Document conventions

### 1.1 WHAT / HOW split

- **Part A (WHAT)** captures the *change as a set of decisions* — why, current, desired, delta, guard, edges, OS impact. Stable; changes trigger re-review.
- **Part B (HOW)** captures *detailing* — workflows, state, parameters, UI, acceptance criteria. Evolves with implementation.

When in doubt: *Is this a decision about what should change, or a detail of how the change plays out?*

### 1.2 Tiers (lead PM governs)

- **Light** — A1, A2, A5, A6, A8, B5. For a contained logic tweak. A8 is always declared (a single "No" row if no OS is touched). A5 rows must be self-contained, because A3 / A4 narrative is skipped at this tier.
- **Full** — all sections. For a change touching multiple workflows, a state machine, money, or migration.

The lead PM picks the tier at triage and records it in the Quick Check. There is no rigid rule that forces a tier — judgement is the mechanism, the checklist is the backstop.

### 1.3 Row conventions

- **Required rows** — every row filled or marked N/A with a reason. "N/A" without a reason reads as "not yet thought through."
- **Example rows** — illustrative starter list. Keep what applies, remove the rest, add your own.

### 1.4 Hint blocks

Cyan hint blocks serve two purposes: disambiguating neighbouring sections that are easy to confuse — **A4 (desired) ↔ A5 (delta)** and **A6 (must-not-change) ↔ A7 (changes / invariants / edges)** — and flagging a rule that is easy to get wrong (e.g. the A3 "don't slip into implementation" tip and the B1 before/after authoring rule). They tell the author which section a given line belongs in, or how to fill it.

### 1.5 Worked examples (visible)

Yellow worked examples show a fully-filled section using one running scenario (an autopay-failure grace window). They are reference material; the author does not edit them.

### 1.6 The before/after colour key (B1)

Changed workflows are drawn twice — Before and After — using one shared key: **Unchanged** (neutral), **Added / new path** (green), **Behaviour changed** (amber), **Removed / replaced** (red). Only nodes inside the blast radius carry colour; the rest stays neutral so the eye lands on the diff.

### 1.7 Mermaid is the only diagram syntax

All diagrams use Mermaid. No SVG or image embeds.

---

## 2. Quick Check (header)

### Purpose

A 7-line triage that lets a reviewer decide whether the change is in their lane and which sections to read first. **Filled last**, after the body is complete.

### Fields and sources

| Field | Source | Captures |
|---|---|---|
| What is changing? | A2 / A5 | One line |
| What triggered it? | A2 | Bug / policy / metric / partner feedback |
| Blast radius | A6 | Existing behaviour & actors touched |
| Reversible? | — | Yes / No — clean rollback path |
| Money / PII / migration? | A5 / A8 (money); escalation flag (PII, migration) | Y/N each — see note |
| OS impact / sign-off? | A8 | None, or list of OS + all-signed-off Y/N |
| Tier | §1.2 | Light / Full |

A "Yes" on **money** is detailed in A5 / A8. **PII and migration have no detail section in this spec by design** — a "Yes" on PII routes to compliance review, and a "Yes" on migration routes to an engineering migration plan (both outside this spec — see §7). Flag them here so the reviewer pulls in the right owner; a "Yes" on either may mean the change is large enough to warrant a full Product Solution Spec instead.

### What belongs / does NOT

Atomic answers and short names; triage in under 10 seconds. No reasoning or caveats — those live in the body.

### Quality bar

Every field filled. No "TBD". A "No" that doesn't apply gets a 3-word reason.

---

## 3. Part A — WHAT (the change)

PM-authored. Stable across implementation revisions.

### A1. Metadata

**Purpose.** Lightweight header plus the link to the baseline being changed.

**What goes here.** Change title (names the behaviour, not the fix); owner (PM); status; version; **baseline** (link to the original PRD / OS spec).

**What does NOT.** Reviewer lists, RACI, timelines.

**Quality bar.** A reader can find the original feature from the baseline link. Title disambiguates from neighbouring changes.

**Common mistakes.** Title names the fix ("add grace timer") rather than the behaviour ("autopay-failure suspension timing"); missing baseline link.

---

### A2. Why This Change

**Purpose.** A short problem statement so the reader sees this change solves a real problem, not a preference.

**What goes here.** Three lines: what is wrong with current behaviour (observable), who experiences it, cost of leaving it.

**What does NOT.** The full business case, research, or RCA — those are upstream. The solution — A5 holds the change; A2 holds the problem.

**Boundary.** A2 frames the *problem*. A4 / A5 frame the *change*.

**Quality bar.** A reader who has never seen this feature understands why the change is worth making. The problem holds even if the specific change is replaced.

**Common mistakes.** A preference dressed as a problem ("it would be cleaner to…"); the solution smuggled into the problem; skipping A2 because "everyone knows why."

---

### A3. Current Behaviour

**Purpose.** How the system observably behaves today, for the slice about to change. The "before."

**What goes here.** Observable behaviour as "Given X, the system does Y." Inputs, decisions, outputs an observer can see.

**What does NOT.** Code structure — tables, services, function names, call paths. The current code is available; you are summarising its behaviour, not its design.

**Hint — A3 or implementation?** If you are naming how it is built, you have slipped out of A3. Pull back to what a user or observer sees.

**Boundary.** A3 = observable status quo. The "after" is A4; the line-by-line change is A5.

**Quality bar.** Engineering agrees A3 is an accurate description of today's behaviour without correcting it.

**Common mistakes.** Describing the codebase instead of the behaviour; describing the desired state by accident.

*Skipped at Light tier — the delta (A5) then stands alone.*

---

### A4. Desired Behaviour

**Purpose.** The same slice, after the change. The "after," described as behaviour the reader can picture.

**What goes here.** Mirror A3's structure so the two read side by side. Trigger / condition → desired observable behaviour.

**What does NOT.** The atomic change list — that is A5. Implementation.

**Boundary.** A4 = the after-state for *comprehension*. A5 = the after-vs-before diff for *build and test*. A4 lets a reader picture the result; A5 is what engineering works from.

**Quality bar.** A4 and A3 can be read as a pair and the reader can state, in their own words, what is different.

**Common mistakes.** A4 duplicating A5 row-for-row (collapse into A5 and keep A4 narrative); A4 that doesn't mirror A3's conditions, so they can't be compared.

*Skipped at Light tier.*

---

### A5. The Delta

**Purpose.** The heart of the spec. The atomic, line-by-line change engineering builds against and test cases are generated from.

**What goes here.** One row per rule, workflow step, output, message, or parameter that changes: *what changes → before → after → why*. Each row is one change.

**What does NOT.** Implementation of the change. Behaviour that stays the same (that is A6). The narrative after-state (A4).

**Hint — A5 or A4?** A single picture of the result → A4. The itemised list of differences → A5.

**Boundary.** A5 names *that* a parameter changes and why; B3 holds its value, range, and scope.

**Quality bar.** Every row is atomic (one change), self-contained (readable without A3 / A4, so Light tier works), and carries a "why." A reviewer can generate at least one test case per row.

**Common mistakes.** Rows that bundle several changes; rows that restate the whole feature; missing "why"; mixing in "what stays the same."

---

### A6. Must-Not-Change — Regression Guard

**Purpose.** The behaviours adjacent to the change that must survive untouched. This is how the change moves fast without breaking trust.

**What goes here.** The handful of behaviours genuinely in the blast radius — what a reasonable engineer might fear breaking — each with a one-line reason it is at risk. Each line becomes a REGRESSION block in B5.

**What does NOT.** An exhaustive inventory of the system. Behaviours that change (A7). Cross-OS sign-off (A8).

**Hint — A6 or A7?** Stays the same, must be protected → A6. Changes or is a new edge → A7. They are mirror images.

**Boundary.** A6 = intra-feature regression (within this feature's behaviour). A8 = cross-OS amendment governance.

**Quality bar.** Lead PM judges sufficiency: long enough that real risks are guarded, short enough that it isn't ritual. Every line has a matching REGRESSION block in B5.

**Common mistakes.** "Everything else stays the same" (ritual, unguarded); an exhaustive list no one will test; a guard line with no B5 block.

---

### A7. Rules, Invariants & Edge Cases Affected

**Purpose.** Business rules the change adds or alters, the invariants that must always hold once it ships, and the new edge cases it introduces. This is a primary source for the B5 verification layer.

**What goes here.** Two subsections:

- *A7.1 Rules & invariants* — rules added / altered, and **invariants**: truths that must hold across all paths after the change (e.g. "a customer is in exactly one service state at any moment"; "a customer kept live by grace is not treated as paid until an actual payment succeeds"). State each as a single condition that can be checked.
- *A7.2 Edge cases* — each with the condition and the intended business outcome.

**What does NOT.** Behaviour that stays the same (A6). The handling mechanism (engineering). User-visible wording for a *new screen state with backend dependency* — that is an Experience–Data Spec.

**Hint — A7 or A6?** Changes, is a new edge, or is a new invariant the change introduces → A7. An existing behaviour that must be *preserved* → A6.

**Boundary.** A7 = what changes plus what must now always hold. A6 = what must not change. Every item here is verified in B5.

**Quality bar.** Invariants are stated as single checkable conditions, not scenarios. A reviewer can stress-test the change and find each scenario here with a decision. Edges needing a decision not yet made are surfaced, not silently left.

**Common mistakes.** Invariants written as scenarios; edges with no intended outcome; an invariant or rule with no matching B5 block; delta rows misfiled as edges.

---

### A8. OS Impact & Sign-off

**Purpose.** A modification often ripples into an OS the PM does not own. This is the accountability gate: declare what is touched and confirm the owner has signed off. Mirrors the Product Solution Spec's A4 (OS ownership), A9 (cross-OS impact), and B3.2 (SPR amendment), with sign-off tracking added because a modification crosses territory that was already shipped.

**What goes here.** One row per OS plausibly in the blast radius: OS | Impacted? (Yes/No) | Amendment (one line) | Amendment type | Amendment owner (consulted) | Sign-off (Received + date / Pending). Amendment types: *Behaviour change*, *SPR parameter*, *Cross-OS contract*, *None*.

**What does NOT.** Integration contract design, event schemas — engineering. Detailed per-OS behaviour — that is the amendment owner's own spec if it warrants one.

**Boundary.** A8 = which OS territory the change crosses and whether they agreed. A6 = what must not change inside this feature.

**Quality bar.** Every relevant OS has an explicit Yes/No. Every "Yes" names a *consulted* owner (not merely assigned) and a sign-off status. A change ships only when every "Yes" is **Received**. If the change touches no OS, a single row says so and why.

**Common mistakes.** Blank "No" rows (declare them); "Yes" with no named owner; shipping with sign-off Pending; assuming "No" without asking the OS owner.

---

## 4. Part B — HOW (detailing)

Detailing engineering needs to build and verify the delta. Light tier may keep only B5.

### B1. Changed Workflows — Before & After

**Purpose.** Make the logic diff visible for any workflow whose decision flow changes.

**What goes here.** Two Mermaid diagrams per affected workflow — **Before** and **After** — sharing the same start and end nodes, coloured with the §1.6 key. Added paths green, changed behaviour amber (After); replaced paths red (Before).

**What does NOT.** Re-drawing the whole system. Restating every A5 row in diagram form — B1 covers only workflows whose *branching* changes. Architecture.

**Boundary.** B1 = decision / workflow flow. B2 = lifecycle state machine. A5 = the textual diff.

**Quality bar.** A reader compares the two diagrams and sees exactly what moved, because the skeletons match and only the diff carries colour. Every new or changed branch has an acceptance criterion in B5.

**Common mistakes.** Before and After drawn with different skeletons (uncomparable); colouring unchanged nodes; one combined diagram instead of two (the diff gets muddy).

*Conditional — skip with a one-line reason if the change is a parameter or message with no flow change.*

---

### B2. State Changes

**Purpose.** Show added / removed / re-routed states when a state machine is touched.

**What goes here.** Before and after transitions; new states and transitions marked.

**What does NOT.** Workflow decision flow (B1). Internal status flags that aren't a real lifecycle.

**Boundary.** B2 = lifecycle states. B1 = decision flow.

**Quality bar.** Every new transition has an acceptance criterion in B5.

**Common mistakes.** Force-fitting a state machine onto a change that has none (mark N/A); leaving a new transition out of B5.

*Strict conditional — only if a state machine is touched.*

---

### B3. Data & Parameter Changes

**Purpose.** Configurable values and tracked data the change introduces or alters, at the business level.

**What goes here.** Parameter / data | old | new | valid range & scope | reason. New data the system must track is named by its business meaning.

**What does NOT.** Field types, columns, storage, schema, indexing — engineering. The decision behind a value lives in A5; B3 holds the value itself.

**Boundary.** A5 = that it changes and why. B3 = the value, range, and scope.

**Quality bar.** Every changed parameter has a default, a valid range, and a scope. New tracked data is described in business terms, never as schema.

**Common mistakes.** Specifying storage or types; a value with no range; data described by column name.

---

### B4. UI Impact

**Purpose.** What the change makes the user see differently, at the level of information and state.

**What goes here.** Surface | what the user now sees. Information and state, not pixels or copy.

**What does NOT.** Visual treatment, exact copy — design. **New screen states that depend on new backend data — raise an Experience–Data Spec and link it here** rather than specifying them in a Change Spec.

**Boundary.** B4 = light user-visible consequence of a logic change. Experience–Data Spec = a design-led change that needs new backend data or rules.

**Quality bar.** A reader knows what changes on screen without the spec straying into design's or the Experience–Data Spec's territory.

**Common mistakes.** Specifying copy or layout; describing new backend-dependent states that should be an Experience–Data Spec.

*Conditional — skip if the change is not user-visible.*

---

### B5. Acceptance Criteria — Verification Layer

**Purpose.** The robust verification layer that proves the generated code is correct and nothing adjacent broke. Because code is AI-generated and review is human, B5 is the contract the code is checked against — its completeness *is* the safety of the change.

**What goes here.** Given / When / Then blocks that **exhaustively** cover four classes, plus regression. Label every block with its class so coverage is auditable, and fill the **B5.1 coverage matrix**.

1. **Business logic** — every rule added or changed (A5 delta, A7.1). Happy and negative paths.
2. **Computation & calculation** — every value the change computes or alters (amounts, durations, counts, thresholds, derived values). Tested at normal values **and** at boundaries: zero, maximum, rounding, off-by-one, window expiry, config extremes.
3. **Invariants** — every "must always hold" truth (A7.1), asserted as a condition that holds across all paths, including failure and mid-transition states.
4. **Use & edge cases** — every happy path, negative path, and edge (A7.2); every new / changed branch (B1) and transition (B2).
5. **Regression** — a REGRESSION block for every A6 guard line.

**B5.1 Coverage matrix.** A required table: class | source section(s) | items | blocks | all covered? The author fills counts; the reviewer audits. The change is not sendable until every row reads "all covered".

**What does NOT.** Test scripts, fixtures, environment setup, test data — QA / engineering. Assertions about implementation rather than observable behaviour.

**Boundary.** B5 = the observable behaviour to verify and the coverage claim. *How* it is tested (framework, harness) is engineering's.

**Quality bar.** Coverage matrix reads "all covered" on all five classes. Every computation has at least one boundary block. Every A7.1 invariant has an INVARIANT block. Every A6 line has a REGRESSION block. Overall bias ~1 happy : 3 negative/edge/regression.

**Common mistakes.** Happy-path-only criteria; a computation tested only at a normal value (no boundary); an invariant declared in A7.1 with no INVARIANT block; an edge / branch / transition with no block; an A6 guard with no REGRESSION block; criteria asserting implementation rather than behaviour; an unlabelled block (class can't be audited).

---

## 5. Closeout

### 5.1 Completeness Checklist

Sendable when: Quick Check filled (incl. tier); required sections filled; conditional sections filled or N/A with a one-line reason; **the B5 coverage matrix reads "all covered" on every class**; **every A6 guard has a matching B5 REGRESSION block**; **every A8 "Yes" has sign-off Received**.

### 5.2 Changelog

Version | Date | Author | Section(s) | Change & rationale | Approver. Substantive changes only.

---

## 6. Cross-section dependencies and update propagation

When a section is updated, check these. **This table is canonical.**

| When you update… | Check these |
|---|---|
| A2 (why) | A5 — does the delta still serve the problem? |
| A3 (current) | A4 (does the "after" still mirror it?), A5 (is the delta still accurate against today's behaviour?) |
| A4 (desired) | A5 (every desired behaviour decomposed into delta rows), B1 (After diagram) |
| A5 (delta) | B1 (flow if a branch changes), B2 (state if states move), B3 (value for a changed parameter), B5 (AC per row; a COMPUTE block per changed calculation), A8 (does a row amend an OS?) |
| A6 (guard) | B5 (a REGRESSION block per guard line) |
| A7.1 (rules / invariants) | B5 (a LOGIC block per rule; an INVARIANT block per invariant) |
| A7.2 (edges) | B5 (an edge block per case); A6 (could protecting an adjacent behaviour be at risk from this edge?) |
| A8 (OS impact) | Quick Check (OS line), B5 (cross-OS AC if a contract changes) |
| B1 (changed workflow) | B5 (AC for each new / changed branch) |
| B2 (state changes) | B5 (AC per new transition) |
| B3 (parameter) | A5 (the decision behind the value) |

---

## 7. Global out-of-scope (where these things live instead)

| Concern | Lives in |
|---|---|
| API contracts, request/response schemas, error codes | Engineering design doc |
| Event / payload schemas, types, validation | Engineering design doc |
| Concurrency / locking, retry / backoff / circuit-breaker | Engineering design doc |
| Database schema, storage, indexing, retention mechanics | Engineering design doc |
| Migration mechanics (sequence, duration, verification, rollback method) | Engineering migration runbook |
| Legal / privacy / compliance assessment (incl. new PII processing) | Compliance review — flagged via Quick Check, escalated if "Yes" |
| Monitoring thresholds, alert routing, paging, escalation | Operations runbook |
| Internal system-edge mechanisms (duplicate detection, dedup keys) | Engineering design doc |
| Effort estimation, story points, sprints, timelines | Project management tool |
| Exact UI copy, typography, layout; accessibility / device standards | Designer's deliverable / Wiom design guidelines |
| The original feature's full behaviour, rationale, evidence | The feature's PRD / OS spec (the baseline) |
| New screen states that need new backend data | Experience–Data Spec |
| Whether to make the change at all (validation, prioritisation) | Upstream decision / roadmap |
| How to refactor or restructure the code to make the change | Engineering |

---

## 8. Traversal patterns for agents

| Agent purpose | Read first | Then |
|---|---|---|
| Summarisation | QC, A2, A5 | A8 if any OS impacted |
| Test generation | B5 + coverage matrix | A5 (delta), A7.1 (rules / invariants), A7.2 (edges), B1 / B2 (branches / transitions), B3 (computations), A6 (regression) |
| Regression scoping | A6 | B5 |
| Cross-OS coordination | QC, A8 | B1, B5 |
| Change-impact review | A5, A6, A8 | B3 |

Specs with Status = "Draft", any A8 sign-off still **Pending**, any A6 line without a B5 REGRESSION block, or a B5 coverage matrix not "all covered" are not-yet-canonical and must not start build.

---

## 9. Spec-quality heuristics

**High-quality signals:**

- A2 states a real, observable problem — not a preference; the change is justified.
- A3 / A4 describe observable behaviour, never code structure, and mirror each other.
- A5 rows are atomic, self-contained (Light tier survives without A3 / A4), and each carries a "why."
- A6 lists only adjacent behaviours genuinely in the blast radius; each maps to a REGRESSION block.
- A7 edges each name an intended outcome; invariants are single checkable conditions, not scenarios.
- A8 has an explicit Yes/No per OS; every "Yes" is consulted with sign-off tracked.
- B1 Before and After share start / end nodes; only diff nodes coloured.
- B5 coverage matrix is "all covered" on all five classes; every computation has a boundary block; every invariant an INVARIANT block; every A6 line a REGRESSION block; ~1 happy : 3 negative/edge/regression.

**Low-quality signals:**

- A2 missing, or a preference dressed as a problem.
- A3 / A4 name tables, services, or functions (implementation creep).
- A5 rows bundle several changes or restate the whole feature.
- A6 says "everything else unchanged," or is exhaustive, or has a line with no REGRESSION block.
- A7 edges with no intended outcome; invariants written as scenarios; delta rows misfiled as edges.
- A8 blanks on "No" rows; a "Yes" shipped with sign-off Pending.
- B1 redraws the whole system, or Before / After use different skeletons.
- B4 specifies new backend-dependent states that should be an Experience–Data Spec.
- B5 is happy-path only; a computation tested at one value only; an invariant or edge with no block; an unlabelled block; criteria asserting implementation rather than behaviour.

---

## 9.1 B5 coverage linter (deduction rules)

The verification layer is the change's safety net, so it is linted hardest. Run these checks against a completed spec; any failure is a reject, not a deduction.

| # | Check | Fails when |
|---|---|---|
| L1 | **Business-logic coverage** | A rule in A5 or A7.1 has no LOGIC block (happy or negative). |
| L2 | **Computation coverage** | A value the change computes or alters (A5 calculation, B3 parameter) has no COMPUTE block. |
| L3 | **Boundary coverage** | A computation has a normal-value block but no boundary block (zero, max, rounding, off-by-one, expiry). |
| L4 | **Invariant coverage** | An A7.1 invariant has no INVARIANT block, or the block is a scenario rather than an always-holds condition. |
| L5 | **Use / edge coverage** | An A7.2 edge, a B1 new/changed branch, or a B2 new transition has no block. |
| L6 | **Regression coverage** | An A6 guard line has no REGRESSION block. |
| L7 | **Matrix integrity** | The B5.1 coverage matrix is absent, has a blank cell, or shows any class not "all covered" while blocks are missing. |
| L8 | **Label integrity** | A block is unlabelled or mislabelled, so its class cannot be audited. |
| L9 | **Behavioural purity** | A block asserts implementation (named services, schemas, internal calls) rather than observable behaviour. |
| L10 | **Balance** | Coverage is overwhelmingly happy-path (materially off the ~1:3 happy-to-negative/edge/regression bias) with no stated reason. |

An agent generating test cases from a completed spec should treat B5 as the source of truth, expand each block into executable tests, and report any L1–L10 failure back rather than silently filling the gap.

---

*End of Change Spec authoring guide v1.0. Sibling of the Product Solution Spec Authoring Guide v2.0.*
