# AI Agent Access (Team / Enterprise)

`gitultra-mcp` is an [MCP](https://modelcontextprotocol.io/) (Model
Context Protocol) server so an AI agent -- Claude Desktop, Claude Code,
or anything else that speaks MCP -- can query your engineering metrics
conversationally, instead of you (or your dashboard) calling
[the API](api-access.md) by hand.

It's a thin layer on top of that same API: every tool call is one
HTTP request to your own already-running `eng-api` instance, using the
same credentials you already set up for it. No new database
connection, no new data collected -- if you already run `eng-api`,
turning this on is just adding one more container.

## Requirements

- A running `eng-api` instance (see [API Access](api-access.md)) --
  this has no other dependency
- Same **Team**/**Enterprise** plan `eng-api` itself needs

## Setup

Using the `eng-metrics-suite-pro` compose bundle, `gitultra-mcp` is
already defined as a service -- set these in `.env` (mostly reusing
what `eng-api` already needs):

```
ENG_API_KEY=<same value you already set for eng-api>
GITULTRA_LICENSE_KEY=<your license key, covering "gitultra-mcp">
GITULTRA_MCP_PORT=8100
```

If your existing license only covers `eng-api`, ask for a reissue that
also includes `gitultra-mcp` in its product list -- one key can cover
both.

Standalone (outside the compose bundle):

```
docker run --rm -p 8100:8000 \
    -e ENG_API_BASE_URL=http://eng-api:8000 \
    -e ENG_API_KEY=<same value as eng-api's API_KEY> \
    -e GITULTRA_LICENSE_KEY=<your license key> \
    ghcr.io/gitultrahq/gitultra-mcp:latest
```

## Tools

One tool per API endpoint, same data, same scoping rules as
[API Access](api-access.md) describes -- `gitultra-mcp` just forwards
whatever you (or the agent) pass through and relays the API's response
or error. Every tool accepts either `period`
(`"last-week"`/`"last-month"`/`"last-quarter"`) or explicit `start`/
`end`, so an agent can say "last week" instead of computing an exact
date range.

| Tool | Same as |
|---|---|
| `author_activity` | Author activity |
| `review_health` | Review health |
| `repo_trends` | Repo trends |
| `contributor_concentration` | Contributor concentration |
| `recency_skew` | Recency skew |
| `team_repo_activity_mismatch` | Team-to-repo activity mismatch |
| `contributor_transition_impact` | Contributor departure/reassignment impact |
| `ticket_lead_time` | Ticket-to-first-commit lead time |
| `ticket_scope_mismatch` | Ticket scope mismatch |
| `planning_signature` | Planning signature |
| `cycle_time` | Cycle time |
| `deployment_frequency` | Deployment frequency |
| `lead_time` | Lead time for changes |
| `change_failure_rate` | Change failure rate |
| `mttr` | Mean time to restore |
| `ai_usage` | AI usage |
| `investment_allocation` | Investment allocation |
| `author_distribution_trend` | Author distribution trend |
| `reviewer_distribution_trend` | Reviewer distribution trend |
| `commits_after_open_distribution_trend` | Commits-after-open distribution trend |
| `executive_report` | Executive report (**Enterprise plan only** -- see [API Access](api-access.md)) |

`investment_allocation` and `executive_report` are the two tools with
no `repo`/`org`/`team` scoping at all, same as their API endpoints --
Jira work items have no repo relationship for the former, and the
latter is deliberately one org-wide view rather than a drill-down tool.

**`repo_trends`, `deployment_frequency`, `lead_time`,
`change_failure_rate`, and `mttr` return a rendered chart image instead
of JSON when called from Claude Desktop.** Every other client --
including Claude Code, which has no general way to render an inline
image -- gets exactly the same JSON these tools have always returned.
For `repo_trends`, `deployment_frequency`, `change_failure_rate`, and
`mttr`, the chart shows org-wide **p50/p90 trend lines across whatever
repos your `repo`/`org`/`team` scoping matched, not a line per repo**
-- the underlying JSON still has full per-repo detail, this is a
readability choice for the chart specifically.

### Manager tools (require email verification)

These six work differently from every tool above: instead of your
shared `ENG_API_KEY` alone, they're scoped to a single manager's own
team(s), gated by that manager proving control of their own email
address first.

| Tool | What it does |
|---|---|
| `verify_manager_email` | Step 1: emails a one-time code to a manager's on-file address. |
| `submit_manager_code` | Step 2: exchanges that code for a session token, valid 24 hours. |
| `get_contributor_summary` | Per-contributor activity for one of your managed teams. |
| `compare_periods` | Current vs. immediately-preceding period, side by side, for one team. |
| `get_team_trend` | Weekly commit/PR trend for one team. |
| `email_my_team_report` | Emails a fresh PDF of one team's report to your own on-file address. |

#### Manager verification flow

1. Ask your agent to call `verify_manager_email` with your email.
   Check the inbox tied to your manager row for a 6-digit code
   (expires in 10 minutes).
2. Have it call `submit_manager_code` with that code. On success, this
   returns a session token.
3. **Hold onto that token for the rest of the conversation** and pass
   it as `token` on every subsequent call to
   `get_contributor_summary`/`compare_periods`/`get_team_trend`/
   `email_my_team_report` -- `gitultra-mcp` keeps no memory between
   tool calls (see the "thin layer on top of the API" note above), so
   your agent client is what carries the token forward, not the
   server. After 24 hours the token expires; start again from step 1.

Non-reportable contributors (an admin-configured privacy setting --
see [Generating Reports](generating-reports.md#optional-columns-manager-reporting-and-pseudonymization))
appear as a stable alias like "Contributor A" in these tools' output,
never their real name/email. That reduces identifiability, it doesn't
achieve legal anonymity -- see that same page for the full caveat.

### Org role tools (require email verification)

Above team level, three org-wide roles -- Admin, Executive, Director --
verify through the **exact same `verify_manager_email`/
`submit_manager_code` flow** above (a person needs a row in
`org_roles.csv` instead of, or in addition to, a manager row -- see
[Generating Reports](generating-reports.md#org_rolescsv-org-level-roles-admin-executive-director)).
No separate verification tools. The three roles aren't a strict
hierarchy -- see [API Access](api-access.md#org-level-roles-admin-executive-director)
for the full permission matrix.

| Tool | Roles | What it does |
|---|---|---|
| `get_org_team_trend` | Admin, Executive, Director | Weekly commit/PR trend, org-wide when `team` is omitted, or for one arbitrary team when given (Admin/Director only). |
| `get_admin_contributor_summary` | Admin | Per-contributor activity for any team, not just ones you manage. |
| `get_admin_period_comparison` | Admin | Current vs. preceding period for any team. |
| `email_admin_team_report` | Admin | Emails a fresh PDF for any team to your own on-file address. |
| `upload_team_map_csv` | Admin | Replaces `team_map.csv` entirely and reloads it live -- no restart. |
| `upload_org_roles_csv` | Admin | Replaces `org_roles.csv` entirely and reloads it live -- always rejected if it would leave zero `admin` rows. |

Both upload tools take `csv_content` as a plain string (the full new
file contents) and require the underlying `eng-api` install to have a
writable path configured for the target file -- see [Getting
Started](getting-started.md). A malformed upload returns a readable
error and changes nothing; the tool doesn't distinguish "eng-api
rejected the file" from any other error shape, so check the message
for specifics.

## Connecting a client

### Claude Code

```
claude mcp add --transport http gitultra http://localhost:8100/mcp \
    -H "Authorization: Bearer <your ENG_API_KEY value>"
```

### Claude Desktop

Current Desktop builds manage connectors through **Settings → Connectors
→ Add custom connector**, not a config file:

1. URL: `https://your-instance:8100/mcp` -- Desktop requires a real
   `https://` URL; a plain `http://` address (including `localhost`)
   is rejected outright, even for a same-machine service. If
   `gitultra-mcp` isn't already behind HTTPS the way the rest of your
   instance is, put it behind the same reverse proxy/TLS termination
   you're using for `eng-api`.
2. Choose **"No sign-in"**, not "Sign in now" -- this server uses a
   static Bearer header, not OAuth. Picking "No sign-in" reveals a
   custom headers field.
3. Add header `Authorization: Bearer <your ENG_API_KEY value>`.

Some older Desktop builds instead read a config file directly:

```json
{
  "mcpServers": {
    "gitultra": {
      "type": "http",
      "url": "https://your-instance:8100/mcp",
      "headers": {
        "Authorization": "Bearer <your ENG_API_KEY value>"
      }
    }
  }
}
```

If your build supports both, the Connectors UI takes precedence --
use it first and only fall back to editing the config file if your
version of Desktop doesn't have a Connectors UI at all.

## Getting access

Same as [API Access](api-access.md) -- available on **Team** and
**Enterprise** plans, see [gitultra.com](https://gitultra.com) for
plan details or to apply for beta access. `executive_report` is
Enterprise-only, same as its underlying API endpoint -- your license
just needs to cover `"executive-report"`, no separate `gitultra-mcp`
grant needed for that one tool specifically. The org role tools need
the same license coverage the manager tools already need -- no
additional grant, whichever plan already gets you manager reporting
also gets you org roles. Full setup details, auth model, and
troubleshooting live in
[gitultra-mcp](https://github.com/GitUltraHQ/gitultra-mcp)'s own
README.
