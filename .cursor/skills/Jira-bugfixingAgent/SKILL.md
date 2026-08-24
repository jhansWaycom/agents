---
name: jira-bugfixing-agent
description: >-
  Jira-bugfixingAgent: analyzes a Jira ticket, implements the correct
  root-cause fix on branch bugfixes/<ticket-key>, lists every file changed,
  runs all module tests, reports all test-case scenarios (success, failure,
  edge, race), pastes local curls plus paired Elasticsearch and DB queries
  for dual validation (chat only, never a .sh file), opens a pull request,
  and after the PR exists prints a filled GitHub-ready description the user
  can paste into the PR before merge, then closes with a self-assessment
  scorecard grading its own run (ticket fidelity, root cause, coverage,
  tests actually executed, dual validation, security, honesty) with
  evidence, unverified claims, and residual risk. Use when the user provides
  a Jira ticket number or key (for example SR-3099) and asks to fix the
  issue, implement the ticket, analyze the ticket, or create a PR for it. Do
  not hallucinate: follow the Jira comments and description only.
---

# Jira-bugfixingAgent

Retrieve the ticket, brief responsibility, implement the **root-cause fix**,
run tests, list **all** scenarios (success, failure, edge, race), paste curls
**and paired ES + DB queries** in chat only, open a PR, print a **filled
PR description** to paste into GitHub before merge, and finish with a
**self-assessment** of the run (Step 10).

Do **not** start coding until analysis is complete. If the ticket is too vague, stop and ask.

## Guardrails — don't hallucinate

**Just follow the comments and descriptions.** The only product requirements
are the Jira **summary**, **description**, and **comments** (later comments
win if they narrow or correct earlier text). Quote those sources in Step 2.

- Do **not** invent expected behavior, APIs, fields, tables, ES indices,
  payloads, error codes, or acceptance criteria
- Do **not** assume a similar past ticket, “usual” Way pattern, or unstated
  product rule is in scope
- Code facts (paths, method names, index/table/column names) come only from
  files you **actually read** in this repo — never from memory of another
  service or an imagined schema
- Dual-validation queries use only discovered names; if you did not find the
  index or table in code, write `N/A — not in code` instead of guessing
- If description and comments conflict, follow the **latest comment**, say so,
  and do not merge both into a new invented requirement
- If anything needed to implement is missing from the ticket **and** the
  code, **stop and ask** — do not fill the gap with a guess
- Do not claim tests, PRs, or file changes you did not produce

## Hard rules — no local query files

**Never** create, stage, commit, or push `local.sh`, `local-validation/**`,
generated curl `*.sh`, or `.sql` query files. Put curls and ES/DB queries in
chat only. Leave those paths untracked if they already exist.

## Input

The user passes a ticket key or number, for example:

- `SR-3099`
- `fix SR-3099`
- `3099` (resolve against Jira; do not guess the project)

Extract the ticket key. If only digits are given, look the issue up in Jira rather
than assuming a project prefix.

## Progress checklist

Copy and track:

```
Ticket: <KEY>
- [ ] 1. Retrieve ticket (summary, description, comments, attachments, links)
- [ ] 2. State responsibility from description+comments only (quote sources; no invented scope)
- [ ] 3. Create branch bugfixes/<KEY> from origin/dev
- [ ] 4. Implement the correct root-cause fix (minimal, ticket-scoped)
- [ ] 5. Add tests for success, failure, edge, and race; run those, then all module tests
- [ ] 6. Paste local curls AND paired ES + DB queries in chat — do NOT write any .sh/.sql file
- [ ] 7. Security / secrets / SOC2 check
- [ ] 8. Collect files changed (path + why), commit, push, open PR
- [ ] 9. Handoff: PR number, files, scenarios, curls, dual-validation queries, paste-ready PR description
- [ ] 10. Self-assessment scorecard: grade this run with evidence, list unverified claims and residual risk
```

## Step 1 — Retrieve the ticket

Use the Atlassian Jira tools. Site: `https://wayglobal.atlassian.net`.
Pass `cloudId` as `wayglobal.atlassian.net`. If a call fails, fall back to
`getAccessibleAtlassianResources`.

Fetch the issue with comments:

```
getJiraIssue(
  cloudId=<cloudId>,
  issueIdOrKey=<KEY>,
  fields=["summary", "description", "status", "issuetype", "priority",
          "labels", "components", "assignee", "reporter", "comment",
          "issuelinks", "attachment"],
  responseContentFormat="markdown"
)
```

`comment` returns comments on `fields.comment.comments`. Read **all** comments;
later comments often contain the real repro, expected behavior, or a narrowed scope.

Also collect: type, status, priority, components, labels, linked issues,
acceptance / repro, environment, affected API if mentioned.

If the ticket is missing or inaccessible, stop and tell the user.

## Step 2 — Analyze and state responsibility

Post this briefing **before** changing code. This is the agent's contract for the run.

```markdown
## Ticket <KEY>
**Summary:** ...
**Type / status / priority:** ...
**Reported by / comments:** ...

### What the ticket says (quote description / comments — do not paraphrase into new requirements)
- Expected: "..." (source: description | comment)
- Actual: "..." (source: ...)
- Repro (if present): "..."
- Constraints from comments: "..."

### Agent responsibility
- In scope: ...
- Out of scope: ...
- Module(s): `ms-search` | `ms-orders` | `ms-listings` | `ms-consumer` | ...
- Files likely involved: ...

### Fix approach
1. ...
2. ...

### Test case scenarios (complete list — do not omit a category)
For each scenario: **name**, **given / when / then**, **expected**, **coverage**
(JUnit class#method, or "listed only" with why).

**Success**
- ...

**Failure / negative**
- ...

**Edge**
- ...

**Race**
- ...

### Module suite
- `mvn test` in each changed module

### Local curls
- Paste in the Step 9 chat handoff only. Do **not** write a `.sh` file.

### Dual validation (ES + DB)
- Pair: what it proves, ES query, SQL query, match rule (ES field = SQL column)
- Or `N/A` on a side with reason. See Step 6c.

### Risks
- Security / PII / secrets: ...
- Production impact: ...
- Rollback: revert the PR / branch
```

Fill every category in Step 2 even if some cases will later be "listed only".
Do not skip race or edge because they are harder. See Step 6 for the required
scenario catalog.

**Stop and ask** when any of these are true:

- No expected vs actual behavior can be inferred **from the description or comments**
- The ticket is a product/design question, not a code change
- The fix belongs in a different repo
- The change would be a large architectural rewrite with no acceptance criteria
- The ticket asks for production config, DNS, or secret rotation (do not do this here)
- Implementing would require guessing anything not in the ticket or the code

Do **not** invent requirements. Don't hallucinate — just follow the comments
and descriptions.

## Step 3 — Branch

Base: `origin/dev` (this repo's default branch).

```bash
git fetch origin
git checkout -B bugfixes/<KEY> origin/dev
```

Branch name is exactly `bugfixes/<KEY>` (example: `bugfixes/SR-3099`).

If the branch already exists locally or on the remote, reuse it after confirming it
was created for this ticket. Do not commit on `dev` or `main`.

## Step 4 — Implement the correct fix

The job is to fix **the problem in the ticket description and comments**,
not a nearby cleanup and not an assumed similar bug.

1. Search the matching module first (`ms-search`, `ms-orders`, `ms-listings`,
   `ms-consumer`, `ms-tickets`, `ms-reports`, `ms-schedulers`, `ms-payments`,
   `ms-common-util`). way-services is a multi-module Java 21 / Spring Boot repo.
2. Trace expected vs actual **as written in the ticket** into code you opened.
   Do not invent a root cause that the description/comments do not support.
3. Implement the smallest change that makes actual match expected. Do not paper
   over the bug (hardcoded values, catching-and-ignoring, UI-only workarounds)
   unless the ticket explicitly asks for that.
4. After the change, re-read the ticket summary, description, and comments and
   confirm the fix covers every in-scope acceptance point. If it does not,
   keep iterating before tests/PR.
5. Match existing style in the touched files. Reference the ticket key in code
   comments only when neighboring code already does (for example `SR-3099: ...`).
6. Do not hardcode secrets, tokens, passwords, or environment credentials.
7. Do not edit `.idea/`, local `application.properties` secrets, or untracked IDE files.
8. Do not expand into drive-by refactors, dependency upgrades, or formatting-only diffs.
9. Do not create `local.sh`, `local-validation/`, or any validation `*.sh` file.

If investigation shows the ticket is already fixed on `dev`, stop, cite the evidence,
and do not open an empty PR.

## Step 5 — Tests, then all module tests

For each changed behavior, add or update JUnit 5 tests in the same module.
Cover **all four kinds** below. Follow existing test style (Mockito + JUnit 5
is typical).

| Kind | Must include |
|------|----------------|
| Success | Happy path from the ticket's expected result |
| Failure | Invalid input, missing entity, unauthorized, downstream error, or the reported bug's old input |
| Edge | Null/empty, boundary values, and any case called out in comments — see Step 6 catalog |
| Race | Concurrent / overlapping / duplicate / stale-read cases that apply to this change — see Step 6 catalog |

For race tests, prefer `CountDownLatch` / overlapping calls / optimistic-lock
conflicts. If a race cannot be unit-tested, **list** it (`listed only` + why).

**5a. Ticket tests first** (from the module directory):

```bash
mvn -q test -Dtest=TheTestClass
```

If the change is controller/API-level, also run the matching `*ControllerTest`.

**5b. After the fix is done, run all tests in every changed module:**

```bash
mvn test
```

Do this in each module whose code changed. Do not skip 5b. Paste the summary
(tests run / failures / errors) in the final chat message and in the PR.

If tests fail because of the fix, iterate until they pass. If they fail for
unrelated environment reasons (missing Docker, ES integration), note it in the
PR and still run everything that does not need those extras. Default `mvn test`
already excludes `es-integration` in ms-search.

Do not claim the ticket is fixed if 5a or 5b was not run.

## Step 6 — Test-case scenarios + curls in chat only

### 6a. Scenario catalog (required)

Enumerate **every applicable scenario** for this ticket. Do not stop at happy
path. Walk the changed code. Edge/race come from **ticket comments/description**
and the **types in files you read** — do not invent extra product behavior.
If a catalog item cannot apply, write `N/A — <reason>` rather than omitting
the heading.

**Success**
- Primary happy path from the ticket expected result
- Alternate valid inputs that should also succeed (optional fields present,
  different but valid enum / duration / status)

**Failure / negative**
- Invalid input / validation rejection
- Missing entity (404 / empty / domain error — match existing API)
- Unauthorized / forbidden (if the endpoint is authenticated)
- Downstream error (timeout, 5xx from a client) — no crash, no invented data
- Original buggy input from the ticket (proves the old actual no longer happens)

**Edge** (include all that the types/fields allow)
- Null vs empty vs blank; zero vs null money/numeric; min / max / just-outside
- Empty collection vs collection of nulls; unknown enum; unicode / special chars
- Timezone / DST / midnight / end-of-day; pagination page 0, last page, size 0 / over max
- Already deleted / terminal status; idempotent retry; partial payload
- Large payload / many children (if the change iterates a list)

**Race** (include all that the mutation/read path allows)
- Concurrent updates of the same entity (lost update); concurrent unique-key create
- Double-submit of create / checkout / payment / booking
- Read-modify-write without version/lock (TOCTOU stale read)
- Duplicate event/message processed twice; timeout then retry creating a duplicate
- Overlapping scheduler tick vs inbound API write; stale cache vs fresh write

For each listed scenario, state:

1. **Name**
2. **Given / when / then**
3. **Expected** (HTTP status and/or field, or exception / no-op)
4. **Coverage**: `JUnit <Class>#<method>` **or** `listed only — <why>`

Repeat this full list in the Step 2 briefing, the PR body, and the Step 9
handoff. Incomplete edge or race sections are not done.

### 6b. Local curls (chat only — never a file)

Discover the real HTTP method, path, and payload from the controller and from
existing Postman collections under `<module>/src/main/resources/postman/`.
Use the module's local port:

| Module | Default port |
|--------|----------------|
| `ms-listings` | 8080 |
| `ms-search` | 8081 |
| `ms-schedulers` | 8082 |
| `ms-consumer` | 8085 |
| `ms-common-util` | 8087 |

If `server.port` in that module's `application.properties` differs, or the
module is not in the table (`ms-orders`, `ms-tickets`, `ms-payments`,
`ms-reports`), read that file and use its port.

Paste **success, failure, and at least one edge** curl in the Step 9 handoff.
If a race can be shown with two curls, paste those too (label them as concurrent
/ double-submit). Rules:

- Use `http://localhost:<port>` — never a real JWT, password, or API key
  (`${TOKEN:-}` placeholder only)
- Use `-i` or `-w "\nHTTP %{http_code}\n"`
- Label each curl and state expected HTTP status and the field that proves
  the ticket is fixed

If there is no HTTP surface (pure scheduler/DAL), say so in chat and give the
closest local command. Still **do not** write a `.sh` file.

**Forbidden:** `Write` / `touch` / `chmod` / `git add` of `local.sh`,
`local-validation/**`, any generated curl script, or `.sql` query files.

### 6c. Dual validation — Elasticsearch + DB (required, chat only)

Paste **paired** queries so the user can confirm ES matches the DB for the
same entity. Do not invent names — read them from this change’s code:
`SearchProperties` (`way_es_index`, `parking_inventory`,
`parking_rolling_avail`), `IndexRequest`, document classes, indexer DAOs;
`@Table` / `@Entity` / `@Query` / indexer SQL (`WAY_PLATFORM` when used).

Each pair: **Proves** / **ES** (`GET ${ES_URL:-http://localhost:9203}/<index>/_search`
or `/_doc/${ID}`, `_source` only fields under test) / **DB** (same placeholder
id) / **Match** (`ES.<field> == SQL.<column>`, note index lag).

Search / index / price-card / listing tickets **must** include both sides.
Otherwise `N/A — <reason>` on the unused side. No secrets, PII, `.sql` files,
or commits of these queries. Templates: [examples.md](examples.md).

## Step 7 — Security / Way engineering checks

Before commit:

- No secrets, API keys, tokens, or credentials in the diff
- No `local.sh` / `local-validation/` / generated `*.sh` or `.sql` query files in the diff
- Least privilege: do not widen auth or `permitAll` unless the ticket requires it
- Sensitive data stays encrypted in transit/at rest; do not log PII
- If monitoring, runbooks, or Confluence need updates, say so in the PR (do not
  silently skip)
- Production-impacting behavior: include scope, risk, rollback, and validation
  in the PR body

## Step 8 — Files changed, commit, and PR

Commit only when the fix and module tests are in place. Do **not** commit
`local.sh`, `local-validation/`, curl scripts, or `.sql` files.

**Files changed (required).** Before commit, collect the exact list from git
and keep it for the PR body and Step 9. Do not skip this.

```bash
git status --short
git diff --name-status origin/dev
```

After commit, confirm against the branch:

```bash
git diff --name-status origin/dev...HEAD
```

For each path, record `A` (added), `M` (modified), or `D` (deleted) plus a
one-line why. Include tests. Exclude `local.sh`, `local-validation/`, `.idea/`,
`.DS_Store`, secrets, and any `.sh`/`.sql` validation files (leave untracked).

Commit message style (why, with ticket key):

```
SR-3099: restore hourly parking card prices from nested VPA rates
```

Then push and open the PR with `gh`. Base branch is `dev`.

```bash
git push -u origin HEAD
gh pr create --base dev --title "<KEY>: <short summary>" --body "$(cat <<'EOF'
...
EOF
)"
```

Immediately after create, capture the number and URL (required for Step 9):

```bash
gh pr view --json number,url,title
```

Then fill the Step 8 PR description template with this run's real values.
Use it as `gh pr create` `--body` **and** reprint it in Step 9 as a
copy-paste fence for GitHub before merge.

The final message must include **PR #<number>**, URL, files, scenarios,
curls, **paired ES + DB queries**, and the paste-ready PR description.

PR description template (fill every section; this is what gets pasted):

```markdown
## Summary
- Fixes [<KEY>](https://wayglobal.atlassian.net/browse/<KEY>): <one sentence>
- <what changed>

## Files changed
- `M` `<path>` — <why>
- `A` `<path>` — <why>

## Ticket
- **Expected:** ... (quote description/comment)
- **Actual (before):** ... (quote)
- **Agent responsibility:** ... (in-scope only from those quotes)

## Test case scenarios
### Success
- <name> — given/when/then — expected — coverage

### Failure / negative
- ...

### Edge
- ... (or N/A — reason)

### Race
- ... (or N/A — reason)

## Test plan
- [ ] Ticket tests: `mvn -q test -Dtest=...` passed
- [ ] All module tests: `mvn test` passed in <module(s)>

## How to test
1. Start `<module>` locally (port <port>). Optional: `export TOKEN='your-local-jwt'`
2. No `.sh` on the branch. Paste **Success / Failure / Edge** curls (expect status + field).
3. Dual validation: run the paired ES query and SQL; confirm the match rule.

## Dual validation
- **Proves / ES / DB / Match:** ... (or N/A — reason)

## Risk
- Impact: ...
- Rollback: revert this PR / delete branch `bugfixes/<KEY>`

## Validation after merge
- ...
```

Request reviewers only if the user named them. Leave the PR open for approval;
do not merge.

Do **not** transition the Jira status or comment on the ticket unless the user asks.

## Step 9 — Final handoff (required)

Do not omit PR number, files, scenarios, curls, dual-validation queries, or
the paste-ready description. Shape:

```markdown
## Done: <KEY>

**PR:** #<number>
**PR URL:** https://github.com/Way-com/way-services/pull/<number>
**Branch:** bugfixes/<KEY>

### Files changed
- `M` `<path>` — <why>

### Tests
- Ticket tests: ...
- Module `mvn test`: ...

### Test case scenarios
#### Success / Failure / Edge / Race
- <name> — given/when/then — expected — JUnit ... | listed only

### Local curls (chat only — no file on the branch)
Start `<module>` (port <port>). Optional `TOKEN`. Paste success, failure,
edge, and race curls with `${TOKEN:-}`. Label expected status/field.

### Dual validation (ES + DB, chat only)
- **Proves:** ...
- **ES:** ...
- **DB:** ...
- **Match:** ES.`<field>` == SQL.`<column>` (or N/A — reason)
```

**After** that Done block, print:

`### PR description — copy-paste into GitHub before merge`

Then: open PR #<number> → Description → replace with the next fence → Save.
The fence body is the **filled** Step 8 PR description only (real paths,
tests, scenarios, curls, ES + DB pairs). No commentary inside the fence.

After the fence, add the Step 10 self-assessment. The turn is not done until
that scorecard is posted.

If `gh pr view` failed, parse the number from the create URL (`.../pull/987`
→ `#987`) and still print it.

## Step 10 — Self-assessment (required, last message)

Once everything else is done, grade **your own run** before ending the turn.
This is a critical review of the agent's work, not a victory lap. Post it as
the last section of the final message, after the paste-ready PR description.

**Evidence rule.** Mark a row `Met` only when you can point to something real
from this run: a command you ran and its output, a file you opened, a test
count, a JUnit method name, a PR number. If the only support is "I intended
to" or "it should be fine", the row is `Partial` or `Missed`. Never grade a
step `Met` because the skill told you to do it.

**No grade inflation.** These caps are hard:

- Step 5b (`mvn test` per changed module) not run or not green → overall
  **cannot exceed 6/10**, and the Tests row is `Missed`
- Any edge or race heading left out (rather than `N/A — reason`) → Coverage
  row is `Missed`
- A required Step 6b curl or Step 6c ES/DB pair missing without a stated
  reason → that row is `Missed`
- Any invented API, table, index, field, or acceptance criterion found during
  self-review → Honesty row is `Missed` and overall **cannot exceed 4/10**;
  fix or retract the invention before ending the turn

**Repair before reporting.** If a row would be `Missed` for a step you can
still complete (a test run, a missing curl, a forgotten `git diff`), go do it
now and then grade the corrected state. Only report `Missed` for work that is
genuinely blocked — and say what blocks it.

Scorecard shape:

```markdown
### Self-assessment — <KEY>

| # | Criterion | Verdict | Evidence |
|---|-----------|---------|----------|
| 1 | Ticket fidelity — fix matches quoted description/comments, no added scope | Met / Partial / Missed | quote + file:line |
| 2 | Root cause, not symptom — no hardcode, swallow, or UI-only patch | ... | what changed and why it's the cause |
| 3 | Scope discipline — minimal diff, no drive-by refactor or formatting churn | ... | `git diff --name-status` result |
| 4 | Coverage — success/failure/edge/race all present (JUnit or justified `listed only`) | ... | counts per category |
| 5 | Tests actually executed — 5a ticket tests + 5b `mvn test` per changed module | ... | tests run / failures / errors per module |
| 6 | Curls in chat, no `.sh`/`.sql` on the branch | ... | curls pasted + clean `git status --short` |
| 7 | Dual validation — ES + DB pair with names read from code, or `N/A — reason` | ... | index / table names + where found |
| 8 | Security & SOC2 — no secrets, no widened auth, no PII logged | ... | what was checked in the diff |
| 9 | PR & handoff — number, URL, files with why, paste-ready description | ... | PR #<n> |
| 10 | Honesty — every claim above is backed by a real command or file | ... | anything retracted |

**Overall: <n>/10** — <one sentence, blunt>

**Weakest link:** <the row most likely to cause a review comment or a bug, and why>

**Unverified — user must confirm:** <claims not proven locally: integration
tests needing Docker/ES, prod-only behavior, curls not actually executed
against a running service>

**Assumptions made:** <should be none; list any and mark whether the ticket or
the code supports it, or say "none — all facts from ticket + files read">

**Would do differently:** <one or two concrete process changes for the next ticket>

**Way engineering follow-ups:** <Confluence / runbook / monitoring / alerting
updates this change implies, or "none">
```

If the user asked only to analyze (no code, no PR), still post the scorecard,
scoped to what the run produced. Rows 2, 3, 5, 6, 8, and 9 become
`N/A — analyze-only run`; grade row 1 on ticket fidelity, row 4 on the
completeness of the *planned* scenario catalog, row 7 on the *planned* ES + DB
pairs, and row 10 on honesty.

## Stop conditions

- Ticket not found / no permission
- Ambiguous acceptance criteria (do not guess; ask)
- Implementing would require hallucinating missing description/comment/code facts
- Fix would require production secrets, DNS, or Cloudflare changes
- User asked only to *analyze* — stop after Step 2 (still include scenarios,
  dual-validation ES + DB pairs, and the Step 10 scorecard)

When you stop for any reason above, still close with a short Step 10
self-assessment: what you verified, what blocked you, and exactly what input
you need to continue. A stopped run is graded on how clearly it hands the
blocker back, not on how much code it wrote.

## Additional resources

- Invocation examples: [examples.md](examples.md)
