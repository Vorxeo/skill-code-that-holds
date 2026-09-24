---
name: code-that-holds
description: >-
  Use while writing or reviewing code, before committing, and when deciding
  whether a change is finished. Catches defect families that survive a green
  test suite (validated-not-enforced, optional controls, rollback counters, null
  vs false, literal drift, AST vs behaviour, multi-door rules, reporting vs
  asking, production config). Distilled from real product defects.
---
# Code that holds

Every defect below was found in a real codebase, and **every one of them was green in its own test suite at the time**. The failures that matter are not the ones a test catches — they are the ones a test cannot see because the test and the code share the same wrong assumption.

Use this while writing, not only when reviewing. A check applied after the fact costs a rewrite; the same check applied while the cursor is on the line costs nothing.

---

## 1. Validated is not enforced

A field accepted by the API, range-checked, returned in the response, and written to no column. The customer reads it in the artefact they confirm, and believes they are covered.

**Follow every new field down three legs, not one:**

- **stored** — is there a column, and does the write path set it?
- **consumed** — does anything read it and change behaviour?
- **reachable** — does the reader actually *run*? A retention sweep whose only caller is a manual POST is a reader that exists and never runs. And can the field be SET from the outermost surface a customer has — the HTTP body, the CLI flag, the tool parameter?

**If it cannot be enforced yet, say so in the payload** (`not_enforced`) rather than returning a field that implies it is.

---

## 2. A control the caller can switch off

```python
if requested_by_agent_id is not None:       # optional in the body
    if agent_id not in descendants:
        raise PolicyDenied(...)             # omitting skips this entirely
```

The docstring said the rule; omitting one optional field made it false for every roleless caller.

**The fix is never "make the field required".** Make *absence* mean something explicit that needs authority:

```python
if requested_by_agent_id is None and not operator_authority:
    raise PolicyDenied("...names no agent, so this is an operator action")
```

**Resolve authority from the credential, never from the body.**

---

## 3. The rollback that undoes your own counter

```python
with transaction(write=True) as session:
    challenge.attempts += 1
    if not code_matches(code, challenge.code_hash):
        raise refusal          # rolls back the +1
```

**Anything that must survive a failure has to commit before the failure.** Claim the attempt in its own committed transaction, *then* compare.

---

## 4. Absence and zero are different, and so are null and false

- `unknown = null, never 0` — a partial count read as a total is a lie.
- `notified: true / false / null` — reached somebody / tried and reached nobody / nothing attempted.
- `objections: null` when a reviewer could not answer — an empty list says someone looked and found nothing.

**Parse other people's booleans defensively.** `bool("false")` is `True`. Prefer explicit membership checks.

---

## 5. A literal copied is a literal that drifts

**Derive. Never copy.** `len(result["jobs"])`, not `4`. When you fix a literal, **grep the whole repository** for it.

---

## 6. Read the parsed tree, not the text

Assert properties of code with `ast`, not substring `in`. **But names are structure; values are behaviour.** If the property is "this surface refuses a roleless caller", call the surface.

**A scan with a blind spot is worse than no scan** — see `guards-that-scan`.

---

## 7. One front door is not the rule

> A rule that lives in one front door is a rule the next front door does not have.

Put the check in the service and pass the authority in. When a setting gains a new input, grep every reader of the old one. Closing a hole is not finished until you listed what *else* reaches the thing you guarded — including inverse operations (export/import, ancestor/descendant, preview/mutation).

**Never write down an assumption as a justification.** If you cannot point at the line that authorises, do not claim one exists.

---

## 8. Reporting is not asking

Before an act: refusing is correct. After an act: refusing makes the record wrong in the direction that flatters the operator. Reported overspend is **recorded, flagged and escalated**, never rejected.

---

## 9. The paperwork must not undo the act

**Commit the consequential act, then do the bookkeeping outside the transaction**, and report `null` when bookkeeping failed — not absence.

---

## 10. Deleting beats wiring

Unreachable credential/money code is a liability. **Two exceptions:** never delete something whose name touches denial, erasure, retention, audit, retraction or halting merely for having no caller — write what replaces it. Never delete a *control*; wire it. Pair with `reachability-audit`.

---

## 11. The comment that stops being true

When you add the thing an existing comment says does not exist, **grep for the claim**.

### 11b. Never pin your own claim in a test

Assert error **codes**, status, shape, refusal, count — not the wording of an explanation.

---

## 12. Production is not code

- **Which identity talks to the database?** PostgreSQL exempts a superuser from RLS, including under FORCE.
- **What starts the periodic work?** Check compose/manifests, not only code.
- **Which settings change the behaviour you will describe, and what do they default to in production?** Run touched tests with production values set.

**Assert properties like these at startup**, not only in a test.

---

## 13. Irreversible controls do not start irreversible

Sweeps that delete customer data: modes `off` / `warn` / `enforced`, default `warn` — compute exactly what would be taken and write that to the record first. See `migration-and-data-safety`.

---

## 14. A refusal must not teach

Equalise messages **and** work on sign-in / challenge paths. Early return on "no such account" enumerates by stopwatch.

---

## 15. Identity is a pair, never an email

`(issuer, subject)`. Matching login to a local account by email alone is a live CVE class.

---

## Working habits

- **Write the test that fails first.**
- **Read the result file, never the notification** (`verified-delivery`).
- **Anchor patches on exact text and let them fail loudly.**
- **Assert the precondition in the fixture**, not in your head.
- **Grep the whole repository** when fixing a literal, path, or claim.
- **Say what was Verified and what was Written.**

## Failure modes (named)

| Name | Meaning |
|------|---------|
| **validated-not-enforced** | Field accepted/returned; not stored or not consumed |
| **omit-to-bypass** | Optional body field skips the control |
| **body-as-principal** | Authority taken from request body |
| **rollback-counter** | Attempt/rate counter undone by the failing transaction |
| **null-collapsed** | null/false/0/[] merged into one meaning |
| **literal-drift** | Magic copied count/list that bit when shape grew |
| **ast-as-behaviour** | Name/structure check treated as a refuse proof |
| **single-door-rule** | Gate only on one transport |
| **report-as-ask** | Post-act refusal that falsifies the record |
| **paperwork-undoes-act** | Bookkeeping failure rolls back the consequential act |
| **delete-the-control** | Removed deny/erase/retention because "unreachable" |
| **stale-comment** | Prose claim no longer true |
| **superuser-RLS** | App DB role bypasses tenant isolation |
| **warn-as-enforced** | User-facing claim implies block; mode is warn |
| **email-as-identity** | Account bind by email alone |

## Interaction with other skills

- **verified-delivery** — habit of reading result files.
- **guards-that-scan** — when you write the checker for these families.
- **reachability-audit** — dead doors vs wrong guards.
- **fail-closed-review** / **adversarial-qa** — authz PRs and omit/second-door.
- **contract-and-compat** — `not_enforced` honesty; front-door parity.
- **migration-and-data-safety** — warn-default sweeps; RLS roles.
- **handoff-faber-rigor** — cite which families you checked in the packet.

## Checklist (before "done")

- [ ] New fields: stored + consumed + reachable + settable from the front door
- [ ] Controls: absence needs authority; credential resolves principal
- [ ] Counters/limits that must survive failure commit before refuse
- [ ] Tri-state null/false/empty preserved where meaning differs
- [ ] No magic literals; grepped after any literal fix
- [ ] Behavioural refuse test if the claim is about refusal
- [ ] All doors (REST/MCP/gRPC/CLI/jobs) and inverses considered
- [ ] Production identity / scheduler / defaults checked when relevant
- [ ] Status is Verified or explicitly Written
