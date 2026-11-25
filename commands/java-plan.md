---
description: Break a Java or connector epic into stories, tasks, acceptance criteria, estimates, milestones and risks
argument-hint: "[epic | feature | new connector for a source]"
allowed-tools: Read, Grep, Glob, Write
disable-model-invocation: true
---

# Delivery plan

**Goal:** $ARGUMENTS

Turn the goal into a plan the team can execute: epics, stories and tasks, each story with acceptance criteria and an estimate, sequenced into milestones, with a Definition of Ready, a Definition of Done and a risk register. If no goal is given, ask for it, and stop.

## 1. Preflight

Right-size the output; planning overhead must stay smaller than the work being planned.

1. **Is it big enough to plan?** A single small, well-understood task doesn't need epics, story points and a risk register. Say a plan would be ceremony, and give an ordered task list instead.
2. **Is the goal defined?** If the outcome, constraints or success criteria are too vague to break down honestly, list the specific questions to answer first. Don't build stories around guesses.
3. **Don't re-plan planned work.** If a backlog or plan exists, in the repo or pasted by the user, read it and propose changes (gaps, re-sequencing, missing risks), not a competing plan.
4. **Estimates need a basis.** If team size or velocity is unknown, give relative sizes only, and say that a calendar mapping needs the team's numbers.
5. **Read the code for technical work.** Base tasks on the real modules, build and test setup, and name the modules each task touches.

## 2. Method

1. **Goal and scope:** the outcome, success metrics, explicit non-goals, and constraints.
2. **Break it down.**
   - Split the epic into user stories: "As a <role> I want <capability> so that <value>".
   - Make each story INVEST: Independent, Negotiable, Valuable, Estimable, Small, Testable.
   - Give each story acceptance criteria as Given / When / Then.
   - Split each story into engineering tasks: design, build, tests, docs, gates, review.
3. **Estimate** with relative story points (Fibonacci) or t-shirt sizes. Flag unknowns as time-boxed spikes, and don't over-precise early.
4. **Sequence.** Map the dependencies, pull risky, blocking and foundational work forward (a walking skeleton first), and group the work into milestones, each with a demoable increment.
5. **Set the quality bars,** and apply them to every item:
   - **Definition of Ready:** clear acceptance criteria, dependencies known, estimable.
   - **Definition of Done:** code, tests, coverage gate, docs, review and a green build.
6. **List the risks,** each with its likelihood, impact, mitigation and owner. Revisit them at every milestone.

## 3. Planning a connector

Use this as the starting backlog for a new source, or as a checklist for a large connector feature. Drop the epics that don't apply.

| Epic | What "done" means |
|---|---|
| 1. Source investigation (spike) | Auth, endpoints, paging, filtering, change tracking, delete visibility, time zones and limits are written down, each marked documented or measured. There's a go/no-go per capability (initial load, change feed, deletes). |
| 2. Connectivity and configuration | Auth works, including token refresh. Every config key is validated at start-up, naming the key. Secrets are masked and never logged. |
| 3. Discovery and schema | Objects and fields are listed from the source, types are mapped, and unknown or changed fields are handled deliberately. |
| 4. Initial load | Stable paging (keyset or token), resume after a crash from the last acknowledged position, no skipped or duplicated rows. |
| 5. Change capture | Inserts, updates and deletes are captured; a restart resumes correctly; an idle feed isn't mistaken for the end. The ordering and lag rules are documented. |
| 6. Host integration | The lifecycle callbacks are wired; the checkpoint is written only after the host acknowledges; the delivery guarantee (at-least-once or exactly-once) is proven by a test. |
| 7. Hardening | Errors fail loudly and resumably, logs and traces are diagnostic without secrets, and a security review is done. |
| 8. Verification | Unit tests against a fake source that reproduces real quirks, integration tests through the host's driver where the host provides one, and a live verification run against a real instance, recorded. |
| 9. Docs and release | Configuration reference, limitations, README and `CLAUDE.md` updated; packaging built; version stamped. |

**Connector Definition of Done** (on top of the general one):
- An integration test covers the change, through the host's driver where the host provides one.
- The configuration reference and the limitations doc are updated.
- A live verification run against a real instance passed, or the reason it couldn't run is written down.
- No new config key is silently ignored.

**Typical connector risks:** API rate limits and quotas; objects with no change feed; deletes that aren't observable; time zones and clock skew; paging caps or unstable paging; API version changes; permission gaps for the integration user; schema drift mid-run; data volume.

## 4. Output

- **Default:** a markdown plan with sections for goal and scope, epics, stories with acceptance criteria and estimates, tasks, sequence and milestones, DoR and DoD, and a risk table. Show it in the chat; save it to a file only if the user asks, where the repo keeps plans (or `docs/plans/<goal>.md`).
- **Tracker import:** if the user wants to import the plan into a tracker, ask which one, and produce a CSV that matches that tracker's import template. Never claim a CSV layout works without checking the template.

## Avoid these errors

- **Real names only.** Tasks name real modules and files. Anything new is marked NEW.
- **Honest numbers.** Don't present relative sizes as dates, or invent a velocity.
- **Deliverable capabilities only.** Don't plan a capability the source can't support, such as a change feed on an object with no change tracking. Plan the spike that settles it.
- **Vertical slices.** Each story delivers testable value and fits in one sprint; the critical path is identified.
