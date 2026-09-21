# MCP audit configuration (optional)

<!--
Copy this file to the root of the audited project as `mcp-audit.config.md`.
The audit skill reads it before starting. Every field is optional; delete what you don't need.
It contains no secrets: never put tokens here.
-->

## Project
- **Name:** gitlab-docs-mcp
- **Modules to audit:** `server/` (ignore `examples/`, `build/`)
- **Owner / contact:** team-platform

## Transports
- **Current:** stdio
- **Planned:** Streamable HTTP (stateless preferred), behind corporate gateway
- **HTTP deployment (when it exists):** internal only / via gateway `<name>` / public

## Primitives in use
- resources: yes (templates: yes)
- prompts: yes
- tools: no (planned: maybe search)
- completions: no
- logging notifications: no

## Clients
- **Target MCP clients:** in-house Kotlin client (supports resources + prompts; no elicitation, no sampling)
- **Target protocol:** 2025-11-25 (SDK-driven). Set to `2026-07-28` only if the project deliberately targets it.

## Upstream systems
- GitLab REST API v4 at `$GITLAB_URL`, token in `$GITLAB_TOKEN` (scope: read_api)
- Readable scope: groups `libs/*`, `platform/docs`

## Accepted exceptions
<!-- Rule ID | justification | expiry (YYYY-MM-DD). Expired entries are reported again. -->
| Rule | Justification | Expires |
|---|---|---|
| SEC-07 | stdio, single user; upstream GitLab enforces its own rate limit | 2027-03-31 |
| AUTH-01 | HTTP will sit behind corporate SSO gateway; no OAuth PRM on the server itself | 2027-06-30 |

## Severity overrides
<!-- Rule ID → new severity, with reason -->
| Rule | Severity | Reason |
|---|---|---|
| RES-08 | MEDIUM | GitLab instance is slow; caching matters here |

## Internal conventions (optional, audited as LOW unless stated)
- Logging: slf4j + logback, JSON encoder, stderr
- Config: env vars prefixed `GDOCS_`
- Naming: tool/prompt names in snake_case
