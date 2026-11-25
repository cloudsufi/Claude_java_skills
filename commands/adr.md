---
description: Write an Architecture Decision Record for a technical decision, in the repo's ADR format and numbering
argument-hint: "[the decision, e.g. use keyset paging instead of offset paging]"
allowed-tools: Read, Grep, Glob, Write, Edit, Bash(git log:*)
disable-model-invocation: true
---

# Architecture Decision Record

**Decision:** $ARGUMENTS

Record one decision: its context, what was decided, the consequences, and the alternatives that lost. If no decision is given, ask for it and stop. An ADR needs a real decision with real alternatives.

## 1. Preflight

1. **Find where decisions already live, and their format.** Look in `docs/adr/`, `doc/adr/`, `docs/decisions/` and `architecture/decisions/`, and for dated decision entries inside `CLAUDE.md` or a README. Connector repos often record decisions inline, as dated "editing rules" or "invariants" that give the reason and the rejected alternative.
   - If the repo uses ADR files, follow their template, numbering and zero-padding.
   - If it records decisions inline, offer to add the entry there in the same style, and write a separate ADR file only if the team wants ADRs.
   - If there's nothing yet, start at `docs/adr/0001-<kebab-title>.md`.
2. **Ground it in the real system.** Read the code and build files the decision touches. Use real module, class and setting names. If the code contradicts the decision's premise, say so first.
3. **Check the inputs are enough.** If the forces, constraints or alternatives are too vague to write honestly, list the specific questions to answer, and stop. Don't invent context.
4. **Check it isn't already decided.** If an existing ADR or inline decision covers the same ground, either supersede it explicitly or say there's nothing new to record.

## 2. Write it (Nygard format, plus alternatives)

```markdown
# 0007. Read the change feed with a watermark window

Date: 2026-10-14

## Status

Proposed (would supersede 0003)

## Context

What forces are at play, with facts and measurements: requirements, constraints,
what breaks today, and what the source or platform allows.

## Decision

What we will do, in one or two plain sentences.

## Consequences

Good and bad, including what becomes harder and the follow-on work.

## Alternatives considered

Each option, and why it lost.
```

- **Number:** the highest existing number plus one, with the same zero-padding. Never reuse a number.
- **Status:** a new ADR starts as Proposed, unless the user says the decision is already agreed. Later statuses are Accepted, Deprecated, or Superseded by NNNN. A Proposed ADR can still be edited; an Accepted one can't.
- **Date:** today's date.
- **Title:** a short imperative phrase; the file name is the number plus the kebab-case title.
- Keep it short, and keep every claim checkable.

## 3. Housekeeping

- **Superseding:** don't rewrite the old ADR. The only edit it gets is its status line, changed to "Superseded by NNNN".
- **Index:** if the ADR folder has an index (a `README.md` listing the ADRs), add the new entry.
- **Links:** link the ADR from the design doc or arc42 section it affects, if one exists.

## Avoid these errors

- Never edit the decision or context of an accepted ADR. Supersede it with a new one.
- Real names only. Every module, class or setting the ADR names exists, or is clearly marked as planned.
- Consequences must include the costs. An ADR with only upsides isn't a decision record.
