# Change Specs — Wiom Change Spec Template v1.0

Workspace for authoring **Change Specs** (PRDs for modifications to existing features). New features use the Product Solution Spec instead; UI-only changes are design PRs (no spec).

## How to author a spec

In Claude Code, say **"new change spec for <feature>"** (or `/change-spec`). The skill enforces the mandated workflow:

1. **Scope check** — is a Change Spec even the right doc?
2. **Context intake** — feed it the current code / OS files; it plays back current behaviour for confirmation
3. **Clarifying questions** — answered BEFORE any drafting (tier, blast radius, OS impact, money/PII/migration, rollback, undecided edges)
4. **Generate** — HTML spec cloned from the official template
5. **Self-review** — context/constraints pass, format pass, L1–L10 coverage lint
6. **HUMAN REVIEW** — hard stop; you review the spec (most critical step)
7. **Feedback & close** — changelog, version bump, closeout checklist

## Layout

- `templates/` — official template HTML + authoring guide (do not edit; v1.0)
- `specs/<change-name>/` — one folder per change spec

## Expectations (from format owner)

- OS amendments approved **before** handoff (every A8 "Yes" → sign-off Received)
- Design changes live in the same spec (B4) — spec first, then design
- Senior PM / colleague review based on level of change & risk
- Format is being tested with the **P2 build** — send all feedback (yours + engineering's) to the format owner
