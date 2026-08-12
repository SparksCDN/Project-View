# GitHub's issue APIs for a read-only client

Research for [#3](https://github.com/SparksCDN/Project-View/issues/3). Part of map [#1](https://github.com/SparksCDN/Project-View/issues/1).

**Date:** 2026-08-12. **Verified against:** `api.github.com` (GitHub.com, not GHES), `gh` 2.97.0, token
scopes `gist, read:org, repo, workflow`, REST API versions `2022-11-28` and `2026-03-10`.

Every empirical claim below was produced by running the quoted command against this repo
(`SparksCDN/Project-View`, issues #1–#8) or, where a state this repo does not contain was needed,
against a public repo. Documentation-only claims are marked as such.

---

## Executive summary

**The frontier computation is cleanly supported, in a single API call, with no N+1 and no special
headers.** This is the load-bearing finding.

Both the parent/child edge and the "is this blocked right now" signal ride along on the ordinary
issue payload:

- `parent_issue_url` — the sub-issue parent, on every issue in a list response.
- `issue_dependencies_summary.blocked_by` — **count of *open* blockers**. Zero means unblocked.
- `assignees`, `labels`, `state`, `created_at`, `updated_at`, `closed_at` — all present.

So the frontier of a map is:

```
GET /repos/{owner}/{repo}/issues?state=open&per_page=100
  filter: parent_issue_url == <map issue url>
       && issue_dependencies_summary.blocked_by == 0
       && assignees == []
```

One request. No preview headers. No per-issue follow-up. The same holds in GraphQL, in one query.

The rest of this document is the evidence, the alternatives, and the sharp edges.

---

## 1. Sub-issues

### Status

Generally available. Sub-issues went GA on 2025-04-09
([changelog](https://github.blog/changelog/2025-04-09-evolving-github-issues-and-projects/)).
The REST endpoints are documented with no preview or beta notice
([REST: sub-issues](https://docs.github.com/en/rest/issues/sub-issues?apiVersion=2022-11-28)).

**No preview or opt-in header is required.** Verified with a bare `curl` carrying only
`Authorization`, `Accept: application/vnd.github+json`, and `X-GitHub-Api-Version: 2022-11-28`:

```bash
curl -s -o /dev/null -w "%{http_code}\n" \
  -H "Authorization: Bearer $(gh auth token)" \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  https://api.github.com/repos/SparksCDN/Project-View/issues/1/sub_issues
# => 200
```

`X-Accepted-Oauth-Scopes` came back **empty** on that endpoint, i.e. no scope beyond whatever grants
access to the repository itself.

### Available in both REST and GraphQL

**REST — read endpoints:**

| Method | Path |
|---|---|
| GET | `/repos/{owner}/{repo}/issues/{issue_number}/sub_issues` |
| GET | `/repos/{owner}/{repo}/issues/{issue_number}/parent` |

```bash
gh api "repos/SparksCDN/Project-View/issues/1/sub_issues?per_page=100"
# => 7 full issue objects (#2–#8), each with labels, assignees, state,
#    sub_issues_summary, issue_dependencies_summary

gh api "repos/SparksCDN/Project-View/issues/3/parent"
# => the full issue object for #1

gh api "repos/SparksCDN/Project-View/issues/1/parent"
# => HTTP 404 {"message":"No parent issue found", ...}
```

Note the 404: `GET /parent` is **not** a safe "is there a parent?" probe — it errors on root issues.
Use `parent_issue_url` from the issue body instead, which is `null` for roots.

**GraphQL:** `Issue.parent`, `Issue.subIssues`, `Issue.subIssuesSummary`. Confirmed present by
schema introspection:

```bash
gh api graphql -f query='{ __type(name:"Issue"){ fields{ name } } }'
# includes: parent, subIssues, subIssuesSummary, trackedIssues, trackedInIssues
```

Field descriptions from introspection:

- `parent` — "The parent entity of the issue." (no arguments)
- `subIssues` — "A list of sub-issues associated with the Issue." (connection: `first`/`last`/`after`/`before`)
- `subIssuesSummary` — `{ total, completed, percentCompleted }`

Do **not** confuse `subIssues` with `trackedIssues`/`trackedInIssues` — the latter are the older
task-list-checkbox mechanism, a different relationship.

### Cost of fetching a whole map's children

Three options, cheapest first:

1. **Zero extra calls.** Every issue in a plain list response already carries `parent_issue_url`.
   Fetch the repo's issues once and group by parent client-side. This is the right move for a
   single-project board.
2. **One call per map**: `GET /issues/{n}/sub_issues?per_page=100`.
   Returns up to 100 children (100 is also GitHub's hard cap on sub-issues per parent, per
   [Adding sub-issues](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/adding-sub-issues)),
   so a map never needs more than one page.
3. **One GraphQL query** covering every issue and its children at once — see §4.

Documented structural limits: **100 sub-issues per parent, 8 levels of nesting**
([docs](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/adding-sub-issues)).
A `first: 100` on `subIssues` therefore never truncates.

---

## 2. Issue dependencies (`blocked_by` / `blocks`)

### Status

Generally available since 2025-08-21
([changelog: Dependencies on issues](https://github.blog/changelog/2025-08-21-dependencies-on-issues/)),
across REST, GraphQL, and webhooks.

### `issue_dependencies_summary` is returned by default

**No special header, no field selection, no opt-in.** It is in the default JSON body of both the
single-issue and the list endpoints:

```bash
gh api repos/SparksCDN/Project-View/issues/5 | jq 'keys'
# includes: issue_dependencies_summary, sub_issues_summary, parent_issue_url
```

```bash
gh api "repos/SparksCDN/Project-View/issues?state=all&per_page=100" \
  | jq '[.[] | {number, issue_dependencies_summary, parent_issue_url}]'
```

Actual output at time of research (2026-08-12):

| Issue | `blocked_by` | `total_blocked_by` | `blocking` | `total_blocking` | `parent_issue_url` |
|---|---|---|---|---|---|
| #1 | 0 | 0 | 0 | 0 | `null` |
| #2 | 0 | 0 | 1 | 1 | `.../issues/1` |
| #3 | 0 | 0 | 2 | 2 | `.../issues/1` |
| #4 | 0 | 0 | 2 | 2 | `.../issues/1` |
| #5 | 1 | 1 | 0 | 0 | `.../issues/1` |
| #6 | 1 | 1 | 0 | 0 | `.../issues/1` |
| #7 | 2 | 2 | 1 | 1 | `.../issues/1` |
| #8 | 2 | 2 | 0 | 0 | `.../issues/1` |

This matches the real edges exactly (#5←#2, #6←#3, #7←#3+#4, #8←#4+#7).

### The critical semantic: `blocked_by` counts OPEN blockers only

Schema introspection gives the definitions:

```bash
gh api graphql -f query='{ __type(name:"IssueDependenciesSummary"){ fields{ name description } } }'
```

- `blockedBy` — "Count of issues this issue is blocked by"
- `totalBlockedBy` — "Total count of issues this issue is blocked by (**open and closed**)"
- `blocking` — "Count of issues this issue is blocking"
- `totalBlocking` — "Total count of issues this issue is blocking (**open and closed**)"

This repo cannot demonstrate the difference (all four blockers are open), so it was verified against
a public repo that has a **closed** blocker:

```bash
gh api "repos/grafana/grafana/issues/122583" | jq '{number, state, issue_dependencies_summary}'
# {"number":122583,"state":"closed",
#  "issue_dependencies_summary":{"blocked_by":0,"total_blocked_by":1,
#                                "blocking":0,"total_blocking":0}}

gh api "repos/grafana/grafana/issues/122583/dependencies/blocked_by" | jq '[.[]|{number,state}]'
# [{"number":122581,"state":"closed"}]
```

One blocker, it is closed, and `blocked_by` is `0` while `total_blocked_by` is `1`. Same result for
`grafana/grafana#130184` (open issue, one closed blocker, `blocked_by: 0`, `total_blocked_by: 1`).

**Consequence: `blocked_by == 0` is exactly the "not blocked right now" predicate the frontier
needs, and GitHub computes it server-side.** The app does not have to walk edges or check blocker
states itself. Use `blocked_by`, never `total_blocked_by`, for the frontier.

### Reading the actual edges

**REST:**

| Method | Path |
|---|---|
| GET | `/repos/{owner}/{repo}/issues/{issue_number}/dependencies/blocked_by` |
| GET | `/repos/{owner}/{repo}/issues/{issue_number}/dependencies/blocking` |

```bash
gh api "repos/SparksCDN/Project-View/issues/8/dependencies/blocked_by" | jq '[.[].number]'
# => [4, 7]
gh api "repos/SparksCDN/Project-View/issues/4/dependencies/blocking" | jq '[.[].number]'
# => [7, 8]
```

Both return full issue objects (with a `repository` key, so cross-repo edges are identifiable), and
paginate with `per_page` (default 30, max 100) / `page`
([docs](https://docs.github.com/en/rest/issues/issue-dependencies)).
`X-Accepted-Oauth-Scopes` was empty on these too.

**GraphQL:** `Issue.blockedBy` and `Issue.blocking`, both connections:

- `blockedBy` — "A list of issues that are blocking this issue."
- `blocking` — "A list of issues that this issue is blocking."

The changelog states a cap of 50 issues per relationship type, so `first: 50` is sufficient.

---

## 3. Labels, state, timestamps, assignees — what one call returns

`GET /repos/{owner}/{repo}/issues` returns full issue objects. Keys observed under API version
`2022-11-28`:

```
active_lock_reason  assignee  assignees  author_association  body  closed_at  closed_by
comments  comments_url  created_at  events_url  html_url  id  issue_dependencies_summary
labels  labels_url  locked  milestone  node_id  number  parent_issue_url
performed_via_github_app  pinned_comment  reactions  repository_url  state  state_reason
sub_issues_summary  timeline_url  title  updated_at  url  user
```

`labels` is the full label object array (`name`, `color`, `description`, ...), `assignees` is the
full user array. **There is no N+1 for labels, state, timestamps, assignees, parent, sub-issue
progress, or blocked/blocking counts.** The only thing requiring extra calls is the *identity* of
individual dependency edges — and the frontier does not need them.

### API version matters slightly

```bash
gh api /versions
# => ["2026-03-10","2022-11-28"]
```

Under `X-GitHub-Api-Version: 2026-03-10` the payload drops the deprecated singular `assignee` and
`user`-adjacent legacy field set slightly — observed keys lose `assignee` (plural `assignees`
remains). `gh` defaults to `2022-11-28`; the current docs examples show `2026-03-10`. Pin one
explicitly rather than inheriting a client default.

Repo label palette (needed to render triage labels with GitHub's own colours) is one more call:

```bash
gh api "repos/SparksCDN/Project-View/labels?per_page=100"
# => 18 labels including the five triage labels and the wayfinder:* set, with color + description
```

---

## 4. REST vs GraphQL for this shape of read

### REST: fewest *concepts*, more bytes

```bash
gh api "repos/{owner}/{repo}/issues?state=all&per_page=100&sort=updated&direction=desc"
```

- 1 request per 100 issues, 1 rate-limit point each.
- Everything needed for the board and the frontier is in the response.
- **Cost measured:** one page of 100 issues from `cli/cli` was **768 KB** and took **1.14 s**.
  Most of that is `body` markdown, which you cannot deselect in REST.

### GraphQL: fewer bytes, edges included, still one round trip per 100

This query returns the complete board *including* every dependency and sub-issue edge:

```graphql
query($owner:String!, $name:String!, $cursor:String) {
  rateLimit { cost remaining nodeCount }
  repository(owner:$owner, name:$name) {
    issues(first:100, after:$cursor, states:[OPEN,CLOSED],
           orderBy:{field:UPDATED_AT, direction:DESC}) {
      pageInfo { hasNextPage endCursor }
      totalCount
      nodes {
        number title state stateReason createdAt updatedAt closedAt url
        author { login }
        issueType { name }
        milestone { title }
        labels(first:20)   { nodes { name color description } }
        assignees(first:10){ nodes { login } }
        parent { number title }
        subIssuesSummary { total completed percentCompleted }
        subIssues(first:100) { totalCount nodes { number state } }
        issueDependenciesSummary { blockedBy blocking totalBlockedBy totalBlocking }
        blockedBy(first:50) { totalCount nodes { number state repository { nameWithOwner } } }
        blocking(first:50)  { totalCount nodes { number state repository { nameWithOwner } } }
      }
    }
  }
}
```

Run it with:

```bash
gh api graphql -F owner=SparksCDN -F name=Project-View -F query=@query.graphql
```

Against this repo it returned the complete graph in one call — all seven parent edges and all six
blocking edges, exactly matching reality.

**Measured cost** (same query against `cli/cli`, 6,295 issues, first page):

| Metric | Value |
|---|---|
| `rateLimit.cost` | **5 points** |
| `rateLimit.nodeCount` | 23,100 |
| Response size | **74 KB** |
| Wall clock | 1.54 s |

Ten times smaller than the REST equivalent, because GraphQL lets you omit `body`.

**Cost scales with nested connections, not with results.** Measured on this repo:

| Query shape | cost | nodeCount |
|---|---|---|
| `issues(first:100)` scalars only | 1 | 100 |
| `+ labels(20) + assignees(10)` | 2 | 3,100 |
| `+ summaries + parent` (no new connections) | **1** | 100 |
| `+ blockedBy(50) + blocking(50) + subIssues(100)` | 3 | 20,100 |
| full query above | 5 | 23,100 |

Note row 3: **`issueDependenciesSummary`, `subIssuesSummary`, and `parent` are free** — they are not
connections, so they add zero cost and zero nodes. A frontier-only GraphQL query costs **1 point**.

### Verdict for this app

| | REST | GraphQL |
|---|---|---|
| Round trips for 300 issues | 3 | 3 |
| Rate-limit points for 300 issues | 3 | 15 (full edges) / 3 (summaries only) |
| Bytes for 300 issues | ~2.3 MB | ~220 KB |
| Dependency *edges* included | No (2 extra calls per issue) | Yes |
| Conditional requests (ETag → 304, free) | **Yes** | No |
| Client complexity | Trivial | Query + cursor handling |

Neither wins on round trips — both cap at 100 items per request. The real trade is:

- **GraphQL** if the app wants the dependency *graph* (to draw edges, or compute transitive
  blocking). It is the only way to get edges without N+1, and it is 10× lighter on the wire.
- **REST** if the app only needs the board and the frontier, because then ETags make repeat polls
  literally free (see §5) and GraphQL has no equivalent.

A defensible hybrid: REST + ETag for the polling loop, GraphQL for the one-off "show me the
dependency graph" view. Both are cheap enough that this is not a forced choice at personal scale.

---

## 5. Rate limits and what they imply for refresh

### Primary limits

| Resource | Limit | Source |
|---|---|---|
| REST, authenticated user token | **5,000 requests/hour** | [docs](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api?apiVersion=2022-11-28) |
| REST, unauthenticated | 60 requests/hour | same |
| GraphQL, user token | **5,000 points/hour** | [docs](https://docs.github.com/en/graphql/overview/rate-limits-and-node-limits-for-the-graphql-api) |
| Search (REST + GraphQL `search`) | **30 requests/minute** | measured, below |
| Code search | 10/minute | measured, below |

Measured live:

```bash
gh api rate_limit | jq '.resources | {core, graphql, search, code_search}'
# core:        {"limit":5000, ...}
# graphql:     {"limit":5000, ...}
# search:      {"limit":30, ...}
# code_search: {"limit":10, ...}
```

REST and GraphQL budgets are **separate buckets** — spending GraphQL points does not consume REST
requests. Confirmed: `core.used` and `graphql.used` tracked independently across the session.

### Secondary limits (apply on top)

From the [REST rate-limit docs](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api?apiVersion=2022-11-28):

- No more than **100 concurrent requests** (shared across REST and GraphQL).
- No more than **900 points/minute** for REST endpoints; **2,000 points/minute** for GraphQL.
- No more than 90 seconds of CPU time per 60 seconds of real time.

None of these are reachable by a single-project desktop client.

### Conditional requests make polling nearly free — verified

The docs say: *"Making a conditional request does not count against your primary rate limit if a
`304` response is returned and the request was made while correctly authorized with an
`Authorization` header."*
([best practices](https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api?apiVersion=2022-11-28))

Verified directly:

```bash
TOK=$(gh auth token)
U="https://api.github.com/repos/SparksCDN/Project-View/issues?per_page=100"
ET=$(curl -s -D - -o /dev/null -H "Authorization: Bearer $TOK" "$U" \
     | grep -i '^etag:' | sed 's/^[Ee]tag: //' | tr -d '\r')

# three unconditional GETs
for i in 1 2 3; do curl -s -D - -o /dev/null -H "Authorization: Bearer $TOK" "$U" \
  | grep -iE '^(HTTP/2|x-ratelimit-used)'; done
# HTTP/2 200 / x-ratelimit-used: 54
# HTTP/2 200 / x-ratelimit-used: 55
# HTTP/2 200 / x-ratelimit-used: 56

# three conditional GETs
for i in 1 2 3; do curl -s -D - -o /dev/null -H "Authorization: Bearer $TOK" \
  -H "If-None-Match: $ET" "$U" | grep -iE '^(HTTP/2|x-ratelimit-used)'; done
# HTTP/2 304 / x-ratelimit-used: 56
# HTTP/2 304 / x-ratelimit-used: 56
# HTTP/2 304 / x-ratelimit-used: 56
```

**Three 304s cost zero rate-limit budget.** This is the single most important operational fact for
refresh design.

### What this implies for refresh frequency

- The issues list response carries `Cache-Control: private, max-age=60, s-maxage=60`. No
  `X-Poll-Interval` header was present on this endpoint.
- With ETags, a 60-second poll of a 3-page project costs **0 points** while nothing changes, and
  3 points on the poll after a change. Even a naive unconditional 60-second poll of 3 pages costs
  180 requests/hour — 3.6% of budget.
- Practical guidance: **poll no faster than 60 s** (matching `max-age`), always send
  `If-None-Match`, and only re-fetch the pages whose ETag changed. Rate limits are a non-issue for
  this app; the constraint is politeness, not budget.
- For incremental refresh, REST supports `since` (issues updated at or after a timestamp) — verified:

  ```bash
  gh api "repos/SparksCDN/Project-View/issues?state=all&since=2026-08-12T21:41:00Z&per_page=100" \
    | jq '[.[].number]'
  # => [3, 2]
  ```

  **Caveat — verified, and important:** `since` filters on `updated_at`, and **adding a dependency
  does not bump `updated_at`**. Issue #8 received two `blocked_by_added` events at 21:40:56 and
  21:40:57, yet its `updated_at` is still its creation time:

  ```bash
  gh api "repos/SparksCDN/Project-View/issues/8" | jq -c '{created_at, updated_at, issue_dependencies_summary}'
  # {"created_at":"2026-08-12T21:40:21Z","updated_at":"2026-08-12T21:40:21Z",
  #  "issue_dependencies_summary":{"blocked_by":2,"total_blocked_by":2,...}}

  gh api "repos/SparksCDN/Project-View/issues/events?per_page=100" \
    | jq -c '[.[] | select(.event=="blocked_by_added" and .issue.number==8) | {created_at, blocker:.blocked_by.number}]'
  # [{"created_at":"2026-08-12T21:40:57Z","blocker":7},{"created_at":"2026-08-12T21:40:56Z","blocker":4}]
  ```

  The same holds for `parent_issue_added`. **An incremental sync built on `since` alone will miss
  structural changes and show a stale frontier.** Either refetch the full list (cheap, and free with
  ETags) or watch the events endpoint below.

- A better structural-change feed is the repo-wide issue events endpoint, which surfaces the
  relationship mutations directly:

  ```bash
  gh api "repos/{owner}/{repo}/issues/events?per_page=100" \
    | jq '[.[].event] | group_by(.) | map({(.[0]): length}) | add'
  # => {"assigned":3,"blocked_by_added":5,"blocking_added":5,
  #     "labeled":8,"parent_issue_added":7,"sub_issue_added":7}
  ```

  `blocked_by_added` events include a `blocked_by: {number, title, state, repository}` object plus
  the full affected issue. The payloads are very large (each event embeds two full issue objects),
  so this is a change-detection signal, not a primary data source.

---

## 6. Pagination

| API | Page size | Max | Verified |
|---|---|---|---|
| REST | `per_page`, default 30 | **100** | `per_page=101` silently clamps; 8-issue repo returned 8, and `cli/cli` returned exactly 100 |
| GraphQL | `first`/`last` | **100** | `first: 101` → `EXCESSIVE_PAGINATION` error (below) |

```bash
gh api graphql -f query='{ repository(owner:"SparksCDN",name:"Project-View"){ issues(first:101){ nodes{number} } } }'
# {"type":"EXCESSIVE_PAGINATION", "message":"Requesting 101 records on the `issues`
#  connection exceeds the `first` limit of 100 records."}
```

REST returns a `Link` header with cursor-based `next` (GitHub now uses opaque `after=` cursors even
in REST — do **not** construct `page=N` URLs yourself):

```
Link: <https://api.github.com/repositories/1332450454/issues?state=all&per_page=3&after=Y3Vyc29yOnYyOpLP...&page=2>; rel="next"
```

GraphQL uses `pageInfo { hasNextPage endCursor }`:

```json
{"totalCount":8,"pageInfo":{"hasNextPage":true,"endCursor":"Y3Vyc29yOnYyOpK0MjAyNi0wOC0xMlQyMTo0MDoxNVrPAAAAATIaI_0="},"nodes":[{"number":1},{"number":2},{"number":3}]}
```

### Cost of a project with a few hundred issues

| Issues | REST requests | REST bytes | GraphQL requests | GraphQL points | GraphQL bytes |
|---|---|---|---|---|---|
| 100 | 1 | ~770 KB | 1 | 5 | ~74 KB |
| 300 | 3 | ~2.3 MB | 3 | 15 | ~220 KB |
| 1,000 | 10 | ~7.7 MB | 10 | 50 | ~740 KB |

Extrapolated from the measured 100-issue page against `cli/cli`. Even 1,000 issues is 10 requests
and ~1% of the hourly budget — and with ETags, near-zero on subsequent refreshes.

### GraphQL node limit

Max **500,000 nodes** per query, and `first`/`last` must be 1–100
([docs](https://docs.github.com/en/graphql/overview/rate-limits-and-node-limits-for-the-graphql-api)).
Verified the enforcement is a *potential*-node calculation, not an actual-results one:

```bash
gh api graphql -f query='{ repository(owner:"SparksCDN",name:"Project-View"){
  issues(first:100){ nodes{ subIssues(first:100){ nodes{ blockedBy(first:100){ nodes{number} } } } } } } }'
# {"type":"MAX_NODE_LIMIT_EXCEEDED","message":"By the time this query traverses to the
#  blockedBy connection, it is requesting up to 1,000,000 possible nodes which exceeds
#  the maximum limit of 500,000."}
```

On an 8-issue repo. The limit is on the query's declared shape, so keep nesting to two levels and
use `first: 50` on dependency connections.

---

## 7. What is NOT readable

### Nothing structural is write-gated

Sub-issues, parents, dependency summaries, and dependency edges are all readable with plain read
access. `X-Accepted-Oauth-Scopes` was empty on every structural endpoint tested.

### Projects v2 needs its own scope — verified

```bash
gh api graphql -f query='{ repository(owner:"SparksCDN",name:"Project-View"){ projectsV2(first:5){ nodes{ title } } } }'
# {"errors":[{"type":"INSUFFICIENT_SCOPES","message":"Your token has not been granted the
#   required scopes to execute this query. The 'projectsV2' field requires one of the
#   following scopes: ['read:project'], but your token has only been granted the:
#   ['gist', 'read:org', 'repo', 'workflow'] scopes."}]}
```

`repo` scope is **not** enough for Projects v2. If the app ever renders project board columns or
custom fields, it needs `read:project` on top. (Curiously, `Issue.projectItems` did *not* error
with the same token, returning `{"totalCount":0}` — so the gate is inconsistent across fields.
Do not rely on that.)

### Issue types are org-scoped

`Issue.issueType` exists and is queryable, but for a user-owned repo it is always `null` and the
type catalogue is unavailable:

```bash
gh api "repos/SparksCDN/Project-View/issues/types"          # => HTTP 404 Not Found
gh api graphql -f query='{ repository(owner:"SparksCDN",name:"Project-View"){ issueTypes(first:10){ nodes{ name } } } }'
# => {"data":{"repository":{"issueTypes":null}}}
```

The app must treat issue type as optional and absent on personal repos. The `wayfinder:<type>`
labels carry that information instead, which is exactly why the skills use labels.

### Search qualifiers for structure do not work — CONFLICT WITH DOCS

The dependencies changelog claims search filters `is:blocked`, `is:blocking`, `blocked-by:`, and
`blocking:` exist ([changelog](https://github.blog/changelog/2025-08-21-dependencies-on-issues/)).
**They returned nothing in every test**, ~30 minutes after the edges were created, on both the
GraphQL `search` connection and REST `search/issues`:

| Query (scoped to this repo) | Expected | Actual |
|---|---|---|
| `is:issue is:blocked` | #5 #6 #7 #8 | **0 results** |
| `is:issue is:blocking` | #2 #3 #4 #7 | **0 results** |
| `is:issue blocked-by:SparksCDN/Project-View#3` | #6 #7 | **0 results** |
| `is:issue blocking:SparksCDN/Project-View#7` | #3 #4 | **0 results** |
| `is:issue parent-issue:SparksCDN/Project-View#1` | #2–#8 | **0 results** |
| `is:issue no:parent-issue` | #1 | #1 ✅ |
| `is:issue no:sub-issue` | #2–#8 | #2–#8 ✅ |
| `is:issue no:blocked-by` | #1 #2 #3 #4 | **all 8** ❌ |
| `is:issue label:wayfinder:map` | #1 | #1 ✅ |
| `is:issue is:open no:assignee label:wayfinder:insighting` | #5–#8 | #5–#8 ✅ |

Control probes establish how failures present:

- `is:bogusqualifierxyz` → 0 results (unknown `is:` values match nothing).
- `has:bogusxyz` → all 8, and `no:bogusxyz` → all 8 (unknown `has:`/`no:` values are **silently
  ignored**). So `has:sub-issue` and `has:parent-issue` returning all 8 means those qualifiers are
  not real, and `no:blocked-by` returning all 8 means it is being ignored too.
- `no:parent-issue` and `no:sub-issue` *do* work, so the search index does know sub-issue structure.
- The search index does have these issues (`repo:... wayfinder` free-text returned 5 of 8).

**Cannot be fully explained.** Either the dependency qualifiers were never shipped as documented,
were renamed, apply only to certain plans/contexts, or the dependency search index lags by more than
30 minutes. The GitHub search-syntax docs
([searching issues and PRs](https://docs.github.com/en/search-github/searching-on-github/searching-issues-and-pull-requests))
document **none** of these qualifiers, which weakens the changelog's claim.

**Recommendation: do not build the frontier on the search API.** Beyond the reliability problem it
is capped at 30 requests/minute and 1,000 results, whereas `issue_dependencies_summary.blocked_by`
on the plain list endpoint is authoritative, unlimited, and free.

---

## 8. Token scopes for read-only access to private repos

*This section reports what is possible. Choosing the mechanism is
[#6](https://github.com/SparksCDN/Project-View/issues/6).*

### Classic PATs

| Need | Minimum scope |
|---|---|
| Read issues in **public** repos | `public_repo` (or no scope at all — public data is readable unauthenticated, at 60 req/hr) |
| Read issues in **private** repos | **`repo`** |
| Projects v2 fields | `+ read:project` |
| Org membership / team data | `+ read:org` |

There is **no read-only issues scope in the classic model.** The narrowest scope that reaches a
private repository's issues is `repo`, which
[grants](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/scopes-for-oauth-apps)
*"full access to public and private repositories including read and write access to code, commit
statuses, repository invitations, collaborators, deployment statuses, and repository webhooks."*
A classic PAT for a read-only app is therefore massively over-privileged — the app's read-only
promise would be enforced only by the app's own code, not by the token.

### Fine-grained PATs

Per
[Permissions required for fine-grained PATs](https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens?apiVersion=2022-11-28),
each of these requires **Issues: Read**:

- `GET /repos/{owner}/{repo}/issues`
- `GET /repos/{owner}/{repo}/issues/{issue_number}`
- `GET /repos/{owner}/{repo}/issues/{issue_number}/sub_issues`
- `GET /repos/{owner}/{repo}/issues/{issue_number}/parent`
- `GET /repos/{owner}/{repo}/issues/{issue_number}/dependencies/blocked_by`
- `GET /repos/{owner}/{repo}/issues/{issue_number}/dependencies/blocking`

So the minimum fine-grained set is:

| Permission | Access | Why |
|---|---|---|
| **Metadata** | Read | Mandatory baseline; required by all repo-scoped fine-grained tokens |
| **Issues** | Read | Everything in this document |
| *Projects* | *Read* | *Only if the app renders Projects v2 boards or fields* |

Plus repository selection (specific repos, or all repos in an account).

This is a genuine read-only credential — the token itself cannot write, so the app's read-only
promise is enforced by GitHub, not by discipline. That is a real argument for fine-grained PATs, but
the decision belongs to #6.

**Not empirically verified:** creating and testing a fine-grained PAT requires a human at
`github.com/settings/tokens`, which is out of scope for a read-only research pass. The scope table
above is from the documentation. The classic-PAT behaviour (`repo` works, `read:project` missing
blocks `projectsV2`) *was* verified with the live session token.

### GitHub Apps / OAuth apps

Both are possible and use the same fine-grained permission names (`issues: read`,
`metadata: read`). GitHub App *user* access tokens inherit the user's 5,000 req/hr primary limit
([docs](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api?apiVersion=2022-11-28)).
Installation tokens get a separate, generally larger allowance. Again: #6's call, not this ticket's.

---

## 9. Copy-pasteable recipes

### A. The whole board, one REST call

```bash
gh api "repos/{owner}/{repo}/issues?state=all&per_page=100&sort=updated&direction=desc"
```

Everything needed to render: labels, assignees, state, timestamps, `parent_issue_url`,
`sub_issues_summary`, `issue_dependencies_summary`.

### B. The frontier of a map, one REST call

```bash
MAP=1; OWNER=SparksCDN; REPO=Project-View
gh api "repos/$OWNER/$REPO/issues?state=open&per_page=100" \
  | jq --arg map "https://api.github.com/repos/$OWNER/$REPO/issues/$MAP" '
      [ .[]
        | select(.parent_issue_url == $map)
        | select(.issue_dependencies_summary.blocked_by == 0)
        | select((.assignees | length) == 0)
        | {number, title, labels: [.labels[].name]} ]'
```

Verified against this repo at 2026-08-12T21:52Z. Result was `[]` — and correctly so: the three
unblocked children (#2, #3, #4) were all assigned at that moment. Dropping the assignee filter
returns exactly `[4, 3, 2]`, and inverting the block filter returns exactly
`[{8, blocked_by:2}, {7, blocked_by:2}, {6, blocked_by:1}, {5, blocked_by:1}]` — the real structure.

### C. The frontier of a map, one GraphQL call (cost: 1 point)

```graphql
{
  repository(owner:"SparksCDN", name:"Project-View") {
    issue(number:1) {
      number title
      subIssuesSummary { total completed percentCompleted }
      subIssues(first:100) {
        totalCount
        nodes {
          number title state url
          assignees(first:5) { totalCount nodes { login } }
          labels(first:10) { nodes { name color } }
          issueDependenciesSummary { blockedBy totalBlockedBy blocking totalBlocking }
        }
      }
    }
  }
}
```

Filter client-side: `state == "OPEN" && issueDependenciesSummary.blockedBy == 0 &&
assignees.totalCount == 0`.

### D. The full dependency graph, one GraphQL call per 100 issues (cost: 5 points)

See §4. Paginate on `pageInfo.endCursor`.

### E. Free polling with ETags

```bash
# first fetch
curl -sD headers.txt -H "Authorization: Bearer $TOKEN" \
  -H "Accept: application/vnd.github+json" \
  -H "X-GitHub-Api-Version: 2022-11-28" \
  "https://api.github.com/repos/OWNER/REPO/issues?state=all&per_page=100" -o page1.json
ETAG=$(grep -i '^etag:' headers.txt | sed 's/^[Ee]tag: //' | tr -d '\r')

# subsequent polls — 304 costs nothing
curl -s -w '%{http_code}\n' -o /dev/null \
  -H "Authorization: Bearer $TOKEN" -H "If-None-Match: $ETAG" \
  "https://api.github.com/repos/OWNER/REPO/issues?state=all&per_page=100"
```

---

## 10. Implications for the spec

1. **The skills-aware premise holds.** GitHub exposes the parent/child edge and a server-computed
   "blocked right now" count on every issue in a list response. The frontier is a client-side filter
   over one HTTP request. No preview headers, no N+1, no write scopes.
2. **`blocked_by`, not `total_blocked_by`.** This distinction is the whole frontier. Getting it
   wrong makes issues look permanently blocked after their blockers close.
3. **Use `parent_issue_url`, not `GET /parent`.** The dedicated endpoint 404s on root issues.
4. **Never use the search API for structure.** It is unreliable for dependencies today and rate
   limited to 30/min.
5. **Refresh is not a rate-limit problem.** ETagged polling at 60 s is free. Design the refresh
   story around freshness and UX, not budget. But **do not build incremental sync on `since`** —
   dependency and parent changes do not bump `updated_at`, so the frontier would go stale silently.
   Refetch the full list; ETags make it free.
6. **Issue types will be `null`** on personal repos. Ticket type must come from `wayfinder:*`
   labels — which is what the skills already do.
7. **Projects v2 is a separate scope decision.** If the app never renders project boards, it never
   needs `read:project`, and the token stays smaller.
8. **A fine-grained PAT with `Issues: Read` + `Metadata: Read` can enforce the read-only promise at
   the credential layer.** Whether to use one is #6's call, but the capability exists and is worth
   knowing about when that decision is made.

---

## Sources

Documentation:

- [REST: sub-issues](https://docs.github.com/en/rest/issues/sub-issues?apiVersion=2022-11-28)
- [REST: issue dependencies](https://docs.github.com/en/rest/issues/issue-dependencies)
- [REST: rate limits](https://docs.github.com/en/rest/using-the-rest-api/rate-limits-for-the-rest-api?apiVersion=2022-11-28)
- [REST: best practices (conditional requests)](https://docs.github.com/en/rest/using-the-rest-api/best-practices-for-using-the-rest-api?apiVersion=2022-11-28)
- [GraphQL: rate limits and node limits](https://docs.github.com/en/graphql/overview/rate-limits-and-node-limits-for-the-graphql-api)
- [GraphQL: forming calls / authentication](https://docs.github.com/en/graphql/guides/forming-calls-with-graphql)
- [Permissions required for fine-grained PATs](https://docs.github.com/en/rest/authentication/permissions-required-for-fine-grained-personal-access-tokens?apiVersion=2022-11-28)
- [Scopes for OAuth apps](https://docs.github.com/en/apps/oauth-apps/building-oauth-apps/scopes-for-oauth-apps)
- [Adding sub-issues (limits)](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/adding-sub-issues)
- [Creating issue dependencies](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/creating-issue-dependencies)
- [Searching issues and pull requests](https://docs.github.com/en/search-github/searching-on-github/searching-issues-and-pull-requests)
- [Changelog: Evolving GitHub Issues and Projects (GA, 2025-04-09)](https://github.blog/changelog/2025-04-09-evolving-github-issues-and-projects/)
- [Changelog: Dependencies on issues (GA, 2025-08-21)](https://github.blog/changelog/2025-08-21-dependencies-on-issues/)

Empirical: all `gh api` / `curl` commands quoted inline, run 2026-08-12 against
`SparksCDN/Project-View` (issues #1–#8), plus `cli/cli` and `grafana/grafana` for scale and for a
closed-blocker case this repo does not contain. GraphQL schema facts come from live introspection
against `api.github.com/graphql`.
