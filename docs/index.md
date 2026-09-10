# eng-metrics-suite

A self-hosted engineering metrics pipeline: import commit and PR/MR stats
from your GitHub, GitLab, Bitbucket, or Azure DevOps org into Postgres,
and generate a PDF report (team/author activity, PR review health, cycle
time, repo trends) from it. Optionally add a Jira integration for two
more capabilities: change failure rate/MTTR (the other two DORA
metrics) and an investment allocation report — see
[Jira Integration](jira-integration.md).

The [eng-metrics-suite](https://github.com/GitUltraHQ/eng-metrics-suite) repo
is just the `docker-compose.yml` needed to run it — the application
images (`git-processor`, `pr-processor`, `eng-reports`, and optionally
`issue-processor`) are published pre-built at
[ghcr.io/gitultrahq](https://github.com/GitUltraHQ?tab=packages).
Free to use; see [LICENSE](https://github.com/GitUltraHQ/eng-metrics-suite/blob/master/LICENSE).

!!! tip "Want API access, AI agent access, or the Executive Report?"
    Everything on this site works with the free, self-hosted tier. For
    [API Access](api-access.md) and [AI Agent Access](mcp-access.md)
    (query your metrics from Claude Desktop or Claude Code) on the
    **Team**/**Enterprise** plans, or the
    [Executive Report](generating-reports.md#executive-report-enterprise-plan)
    (Enterprise plan only), sign up or apply for beta access at
    [gitultra.com](https://gitultra.com).

## Requirements

- Docker + Docker Compose

## The pieces

- **`git-processor`** — clones queued repos and imports commit stats.
- **`pr-processor`** — imports PR/MR stats from the same repos' vendor APIs.
- **`eng-reports`** — reads both, renders a PDF report on demand.
- **`issue-processor`** (optional) — imports Jira data for change
  failure rate/MTTR and/or investment allocation. Idles cleanly if
  unconfigured — see [Jira Integration](jira-integration.md).
- **Postgres** — shared database all of the above read/write.

All application services log structured JSON to stdout and are
controlled by the same `docker compose` commands — see
[Getting Started](getting-started.md) to begin.
