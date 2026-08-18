# Jira-bugfixingAgent examples

## How to invoke

In a **new** Agent chat in this repo (not the setup chat):

```
Fix SR-3099
```

```
Analyze SR-3127 and fix it. Open a PR when tests pass.
```

```
Ticket: SR-3082
```

The agent should load this skill, pull the Jira ticket, brief you on
responsibility and **all** test-case scenarios (success, failure, edge, race),
then branch `bugfixes/<KEY>` / root-cause fix / tests / list files changed /
curls in chat (no `.sh` file) / PR / paste-ready PR description.

The agent must **not** create, commit, or push `local.sh` or
`local-validation/<KEY>.sh`.

Analyze-only (no code, no PR):

```
Analyze SR-3099. Do not change code.
```

## Example briefing (Step 2)

```markdown
## Ticket SR-3099
**Summary:** Hourly parking card shows $0 instead of the nested VPA rate
**Type / status / priority:** Bug / To Do / High

### What the ticket says
- Expected: Hourly card uses the nested VPA rate for the selected duration
- Actual: Card renders $0 after the rate-table change
- Repro: Search airport parking → open listing → hourly card
- Constraints from comments: Do not change daily/overnight pricing

### Agent responsibility
- In scope: Restore hourly price mapping in ms-search from nested VPA rates
- Out of scope: Redesign of the price-stats API
- Module(s): `ms-search`
- Files likely involved: parking card mapper / price stats service

### Fix approach
1. Trace where hourly card price is populated
2. Read nested VPA rate instead of the flattened field
3. Keep daily/overnight paths unchanged

### Test case scenarios
**Success**
- Nested VPA present — given a listing with nested hourly rate 12.50, when
  price stats are built, then hourly card shows 12.50 —
  `PriceStatsServiceTest#hourlyCardUsesNestedVpaRate`

**Failure / negative**
- Missing nested VPA — given no nested rate, when stats are built, then no
  crash and no invented price —
  `PriceStatsServiceTest#missingNestedVpaDoesNotInventPrice`

**Edge**
- Zero vs null rate — given nested rate 0 vs null, when stats are built, then
  0 displays as 0 and null omits/does not show $0 from a missing field —
  `PriceStatsServiceTest#zeroRateVsNullRate`
- Empty rates list — given empty nested collection, then same as missing —
  `PriceStatsServiceTest#emptyNestedRates`

**Race**
- Concurrent search requests for the same listing — given two overlapping
  calls, when both read nested VPA, then both return the same hourly rate
  (read-only path; no lost update) —
  `PriceStatsServiceTest#concurrentReadsReturnSameHourlyRate`
  or listed only if the mapper is stateless and a latch test adds no signal

### Module suite
- `mvn test` in `ms-search`

### Local curls
- Chat handoff only. Do not write `local-validation/SR-3099.sh`.

### Risks
- Security / PII / secrets: none
- Production impact: search result prices for hourly parking
- Rollback: revert the PR
```

## Example PR title

```
SR-3099: restore hourly parking card prices from nested VPA rates
```

## Example local curls (chat only)

Do **not** write a file. Paste curls in the Step 9 handoff:

```
# Start ms-search locally on 8081, then copy-paste from chat.
# Token is $TOKEN, never a real JWT.
```

Include a success curl (expected 200, hourly rate present), a failure curl
(expected 4xx or empty rate, no crash), an edge curl (zero vs null), and a
race pair if the API can double-submit.

## Example final handoff (Step 9)

After the fix, the last message must look like this so you can review files,
run curls from chat, and merge the PR. There is **no** `.sh` on the branch.

```markdown
## Done: SR-3099

**PR:** #987
**PR URL:** https://github.com/Way-com/way-services/pull/987
**Branch:** bugfixes/SR-3099

### Files changed
- `M` `ms-search/src/main/java/.../PriceStatsService.java` — read nested VPA hourly rate
- `A` `ms-search/src/test/java/.../PriceStatsServiceTest.java` — success/failure/edge/race cases

### Tests
- Ticket tests: PriceStatsServiceTest passed
- Module `mvn test`: ms-search passed

### Test case scenarios
#### Success
- Nested VPA present → hourly card shows that rate — JUnit PriceStatsServiceTest#hourlyCardUsesNestedVpaRate

#### Failure / negative
- Missing nested VPA → no crash, no invented price — JUnit PriceStatsServiceTest#missingNestedVpaDoesNotInventPrice

#### Edge
- Zero vs null rate — JUnit PriceStatsServiceTest#zeroRateVsNullRate
- Empty nested rates — JUnit PriceStatsServiceTest#emptyNestedRates

#### Race
- Concurrent reads of the same listing return the same hourly rate —
  JUnit PriceStatsServiceTest#concurrentReadsReturnSameHourlyRate

### Local curls (chat only — no file on the branch)
1. Start `ms-search` locally (port 8081).
2. Optional: `export TOKEN='your-local-jwt'`
3. Copy-paste the success, failure, edge, and (if applicable) race curls from chat.
```

### PR description — copy-paste into GitHub before merge

Open PR #987 → Description → replace with the block below → Save.

```markdown
## Summary
- Fixes [SR-3099](https://wayglobal.atlassian.net/browse/SR-3099): hourly parking card uses nested VPA rate
- Read nested VPA hourly rate instead of the flattened field

## Files changed
- `M` `ms-search/src/main/java/.../PriceStatsService.java` — read nested VPA hourly rate
- `A` `ms-search/src/test/java/.../PriceStatsServiceTest.java` — success/failure/edge/race cases

## Ticket
- **Expected:** Hourly card uses the nested VPA rate
- **Actual (before):** Card renders $0
- **Agent responsibility:** Restore hourly price mapping in ms-search

## Test case scenarios
### Success
- Nested VPA present → hourly card shows that rate — PriceStatsServiceTest#hourlyCardUsesNestedVpaRate

### Failure / negative
- Missing nested VPA → no crash, no invented price — PriceStatsServiceTest#missingNestedVpaDoesNotInventPrice

### Edge
- Zero vs null rate — PriceStatsServiceTest#zeroRateVsNullRate
- Empty nested rates — PriceStatsServiceTest#emptyNestedRates

### Race
- Concurrent reads return the same hourly rate — PriceStatsServiceTest#concurrentReadsReturnSameHourlyRate

## Test plan
- [x] Ticket tests passed
- [x] `mvn test` passed in ms-search

## How to test
1. Start ms-search locally (port 8081)
2. Optional: `export TOKEN='your-local-jwt'`
3. Run the success/failure/edge curls from the agent chat (no `.sh` on the branch)

## Risk
- Impact: search result prices for hourly parking
- Rollback: revert this PR / delete branch `bugfixes/SR-3099`

## Validation after merge
- Hourly parking card shows nested VPA rate, not $0
```

