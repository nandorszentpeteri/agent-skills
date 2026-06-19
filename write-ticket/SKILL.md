---
name: write-ticket
description: >-
  Write, draft, create, or extend tickets/issues in Nandor's style, for any
  tracker. Use this skill whenever the user asks to write a ticket, create an
  issue, draft a story or task, extend an existing ticket with more detail, or
  convert investigation notes into a proper ticket. Covers Bug, Task, Story,
  and Investigation ticket types with the correct structure and sections for
  each, then files to the named tracker (e.g. Jira) on approval.
---

# Write Ticket

Write tickets in Nandor's style. The goal is a ticket that is immediately
actionable: clear scope, unambiguous acceptance criteria, and an explicit
justification for the work.

## Drafting vs. filing

**Default to drafting.** When asked to "write" or "create" a ticket, produce the
full ticket in the chat first and let the user review it. Do **not** file it
into the tracker yet — a ticket that lands on a live board half-formed is worse
than no ticket, because someone may pick it up or triage it before it's right.

**File only after explicit approval.** Once the user approves the draft (e.g.
"looks good", "file it", "create it"), create the real ticket via whatever
tracker the user named or has connected. When filing:

- **Adapt the draft to the target's format.** The draft is written in markdown;
  the tracker may not accept markdown. Convert to the target's native format and
  keep the **summary/title** as plain text (no markdown, no code ticks).
  Determine the target from what the user named or the available tooling — don't
  assume one. For example:
  - **Jira** → use the Atlassian MCP (`createJiraIssue`); body is ADF / Jira
    wiki markup, not markdown.
  - **GitHub / GitLab Issues** → markdown is accepted mostly as-is.
  - **Linear** → markdown is accepted.
  - Any other tracker → match its documented format.
- Confirm the project/board, issue type, and any required fields before the call
  if they aren't already known. Don't guess a project key.
- **Honor any placement or metadata the user specifies.** If they say "put it
  under epic X", "add it to the current sprint", "assign it to Jane", "label it
  `tech-debt`", "set priority high", or similar, set those fields when creating
  the ticket — don't drop them or leave them for a follow-up. Common fields:
  parent/epic link, sprint, assignee, reporter, labels/components, priority,
  fix version, story points, due date. Map each to the target tracker's field
  (e.g. in Jira: epic → parent or Epic Link, sprint → the Sprint field,
  assignee → `assignee` accountId). If a named value can't be resolved (an epic
  key, a sprint, or a user that doesn't exist or is ambiguous), surface that and
  ask rather than silently skipping it.
- After filing, report the created ticket's id/key and link back to the user.

If the user explicitly says to file directly (skipping review), do so — but the
review-first default protects against shipping a rough draft to a shared board.

## Ticket types and structure

### Bug

**Summary:** `<Component>: <symptom>` — exact and terse. Name the thing that's
broken and what it does wrong.

```
"Settings dialog: \"Email\" label overlaps the input placeholder"
"Auth: login redirect loop on expired session"
```

**Body sections:**

```markdown
### Steps to reproduce

1. ...
2. ...

### Expected

What should happen.

### Actual

What actually happens.

### Notes

Extra context, affected environments, workarounds, screenshot references.
```

Reference a screenshot when the bug is visual.

---

### Task / Story

**Summary:** Action verb + what + context. Concise, no padding.

```
"Extract shared AI skills into a standalone repo with git submodule"
"Add per-MR ephemeral branch deployments"
"Implement SemVer for template components"
```

**Body sections:**

```markdown
## Acceptance Criteria

Clear, unambiguous requirements. Format them however makes the requirement most
readable — bullet list, numbered list, or prose paragraphs are all fine. The
point is that someone reading it can tell without ambiguity when the work is
done. Avoid vague phrases like "it should work" or "handle the case".

## Why

The motivation. Why does this work need to happen? What problem does it solve,
what risk does it mitigate, or what constraint does it satisfy? This section
must always be present — a ticket without a why is a ticket that will be
deprioritised or misunderstood.

## How / Implementation  *(include when the approach is non-obvious)*

Numbered steps, code blocks, YAML examples, config snippets. Add a table for
scope or effort breakdown when there are multiple moving parts.

| Item | Effort |
|------|--------|
| ...  | Small  |

## Related  *(include when cross-links exist)*

- TICKET-123 — brief description of relationship
- MR: https://... (add MR link when the ticket is completed)
```

---

### Investigation

**Summary:** Prefix with `INVESTIGATE - `, then a concise description of what
needs to be investigated.

```
"INVESTIGATE - Branch deployment options"
"INVESTIGATE - Code Owners hierarchy resolution"
```

The body can be lighter — a brief statement of the question to answer and, if
known, what a good output from the investigation looks like (a decision, a
doc, a follow-up ticket, etc.).

---

## Worked examples

These are real tickets in the target style. Use them to calibrate tone, density,
and section depth — match the level of detail to the input, as these do.

**Bug** — summary: `Settings dialog: "Email" label overlaps the input placeholder`

```markdown
On the account **Settings → Profile** page, opening the "Edit contact" dialog
shows an email field whose **"Email" label overlaps the "Enter your email"
placeholder text**. The two pieces of text render on top of each other,
producing garbled, illegible text.

### Steps to reproduce

1. Go to Settings → Profile.
2. Click "Edit contact" to open the dialog.
3. Observe the email field.

### Expected

The field label and placeholder are displayed cleanly without overlapping.

### Actual

The "Email" label is rendered directly over the "Enter your email" placeholder,
so both overlap and are hard to read.

### Notes

* Affects the email input in the edit-contact dialog only.
* See attached screenshot.
```

**Task** — summary: `Extract shared AI skills into a standalone repo with git submodule`

```markdown
## Acceptance Criteria

* Standalone skills repo created with the portable skills extracted
* Submodule added to the consuming repo at `.agents/skills/shared/`
* Project-specific skills remain in-repo unchanged
* AGENTS.md updated with submodule contribution rules
* At least one other consuming repo adopts the submodule

## Why

The skills currently live in only one repo. Other repos need the same set.
Extracting them into a standalone repo enables reuse without copy-pasting skill
files across projects — single source of truth, clear ownership boundary, and
new repos adopt the full set with one submodule.

## How

1. Create the standalone skills repo containing the portable skills
2. Add it as a submodule at `.agents/skills/shared/` in consuming repos
3. Project-specific skills stay at `.agents/skills/<name>/`
4. Contributing to shared skills happens upstream; consumers bump the ref

## Related

* TICKET-123 — original in-repo skills work
```

---

## General conventions

- **Code and config go in fenced blocks** — YAML, shell scripts, JSON, inline
  commands all get fenced. Never paste raw config as prose.
- **Tables for scope** — when a task has multiple components with different
  effort levels, use a markdown table rather than a flat list.
- **Why is mandatory** for Task and Story types. It answers the "should we do
  this at all?" question before implementation begins.
- **Related / MR links at the bottom** — add them when they exist; don't
  create placeholder sections for links that don't exist yet.
- **AI-generated tickets** — if the ticket body is entirely AI-generated
  (rather than based on real investigation or design notes from the engineer),
  note this explicitly at the top: `*Note: this ticket is entirely AI generated
  for the purpose of preserving investigation context.*`

## Using additional context

The user may provide supporting material alongside the ticket request — research
notes, benchmark data, architecture diagrams, API docs, error logs, design
decisions, Slack threads, or raw investigation output. Treat this as the source
of truth and fold it into the ticket properly:

- **Research / investigation notes** → distill into the How/Implementation
  section; preserve key findings, discard noise
- **Technical data** (benchmarks, metrics, error rates) → include as a fenced
  block or table in the relevant section; cite what it shows, not just the raw
  numbers
- **Architecture or design context** → use to sharpen the Acceptance Criteria
  and inform the How section; don't just quote it back verbatim
- **Existing ticket or thread** → extract the signal, restructure into the
  correct ticket format, add any missing mandatory sections

The richer the context the user provides, the more specific and concrete the
ticket should be. Vague input produces a lighter ticket; detailed input should
produce a detailed ticket. Never invent specifics that aren't in the context —
if something is unclear, leave a `[TBD]` placeholder or ask.

## Workflow

1. Identify the ticket type: Bug, Task/Story, or Investigation.
2. Absorb any additional context the user provided before drafting.
3. Draft the summary line first — get the wording tight before writing the body.
4. Fill in the sections for that type, in order, weaving in context where it belongs.
5. Check: does every requirement in AC have a clear pass/fail condition? Is the
   Why section present and specific? Are code examples in fenced blocks?
6. Present the draft for review. Don't file it yet (see **Drafting vs. filing**).
   If something essential is genuinely unclear — and you can't fill it with a
   `[TBD]` placeholder — ask before drafting rather than inventing specifics.
7. On approval, file the ticket into the tracker and report the key + link.

If the user is extending an existing ticket, preserve what's there and add the
missing sections or flesh out thin ones — same review-then-file flow applies.
