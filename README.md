# Interrogation Portal — Semantic Model

The grounding corpus for the Interrogation Portal AI layer. One markdown file per concept,
read at inference time to ground the AI so it answers manager questions accurately and never
hallucinates joins.

This is documentation ABOUT the backend. It is not SQL and it is not the database. The trusted
SQL primitives it references (the `fn_Resolve*` resolvers, the `Dash_*_Hydrated` hydrators, the
ledger) are what actually execute; these files tell the AI what is true and where to look.

## What each file is

One concept, two faces:
- structure (frontmatter) — tables, keys, accessors, relationships, disclosure
- prose (`## Meaning`) — what the concept means, what it is NOT, the gotchas

Files are numbered by descent order down the spine: `NN-concept.md`.

## The spine (status)

| # | file | concept | status |
|---|------|---------|--------|
| 01 | coordinate-frame | Client & Time (whose, and when) | done |
| 02 | actor | Actor & Access Scope (who's asking) | done |
| 03 | eligibility | Badges, requirements, the gate (replaces rulebook) | done — re-cut 2026-09-15 |
| 04 | structure | Work-Definition Shell (mission type / seat / mission) | done |
| 05 | labor | Team / Driver (the supply side) | done — re-cut 2026-09-15 |
| 06 | assets | Instance Layer (owned, held, conditioned) | provisional |
| 11 | documents | the held proof | done — re-cut 2026-09-15 |
| 10 | output | derived metrics & recommendations | NEXT |
| 07 | preparation / execution | the plan vs what happened | pending |
| 09 | ledger | the audit spine + narrative layer | pending |
| 08 | governance | gates, compliance | pending |

Living corpus. Concepts are added and refined continuously; a correction is a text edit that
takes effect on the next question, no retraining.

## Entry format

Frontmatter:
- `concept / title / kind / branch` — identity
- `aka` — the words a manager actually uses (maps language to the right concept)
- `disclosure` — `citable` / `internal` / `gated` fields (the static half of access control)
- `grounding` — tables (keys, carries, role), canonical accessors, scope predicate
- `relationships` + `realized_in` — edges to neighboring concepts, including the
  definition-to-instance arcs where the highest-value questions live
- `cite` — the rows an answer must trace back to

Body:
- `## Meaning` — the prose face

## Conventions

- `[[concept]]` links a related concept; its file is `NN-concept.md`.
- Disclosure is two-tier: the static class here INTERSECT the actor's live permissions.
- `fill_reality` blocks are counts for demo client 7293, with the verification date.
- Never expose the engine: cite evidence (rows), never the method.

## How the AI layer consumes this

Grounding for both layers: the curated intents (deterministic, trusted SQL) and the general
schema-aware fallback (a read-only query composed only from what the model sanctions). See the
AI Layer Architecture Handover for the full request pipeline and guardrails.
