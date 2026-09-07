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

`investment_allocation` is the one tool with no `repo`/`org`/`team`
scoping at all, same as its API endpoint -- Jira work items have no
repo relationship.

## Connecting a client

### Claude Code

```
claude mcp add --transport http gitultra http://localhost:8100/mcp \
    -H "Authorization: Bearer <your ENG_API_KEY value>"
```

### Claude Desktop

Add to your MCP server config:

```json
{
  "mcpServers": {
    "gitultra": {
      "type": "http",
      "url": "http://localhost:8100/mcp",
      "headers": {
        "Authorization": "Bearer <your ENG_API_KEY value>"
      }
    }
  }
}
```

## Getting access

Same as [API Access](api-access.md) -- available on **Team** and
**Enterprise** plans, see [gitultra.com](https://gitultra.com) for
plan details or to apply for beta access. Full setup details, auth
model, and troubleshooting live in
[gitultra-mcp](https://github.com/GitUltraHQ/gitultra-mcp)'s own
README.
