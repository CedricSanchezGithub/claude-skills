# Sources and refresh procedure

> **For the auditing agent:** this file is for traceability and maintenance only. **Never fetch these URLs during an audit.** The skill runs offline, and the other reference files are the source of truth.

**Snapshot date:** 2026-09-18
**Spec revisions captured:** 2026-07-28 (latest), 2025-11-25 (baseline)
**Kotlin SDK captured:** 0.15.0 (latest release, tag dated 2026-07-28). Source read at that tag, plus main-branch HEAD `ae44bab` for unreleased notes.

## 1. Sources

### MCP specification: 2026-07-28

| Topic | URL | Used in |
|---|---|---|
| Changelog | https://modelcontextprotocol.io/specification/2026-07-28/changelog.md | spec-2026-07-28-delta.md |
| Deprecated registry | https://modelcontextprotocol.io/specification/2026-07-28/deprecated.md | spec-2026-07-28-delta.md §3 |
| Base protocol | https://modelcontextprotocol.io/specification/2026-07-28/basic/index.md | spec-core.md, delta |
| Versioning | https://modelcontextprotocol.io/specification/2026-07-28/basic/versioning.md | versions.md, delta §4 |
| server/discover | https://modelcontextprotocol.io/specification/2026-07-28/server/discover.md | delta |
| Patterns (MRTR, subscriptions, cancellation, progress) | https://modelcontextprotocol.io/specification/2026-07-28/basic/patterns/ | delta |
| Utilities (caching, completion, logging, pagination) | https://modelcontextprotocol.io/specification/2026-07-28/server/utilities/ | spec-core.md, delta |
| Resources / Prompts / Tools | https://modelcontextprotocol.io/specification/2026-07-28/server/{resources,prompts,tools}.md | spec-primitives.md |
| Elicitation / Sampling / Roots | https://modelcontextprotocol.io/specification/2026-07-28/client/{elicitation,sampling,roots}.md | spec-primitives.md §5 |
| Transports (stdio, Streamable HTTP) | https://modelcontextprotocol.io/specification/2026-07-28/basic/transports/ | transport-*.md |
| Authorization | https://modelcontextprotocol.io/specification/2026-07-28/basic/authorization/ | transport-http.md §4 |
| Security best practices | https://modelcontextprotocol.io/docs/2026-07-28/tutorials/security/security_best_practices.md | transport-http.md §5, security.md |
| Schema | https://raw.githubusercontent.com/modelcontextprotocol/modelcontextprotocol/main/schema/2026-07-28/schema.ts | all spec files |

### MCP specification: 2025-11-25

| Topic | URL |
|---|---|
| Lifecycle | https://modelcontextprotocol.io/specification/2025-11-25/basic/lifecycle.md |
| Changelog | https://modelcontextprotocol.io/specification/2025-11-25/changelog.md |
| Utilities (cancellation, progress, ping) | https://modelcontextprotocol.io/specification/2025-11-25/basic/utilities/ |
| Server utilities (logging, pagination, completion) | https://modelcontextprotocol.io/specification/2025-11-25/server/utilities/ |
| Resources / Prompts / Tools | https://modelcontextprotocol.io/specification/2025-11-25/server/ |
| Transports (single page) | https://modelcontextprotocol.io/specification/2025-11-25/basic/transports.md |
| Authorization (single page) | https://modelcontextprotocol.io/specification/2025-11-25/basic/authorization.md |
| Security best practices | https://modelcontextprotocol.io/docs/2025-11-25/tutorials/security/security_best_practices.md |

### Kotlin SDK

| Item | Location |
|---|---|
| Repository | https://github.com/modelcontextprotocol/kotlin-sdk (tag `0.15.0`) |
| Server API | `kotlin-sdk-server/src/commonMain/kotlin/io/modelcontextprotocol/kotlin/sdk/server/` (`Server.kt`, `StdioServerTransport.kt`, `KtorServer.kt`, `StreamableHttpServerTransport.kt`, `HostValidation.kt`) |
| Types and protocol constants | `kotlin-sdk-core/src/commonMain/kotlin/io/modelcontextprotocol/kotlin/sdk/types/` (`common.kt`: `LATEST_PROTOCOL_VERSION`, `SUPPORTED_PROTOCOL_VERSIONS`) |
| Samples | `samples/weather-stdio-server`, `samples/kotlin-mcp-server`, `samples/simple-streamable-server` |
| Release notes | https://github.com/modelcontextprotocol/kotlin-sdk/releases (no CHANGELOG file in the repo) |

Caveats at capture time:

- Maven Central metadata could not be read from the capture environment. The version list comes from git tags; the samples resolve 0.15.0 from Maven Central.
- WebFetch truncated `schema.md` (2026-07-28). Types were taken from `schema.ts` instead.

## 2. Refresh procedure (someone with internet access, about 1–2 h)

1. **Check for new spec revisions:** open https://modelcontextprotocol.io/llms.txt and look for `/specification/<date>/` entries newer than 2026-07-28.
2. **Check the SDK:** find the latest release tag of `modelcontextprotocol/kotlin-sdk`, and in `types/common.kt` read `LATEST_PROTOCOL_VERSION` and `SUPPORTED_PROTOCOL_VERSIONS`.
3. **If the SDK now supports 2026-07-28 (or newer):**
   - make that revision the baseline in `versions.md` §3;
   - move the relevant `FUT-*` rules into the normal categories with real severities;
   - update `spec-core.md` and `spec-primitives.md` (lifecycle → `server/discover`; `resultType`; caching; `-32602` for not-found; MRTR);
   - update `transport-http.md` (no sessions; `Mcp-Method` / `Mcp-Name` headers).
4. **Update `kotlin-sdk.md`:**
   - the version table in `versions.md` §2;
   - new deprecations in §7 (search the source for `@Deprecated`);
   - behaviour changes in §8 (read the release notes);
   - recheck §4 (error behaviour in `Server.kt`), §3 (registration signatures) and §6 (dispatchers and concurrency).
5. **Update `checklist.md`:** add, remove or re-grade rules. Keep existing IDs stable and never reuse a retired ID; mark retired rules "(retired)" rather than deleting them.
6. **Bump the snapshot date** in `versions.md`, in this file and in `SKILL.md`.
7. **Sanity checks:**
   - no URL appears outside this file: `grep -rnE 'https?://' --include=*.md . | grep -v SOURCES.md`, where the only allowed hits are placeholder or example hosts;
   - every rule ID cited in the other references exists in `checklist.md`.
