---
name: audit-mcp-server-kotlin
description: Audits an existing Kotlin MCP server (official Kotlin SDK, Gradle Kotlin DSL, stdio and/or Streamable HTTP) against MCP specification best practices, security and SDK currency, using an offline reference snapshot. Produces a read-only report with rule IDs; applies fixes only after the developer approves specific findings. Use when asked to review, audit, check, harden or modernize a Kotlin MCP server.
---

# Audit a Kotlin MCP server

You review an **existing** MCP server written in Kotlin with the official SDK (`io.modelcontextprotocol:kotlin-sdk*`). The output is a findings report. Code changes happen only after explicit approval. The skill works for any server: resources, prompts, tools, completions, logging, stdio or HTTP.

Reference snapshot date: **2026-09-18** (see `references/versions.md`).

## Hard rules (always apply)

1. **Offline.** Never fetch URLs, browse, or query package registries. The files in `references/` are the only source of truth.
   - If something is not covered there, write "Not covered by the reference snapshot". Do not guess.
   - URLs in `references/SOURCES.md` are for maintainers only.
2. **Read-only by default.** Allowed: list, read and search files, including read-only shell commands such as `rg`, `grep`, `find`, `ls`, `cat`, `head` and `wc`. Not allowed:
   - editing or creating files, with one exception: the report file, when the developer asks for it (Step 4);
   - running Gradle, the server or its tests;
   - installing anything;
   - touching git state.
3. **Execution only on explicit request.** Anything beyond read-only inspection (`gradle`, `java`, `python`, the smoke-test scripts, the server itself) runs only if the developer explicitly asks for it in the current conversation. State the exact command before running it, including any prerequisite such as `installDist`, and run only that.
   - Otherwise, give the commands from `references/build-and-run.md` for the developer to run.
4. **Fixes only after approval.** After the report, wait. Apply only the rule IDs the developer approves ("fix STDIO-01, ERR-01", or "all HIGH and above"). Then:
   - keep the diffs minimal and scoped to those findings;
   - do not refactor anything else;
   - never change tool names, prompt names, resource URIs or URI templates without pointing out that this breaks clients;
   - adding or removing a dependency (for example switching the logging backend) needs explicit approval, even inside an approved finding.
5. **Evidence or nothing.** Every finding cites:
   - a rule ID from `references/checklist.md`;
   - `file:line`;
   - a short code excerpt;
   - the reference that justifies it.
   A search hit alone is not evidence: read the surrounding code. For something *missing*, cite where it should be and what you searched (see the checklist header).
6. **Protect secrets.** Never reproduce a secret in full. Show the first 4 characters followed by `…`, and recommend rotation.
7. **Stay in scope.** Audit only the MCP server modules. Mention unrelated issues only when they are security-relevant.

## Reference map (load on demand, not all at once)

| File | Load when |
|---|---|
| `references/checklist.md` | Always. It holds all rules, severities and search patterns |
| `references/versions.md` | Always, first. It sets the baseline and the freshness rules |
| `references/kotlin-sdk.md` | Always. SDK API, pitfalls, deprecations, error behaviour |
| `references/spec-core.md` | Capabilities, errors, pagination, logging, progress, cancellation, completion |
| `references/spec-primitives.md` | The server exposes resources, prompts or tools, or uses elicitation, sampling or roots |
| `references/transport-stdio.md` | stdio entry point exists |
| `references/transport-http.md` | HTTP entry point exists or is planned; auth |
| `references/security.md` | Always (SEC-* rules) |
| `references/spec-2026-07-28-delta.md` | Writing FUT-* findings, or the project targets 2026-07-28 |
| `references/report-template.md` | Writing the report |
| `references/build-and-run.md` | Giving verification instructions, or running on request |
| `assets/mcp-audit.config.example.md` | Suggesting a config file to the developer |

## Workflow

### Step 0: Project configuration

Look for `mcp-audit.config.md` (or `.mcp-audit.md`) at the repository root. If it exists, read it. It declares:

- transports (current and planned);
- primitives, target clients and the target protocol;
- the readable scope of upstream systems;
- accepted exceptions and severity overrides.

If it is missing, continue with defaults. At the end, suggest creating one from `assets/mcp-audit.config.example.md`.

### Step 1: Reconnaissance (build the server profile)

1. **Build files:** `settings.gradle.kts`, `build.gradle.kts`, `gradle/libs.versions.toml`, `gradle.properties`, `buildSrc/`. Extract:
   - SDK, Kotlin, Ktor and JVM toolchain versions;
   - the logging backend;
   - `application` / `shadow` configuration.
2. **Entry points.** Search `StdioServerTransport`, `mcpStreamableHttp`, `mcpStatelessStreamableHttp`, `StreamableHttpServerTransport`, `SseServerTransport`, `\bmcp\s*[({]`, `createSession`, `fun main`.
3. **Server definition.** Search `Server(`, `ServerCapabilities(`, `Implementation(`, `instructions`.
4. **Inventory:**
   - `addResource`, `addResourceTemplate`, `addPrompt`, `addTool`, `setRequestHandler<`;
   - `createElicitation`, `createMessage`, `listRoots`;
   - `sendLoggingMessage`, `sendResourceUpdated`.
   Record names, URIs and URI templates.
5. **Upstream access:** HTTP clients, DB, file-system access, credentials (`getenv`, config files).
6. **Logging config files:** `logback*.xml`, `log4j2*.xml`, `simplelogger.properties`.
7. **Tests:** `src/test`, and use of `kotlin-sdk-testing` or `kotlin-sdk-client`.

Write the profile, as in section 1 of the report template, before auditing.

### Step 2: Baseline and applicability

- Determine the baseline with `versions.md` §3. If the SDK version cannot be resolved statically (BOM, property, convention plugin), mark VER-01 and VER-02 as "not verifiable statically", assume the 2025-11-25 baseline, and give the `dependencies` command from `build-and-run.md` §1. With SDK ≥ 0.10.0 it is **MCP 2025-11-25**; 2026-07-28 is forward-compat (INFO) unless the config says otherwise.
- Apply the snapshot freshness rule (`versions.md` §4).
- Choose the rule categories from the profile:
  - VER, LIFE, ERR, SEC, KT, UTIL, TEST, FUT: always;
  - STDIO: if a stdio entry point exists;
  - HTTP and AUTH: if an HTTP entry point exists. If HTTP is only planned, evaluate HTTP-10 only (readiness notes);
  - RES, PRM, TOOL: according to the inventory.

### Step 3: Audit

Go category by category through `checklist.md`. For each applicable rule:

1. run its search patterns;
2. read the hits in context;
3. decide pass, finding, or not verifiable statically.

Apply the config's exceptions and overrides. Re-flag expired exceptions.

Useful habits:

- Check both directions: declared but unimplemented, and implemented but undeclared.
- Follow the data: take each client-controlled input (URI, template variable, argument) and trace it to file-system, upstream or process calls (SEC-03/04/05).
- Look at error paths as carefully as happy paths (ERR-*).
- For stdio, trace **everything that can reach stdout** (STDIO-01/02).
- Group identical defects into one finding with several locations.

### Step 4: Report

Follow `references/report-template.md` exactly. Write in the developer's language; keep rule IDs and code in English.

- Present the report in the conversation.
- Write it to a file (for example `docs/mcp-audit/<date>.md`) only if the developer asks.
- End by asking which rule IDs to fix.

### Step 5: Fixes (only after approval)

1. Restate the approved IDs and the files you will touch.
2. Apply minimal changes that follow the "Fix" guidance of each rule and the idioms in `kotlin-sdk.md`. Keep the project's code style.
3. Do not bump dependency versions unless the approved finding is about versions (VER-*).
4. Summarize the changes per rule ID, with `file:line`.
5. Give the verification commands (`build-and-run.md` §6). Do not run them unless the developer explicitly asks.

## Severity quick guide

- **CRITICAL:** exploitable (secrets, traversal, token passthrough, no DNS-rebinding protection) or protocol-breaking (stdout pollution).
- **HIGH:** breaks a spec MUST at the baseline, or is a serious robustness risk.
- **MEDIUM:** breaks a SHOULD, uses a deprecated API, or is a robustness or maintainability issue.
- **LOW:** a quality improvement.
- **INFO:** forward-compatibility (2026-07-28) or an observation.

## If the project doesn't fit

- **Not using the official Kotlin SDK** (for example a hand-written JSON-RPC server, or a Java SDK): say so. Audit the spec-level rules (spec-*, transport-*, security), and mark SDK-specific rules (kotlin-sdk.md, KT-*, VER-*) as not applicable.
- **SDK newer than 0.15.0:** audit with the known rules, flag uncertainty in the report header, and recommend refreshing the skill (`SOURCES.md` §2).
- **Client code in the same repository:** out of scope. Audit only the server side.
