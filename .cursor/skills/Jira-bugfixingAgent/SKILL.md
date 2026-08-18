---
name: jira-bugfixing-agent
description: >-
  Jira-bugfixingAgent: analyzes a Jira ticket, implements the correct
  root-cause fix on branch bugfixes/<ticket-key>, lists every file changed,
  runs all module tests, reports all test-case scenarios (success, failure,
  edge, race), pastes local curls in chat only (never writes or commits a
  .sh file), opens a pull request, and after the PR exists prints a filled
  GitHub-ready description the user can paste into the PR before merge. Use
  when the user provides a Jira ticket number or key (for example SR-3099)
  and asks to fix the issue, implement the ticket, analyze the ticket, or
  create a PR for it.
---

# Jira-bugfixingAgent

Retrieve the Jira ticket, state responsibility, implement the **correct
root-cause fix**, run ticket tests plus the module suite, list **all**
test-case scenarios (success, failure, edge, race), paste local curls **in
chat only**, open a PR on `bugfixes/<ticket-key>`, and after the PR exists
print a **filled PR description** the user can paste into GitHub before merge.

Do **not** start coding until analysis is complete. If the ticket is too vague, stop and ask.

## Hard rules — no local shell files

**Never** create, write, chmod, stage, commit, or push any of these:

- `local.sh`
- `local-validation/<KEY>.sh`
- `local-validation/` (any file)
- any other generated `*.sh` curl/validation script

Do not add those paths if they already exist untracked. Do not mention a
"local curl file" in the PR or commit. Put curls in the chat handoff only.

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
- [ ] 2. State responsibility, implementation plan, and ALL test-case scenarios
- [ ] 3. Create branch bugfixes/<KEY> from origin/dev
- [ ] 4. Implement the correct root-cause fix (minimal, ticket-scoped)
- [ ] 5. Add tests for success, failure, edge, and race; run those, then all module tests
- [ ] 6. Paste local curls in chat only — do NOT write or commit any .sh file
- [ ] 7. Security / secrets / SOC2 check
- [ ] 8. Collect files changed (path + why), commit, push, open PR
- [ ] 9. Handoff: PR number, files, scenarios, curls, and paste-ready PR description
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

Also collect:

- Issue type, status, priority, components, labels
- Linked issues (blockers, clones, related bugs)
- Acceptance criteria / repro steps in description or comments
- Environment (prod / staging / iOS / Android / web)
- Affected service or API path if mentioned

If the ticket is missing or inaccessible, stop and tell the user.

## Step 2 — Analyze and state responsibility

Post this briefing **before** changing code. This is the agent's contract for the run.

```markdown
## Ticket <KEY>
**Summary:** ...
**Type / status / priority:** ...
**Reported by / comments:** ...

### What the ticket says
- Expected: ...
- Actual: ...
- Repro (if present): ...
- Constraints from comments: ...

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

### Risks
- Security / PII / secrets: ...
- Production impact: ...
- Rollback: revert the PR / branch
```

Fill every category in Step 2 even if some cases will later be "listed only".
Do not skip race or edge because they are harder. See Step 6 for the required
scenario catalog.

**Stop and ask** when any of these are true:

- No expected vs actual behavior can be inferred
- The ticket is a product/design question, not a code change
- The fix belongs in a different repo
- The change would be a large architectural rewrite with no acceptance criteria
- The ticket asks for production config, DNS, or secret rotation (do not do this here)

Do not invent requirements that are not in the ticket or comments.

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

The job is to fix **the problem described in the ticket**, not a nearby cleanup.

1. Search the matching module first (`ms-search`, `ms-orders`, `ms-listings`,
   `ms-consumer`, `ms-tickets`, `ms-reports`, `ms-schedulers`, `ms-payments`,
   `ms-common-util`). way-services is a multi-module Java 21 / Spring Boot repo.
2. Trace expected vs actual from the ticket into the code. Find the root cause
   (wrong field, missing null check, bad query, incorrect mapping, etc.).
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

After the fix is written, keep a running list of every path you created or
edited and a one-line reason. You will print this list in the PR and in Step 9.

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

For race tests, prefer existing module patterns. Typical Java approaches:
`CountDownLatch` + thread pool, two overlapping service calls, optimistic-lock
version conflict, duplicate-message / idempotent consumer. If the code path
cannot express a race in unit tests, still **list** the scenario (coverage:
listed only + why) and add the closest sequential test (e.g. retry after
partial success).

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
path. Walk the changed code and include cases from this catalog when they can
happen. If a catalog item cannot apply, write `N/A — <reason>` under that
heading rather than omitting the heading.

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
closest local command (job trigger, SQL, or nearest public API). Still **do
not** write a `.sh` file.

**Forbidden:** `Write` / `touch` / `chmod` / `git add` of `local.sh`,
`local-validation/**`, or any generated curl script. Do not finish the run
by creating those files.

## Step 7 — Security / Way engineering checks

Before commit:

- No secrets, API keys, tokens, or credentials in the diff
- No `local.sh` / `local-validation/` / generated `*.sh` in the diff
- Least privilege: do not widen auth or `permitAll` unless the ticket requires it
- Sensitive data stays encrypted in transit/at rest; do not log PII
- If monitoring, runbooks, or Confluence need updates, say so in the PR (do not
  silently skip)
- Production-impacting behavior: include scope, risk, rollback, and validation
  in the PR body

## Step 8 — Files changed, commit, and PR

Commit only when the fix and all module tests are in place. Do **not** commit
`local.sh`, `local-validation/`, or any generated curl script. Do not commit
unless this skill's workflow is running to completion (the user already asked
to fix and open a PR).

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
one-line why. Include tests. **Exclude** `local.sh`, `local-validation/`,
`.idea/`, `.DS_Store`, and local secrets. If git status shows a `.sh`
validation file, leave it untracked and do not add it.

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
curls, and the paste-ready PR description — not only a compare URL.

PR description template (fill every section; this is what gets pasted):

```markdown
## Summary
- Fixes [<KEY>](https://wayglobal.atlassian.net/browse/<KEY>): <one sentence>
- <what changed>

## Files changed
- `M` `<path>` — <why>
- `A` `<path>` — <why>

## Ticket
- **Expected:** ...
- **Actual (before):** ...
- **Agent responsibility:** ...

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

Do not omit PR number, files, scenarios, curls, or the paste-ready
description. Shape:

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
```

**After** that Done block, print:

`### PR description — copy-paste into GitHub before merge`

Then: open PR #<number> → Description → replace with the next fence → Save.
The fence body is the **filled** Step 8 PR description only (real paths,
tests, scenarios, curls). No commentary inside the fence. Required once
the PR exists.

If `gh pr view` failed, parse the number from the create URL (`.../pull/987`
→ `#987`) and still print it.

## Stop conditions

- Ticket not found / no permission
- Ambiguous acceptance criteria
- Fix would require production secrets, DNS, or Cloudflare changes
- User asked only to *analyze* the ticket — then stop after Step 2 (still
  include the full test-case scenario list in the briefing)

## Additional resources

- Invocation examples: [examples.md](examples.md)
