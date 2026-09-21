# Audit checklist

Every finding in a report must cite a rule ID from this file. Rules are grouped by category. Each rule states:

- **Applies:** when to evaluate it. Skip the rule and list it under "Not applicable" otherwise.
- **Check:** what to verify by reading code and config.
- **Search:** regexes to run with the agent's code-search tool. They use **ripgrep / PCRE syntax** (`\b`, `\s`, `\d`); with GNU grep use `grep -P`. Patterns starting with `-` must be passed with `rg -e '…'`. These are hints, not proof: confirm each hit by reading the surrounding code.
- **Ref:** the reference file and section behind the rule.
- **Fix:** the typical remediation. Adapt it to the project; never apply it without approval.

## Severity scale

| Severity | Meaning |
|---|---|
| **CRITICAL** | Exploitable security issue, or a protocol-breaking defect (the server cannot work reliably with clients) |
| **HIGH** | Violates a spec **MUST** at the audit baseline, or a serious security or robustness risk |
| **MEDIUM** | Violates a spec **SHOULD**, uses a deprecated API, or carries a notable robustness or maintainability risk |
| **LOW** | Best-practice or quality improvement |
| **INFO** | Forward-compatibility (2026-07-28) or an observation with no action required now |

**Evidence for absent things.** When a finding is about something *missing* (no tests, no auth, no rate limit, no `instructions`), cite where it should be and what you searched: for example "absent — searched `authenticate\(` and `install\(Authentication\)` in `server/src/main`; routes defined in `Http.kt:40`".

**Accepted exceptions** from the config file are not reported as findings. They are listed in report §8. Once an exception has expired, the rule is reported again at its normal severity.

**Primary rule for overlaps.** Report each defect once, under the rule that owns it:
- secrets in logs or MCP log notifications → SEC-02 (UTIL-01 covers only capability, rate and logger naming);
- missing required prompt arguments → ERR-04 (PRM-01 covers declaration and descriptions);
- stdout written by a logging backend → STDIO-02 (STDIO-01 covers code and other sources).

**When the project targets 2026-07-28** (config or code), FUT-* rules become normal findings: grade them as for any rule (MUST → HIGH, SHOULD → MEDIUM), using spec-2026-07-28-delta.md as the reference.

Severity may be raised or lowered by one level with a written reason, for example "stdio-only, single user → rate limiting LOW". Config-file overrides apply as well.

---

## VER: Versions and dependencies

### VER-01 · HIGH · SDK supports the current baseline protocol
- **Applies:** always.
- **Check:** find the `io.modelcontextprotocol:kotlin-sdk*` version. Versions below 0.10.0 cannot negotiate `2025-11-25`, and versions below 0.8.0 lack structured tool output and elicitation.
- **Search:** `io\.modelcontextprotocol`, `kotlin-sdk`, `mcp-kotlin` in `*.gradle.kts`, `libs.versions.toml`, `gradle.properties`.
- **Ref:** versions.md §2
- **Fix:** upgrade to the latest version known to the snapshot (0.15.0), and review the breaking changes in kotlin-sdk.md §8.

### VER-02 · MEDIUM · SDK is behind the snapshot's latest release
- **Applies:** SDK ≥ 0.10.0 and < 0.15.0.
- **Check:** list the behaviour changes between the project version and 0.15.0 (kotlin-sdk.md §8) that matter to this project. Security-relevant ones: the DNS-rebinding default in 0.13.0; concurrent handlers in 0.15.0.
- **Ref:** kotlin-sdk.md §8
- **Fix:** plan the upgrade, with the migration notes listed in the report.

### VER-03 · MEDIUM · Deprecated SDK APIs in use
- **Applies:** always.
- **Check:** compare the code against the deprecation table. APIs already removed in newer versions block upgrades: raise to HIGH if an upgrade is planned.
- **Search:** `server\.connect\(`, `McpError`, `Tool\.(Input|Output)`, `import io\.modelcontextprotocol\.kotlin\.sdk\.[A-Z]`, `JSONRPCError`, `\bErrorCode\.` (without `RPCError.`), `Role\.(user|assistant)\b`, `LoggingLevel\.[a-z]`, `PromptMessageContent`, `EmptyRequestResult`, `CreateElicitation(Request|Result)`, `\bResourceReference\b`, `ElicitRequestParams\(`, `Application\.MCP`, `Routing\.mcp`, `mcpWebSocketTransport`, `mcpWebSocket\(\s*options`, `StreamableHttpServerTransport\(\s*(enableJsonResponse|\))`, `mcpStatelessStreamableHttp\([^)]*eventStore`, `LegacyTitledEnumSchema`, `RequestHandlerExtra\(`. For `StdioServerTransport(` read every call: named `inputStream =`/`outputStream =` **or positional arguments** (often split over several lines) mean the deprecated constructor.
- **Ref:** kotlin-sdk.md §7
- **Fix:** apply the listed replacement.

### VER-04 · LOW · Dependency versions pinned and aligned
- **Applies:** always.
- **Check:**
  - no dynamic versions (`+`, `latest.release`);
  - Ktor modules aligned through the BOM or a single version;
  - an HTTP engine explicitly declared when HTTP is used (engines are not transitive).
- **Search:** `\+"`, `latest\.release`, `latest\.integration`, `ktor-server-(cio|netty|jetty)`.
- **Ref:** kotlin-sdk.md §1, security.md §8
- **Fix:** use a version catalog and `platform(libs.ktor.bom)`.

---

## LIFE: Capabilities and server identity

### LIFE-01 · MEDIUM · Declared capabilities match what is implemented
- **Applies:** always.
- **Check:** every declared capability (`tools`, `resources`, `prompts`, `logging`, `completions`) has at least one registration or handler. The SDK already throws if something is registered without its capability. Look for the reverse: a capability declared with nothing behind it, for example `completions` without a `CompleteRequest` handler. **Exception the SDK does not catch:** a `setRequestHandler<CompleteRequest>` without `completions` declared. That breaks a spec MUST, so report it as HIGH.
- **Search:** `ServerCapabilities\(`, `addTool`, `addResource`, `addResourceTemplate`, `addPrompt`, `setRequestHandler<CompleteRequest>`, `sendLoggingMessage`.
- **Ref:** spec-core.md §3
- **Fix:** remove the unused capability or implement it.

### LIFE-02 · MEDIUM · `listChanged` and `subscribe` are truthful
- **Applies:** a `listChanged` or `subscribe` flag is true.
- **Check:**
  - `listChanged = true` is only useful if the set can change at runtime, through `add*` / `remove*` after startup;
  - `resources.subscribe = true` requires the server to call `sendResourceUpdated(...)` when the underlying data changes.

  A declared-but-never-fired `subscribe` misleads clients: clients subscribe and never hear back.
- **Search:** `listChanged\s*=\s*true`, `subscribe\s*=\s*true`, `sendResourceUpdated`, `remove(Tool|Resource|Prompt)`.
- **Ref:** spec-primitives.md §2.1, §2.7
- **Fix:** set the flags to false or omit them, or implement change detection and notifications.

### LIFE-03 · LOW · Server identity and instructions
- **Applies:** always.
- **Check:**
  - `Implementation.name` is stable and meaningful;
  - `version` comes from the build (not a hardcoded `"1.0.0"` that never changes);
  - `title` is set for display;
  - `instructions` briefly tell the model what the server offers and how to use it, when that is non-obvious.
- **Search:** `Implementation\(`, `instructions`.
- **Ref:** spec-core.md §1
- **Fix:** inject the version from Gradle (for example a generated `BuildConfig`, or reading `Implementation-Version` from the manifest), and add a short `instructions` string.

### LIFE-04 · MEDIUM · Client capabilities checked before using client features
- **Applies:** code calls `createElicitation`, `createMessage` (sampling) or `listRoots`.
- **Check:** the call is guarded by the client's declared capability (`session.clientCapabilities?.elicitation` / `.sampling` / `.roots`, including the specific elicitation mode and `sampling.tools`). The spec says MUST NOT send undeclared features. With `enforceStrictCapabilities = true` (the default), SDK 0.15.0 throws instead of sending, so an unguarded call is a **robustness** defect: it fails at runtime, and the exception text leaks through the tool result. **Raise to HIGH if `enforceStrictCapabilities = false`**, because the SDK then sends undeclared requests.
- **Search:** `createElicitation`, `createMessage`, `listRoots`, `clientCapabilities`, `enforceStrictCapabilities`.
- **Ref:** spec-core.md §3, spec-primitives.md §5
- **Fix:** add a guard with a graceful fallback (a tool execution error explaining what is missing).

---

## ERR: Errors

### ERR-01 · MEDIUM · Missing resources raise an error, never empty contents
- **Applies:** resources or resource templates. A SHOULD at the 2025-11-25 baseline; HIGH when the project targets 2026-07-28, where it is a MUST.
- **Check:** template and dynamic read handlers throw `McpException(RPCError.ErrorCode.RESOURCE_NOT_FOUND, "Resource not found", data = {uri})` when the target does not exist. They must not return `ReadResourceResult(contents = emptyList())`, and they must not return a text such as "not found" as if it were content.
- **Search:** `ReadResourceResult\(`, `emptyList\(\)`, `contents\s*=\s*listOf\(\)`, `RESOURCE_NOT_FOUND`, `"not found"`.
- **Ref:** spec-primitives.md §2.5, spec-core.md §5
- **Fix:** throw `McpException` with the not-found code, centralized in one helper (see FUT-03).

### ERR-02 · MEDIUM · Tool failures are tool execution errors with actionable text
- **Applies:** tools.
- **Check:**
  - input validation failures, upstream failures and business errors return `CallToolResult(isError = true, …)` with a message the model can act on;
  - early-return branches for missing arguments set `isError = true`. The official sample forgets this;
  - handlers do not rely on letting exceptions escape (see ERR-03).
- **Search:** `CallToolResult\(`, `return@addTool`, `isError`.
- **Ref:** spec-primitives.md §4.5
- **Fix:** return explicit `isError = true` results with clear, safe messages.

### ERR-03 · HIGH · Exception messages do not leak internal details
- **Applies:** always.
- **Check:** the SDK forwards `e.message` to the client for any exception escaping a tool handler, and for non-`McpException`s from other handlers (kotlin-sdk.md §4). Look for handlers that do not catch upstream or IO exceptions whose messages can contain:
  - URLs with tokens;
  - internal hostnames;
  - SQL;
  - file paths;
  - stack details.
- **Search:** handler bodies without `try`, `throw IllegalStateException\(`, `error\(`, `require\(`, `check\(` with interpolated internals, `\$\{e\.message\}`, `e\.toString\(\)`, `stackTraceToString`.
- **Ref:** security.md §3, kotlin-sdk.md §4
- **Fix:** catch at the handler boundary, log the details to stderr or the log, and return or throw a curated message. Always rethrow `CancellationException`.

### ERR-04 · MEDIUM · Prompt errors use -32602
- **Applies:** prompts with arguments.
- **Check:** a missing required argument, or an invalid argument value, throws `McpException(RPCError.ErrorCode.INVALID_PARAMS, …)`. The SDK does not enforce `required = true`. Unknown prompt names: SDK 0.15.0 returns -32603; report this as INFO (SDK limitation).
- **Search:** `addPrompt`, `required\s*=\s*true`, `request\.arguments`.
- **Ref:** spec-primitives.md §3
- **Fix:** add argument validation at the start of each prompt provider.

### ERR-05 · LOW · Custom error codes respect reserved ranges
- **Applies:** code creates `McpException` with literal numeric codes.
- **Check:** do not invent codes in -32020..-32099 (spec-reserved in 2026). Avoid new codes in -32000..-32019. Prefer the standard codes; allocate application codes outside -32768..-32000.
- **Search:** `McpException\(\s*-?\d+`, `code\s*=\s*-3\d{4}`.
- **Ref:** spec-core.md §5
- **Fix:** use `RPCError.ErrorCode` constants or codes outside the reserved range.

---

## STDIO: stdio transport

### STDIO-01 · CRITICAL · Nothing but MCP messages on stdout
- **Applies:** stdio transport.
- **Check:** no code path writes to stdout. That includes startup, shutdown, error handling, libraries and child processes. See the full list in transport-stdio.md §2.
- **Search:**
  - `\bprintln\(`, `\bprint\(`, `System\.out`, `System\.\`out\``. Exclude the single use passed to `StdioServerTransport`;
  - `inheritIO\(\)`, `Redirect\.INHERIT`;
  - `prettyPrint\s*=\s*true`;
  - `embeddedServer\(` in stdio mode.
- **Ref:** transport-stdio.md §1–2
- **Fix:** route everything to a logger configured for stderr, or to `System.err`.

### STDIO-02 · CRITICAL · Logging backend writes to stderr
- **Applies:** stdio transport.
- **Check:** identify the logging backend on the runtime classpath. Then:
  - Logback: every `ConsoleAppender` has `<target>System.err</target>`, because the default is System.out;
  - Log4j2: every `Console` appender has `target="SYSTEM_ERR"`;
  - slf4j-simple: `logFile` is not `System.out`;
  - no `logback-test.xml` leaks into the main classpath;
  - **a backend on the classpath without any config file** (Logback or Log4j2) logs to stdout by default.

  CRITICAL when a console output really targets stdout. Downgrade to LOW (a hardening note) when the config is correct but fragile.
- **Search:** files `logback*.xml`, `log4j2*.xml`, `simplelogger.properties`; patterns `ConsoleAppender`, `<Console`, `SYSTEM_OUT`, `System.out`.
- **Ref:** transport-stdio.md §2
- **Fix:** set the stderr target, or switch to slf4j-simple (stderr by default).

### STDIO-03 · HIGH · Process exits when the client disconnects
- **Applies:** stdio transport.
- **Check:**
  - the main waits on `session.onClose { done.complete() }`, not on `server.onClose` (which does not fire on stdin EOF);
  - there is no `while(true)`, `Thread.sleep(Long.MAX_VALUE)` or bare `awaitCancellation()` keep-alive;
  - resources (HTTP clients, caches, thread pools) are closed on exit;
  - non-daemon threads do not keep the JVM alive.
- **Search:** `onClose`, `Job\(\)`, `done\.join`, `awaitCancellation`, `Thread\.sleep`, `while\s*\(\s*true`, `Executors\.new`, `System\.exit`.
- **Ref:** transport-stdio.md §3, spec-core.md §1
- **Fix:** use the `session.onClose` pattern, and close resources in a `finally` block or `onClose`.

### STDIO-04 · MEDIUM · Current `StdioServerTransport` API
- **Applies:** stdio transport and SDK ≥ 0.13.0.
- **Check:** the constructor is called with named `input =` / `output =`, not `inputStream =` / `outputStream =` or two positional arguments (both resolve to the deprecated constructor).
- **Search:** `StdioServerTransport\(`.
- **Ref:** transport-stdio.md §3
- **Fix:** `StdioServerTransport(input = System.\`in\`.asSource().buffered(), output = System.out.asSink().buffered())`.

### STDIO-05 · MEDIUM · Credentials from the environment, not arguments or files in the repo
- **Applies:** stdio transport with upstream credentials.
- **Check:**
  - tokens are read from env vars, for example `System.getenv("GITLAB_TOKEN")`;
  - they are not taken from CLI args or committed config;
  - a missing variable fails fast with a clear stderr message;
  - there is no OAuth flow on stdio (the spec says SHOULD NOT).
- **Search:** `getenv`, `args\[`, `token`, `application\.(conf|properties|ya?ml)`.
- **Ref:** transport-stdio.md §4, security.md §4
- **Fix:** read from env, validate at startup, and document the variable in the README and the client config example.

### STDIO-06 · LOW · Launch command documented and not via Gradle `run`
- **Applies:** stdio transport.
- **Check:** the README or docs show how an MCP client launches the server: a fat jar (`shadow`), or `installDist` with `build/install/<app>/bin/<app>`. It must not be `./gradlew run`, whose own output pollutes stdout and is slow to start.
- **Search:** README, `shadow`, `application \{`, `installDist`.
- **Ref:** transport-stdio.md §2
- **Fix:** add a `shadowJar` or `installDist` based launch example with the required env vars.

---

## HTTP: Streamable HTTP (evaluate HTTP-01..09 when HTTP exists; if HTTP is only planned, evaluate HTTP-10 only, which covers HTTP-09 as a readiness note)

### HTTP-01 · CRITICAL · DNS-rebinding and Origin protection active
- **Applies:** HTTP transport.
- **Check:**
  - `mcpStreamableHttp` / `mcpStatelessStreamableHttp` / `mcp` are not called with `enableDnsRebindingProtection = false`;
  - on SDK 0.8.4–0.12.x, it is explicitly set to `true`;
  - manual `StreamableHttpServerTransport` routes `install(DnsRebindingProtection)` **with `allowedOrigins` set**. In the plugin it defaults to `null`, which disables Origin validation;
  - when deployed on a hostname, `allowedHosts` **and** `allowedOrigins` are set (custom hosts with null origins skip origin checks).
- **Search:** `enableDnsRebindingProtection`, `DnsRebindingProtection`, `allowedHosts`, `allowedOrigins`, `StreamableHttpServerTransport\(`.
- **Ref:** transport-http.md §1, §3
- **Fix:** keep the protection on, and configure explicit hosts and origins for deployment.

### HTTP-02 · HIGH · Bind address
- **Applies:** HTTP transport.
- **Check:** local and dev runs bind `127.0.0.1`. `0.0.0.0` or no host is acceptable only when fronted by authentication and deployed intentionally; the config file should say so.
- **Search:** `embeddedServer\(`, `host\s*=`, `0\.0\.0\.0`.
- **Ref:** transport-http.md §1
- **Fix:** `host = "127.0.0.1"` by default, and make the bind address configurable.

### HTTP-03 · HIGH · Authentication on MCP routes
- **Applies:** HTTP transport reachable by anything other than the local user.
- **Check:** MCP routes sit inside `authenticate(...) { }` or behind a verified gateway (see AUTH-*). Unauthenticated remote MCP endpoints are a finding.
- **Search:** `install\(Authentication\)`, `authenticate\(`, `bearer\(`, `jwt\(`.
- **Ref:** transport-http.md §3–4
- **Fix:** add Ktor authentication, or document and verify the gateway.

### HTTP-04 · HIGH · CORS restricted
- **Applies:** HTTP transport with `install(CORS)`.
- **Check:** no `anyHost()` in deployed configurations; allowed origins are explicit. The `Mcp-Session-Id` and `Mcp-Protocol-Version` headers are allowed and exposed only if browser clients need them.
- **Search:** `install\(CORS\)`, `anyHost\(\)`, `allowHost\(`.
- **Ref:** transport-http.md §3
- **Fix:** replace `anyHost()` with an explicit allowlist from config.

### HTTP-05 · MEDIUM · Streamable HTTP, not the legacy SSE transport
- **Applies:** HTTP transport.
- **Check:** the project does not use `Route.mcp { }` / `Application.mcp { }` / `SseServerTransport` for new endpoints. That transport has been deprecated since protocol 2025-03-26.
- **Search:** `\bmcp\s*[({]`, `SseServerTransport`, `sse\(`.
- **Ref:** transport-http.md §2–3
- **Fix:** migrate to `mcpStreamableHttp` or `mcpStatelessStreamableHttp`.

### HTTP-06 · MEDIUM · Sessions are secure and not used as authentication
- **Applies:** stateful Streamable HTTP.
- **Check:**
  - a custom `setSessionIdGenerator` stays cryptographically random: `Uuid.random()` or `UUID.randomUUID()`, not counters or timestamps;
  - per-session state is keyed by the authenticated user and the session;
  - `sessionId` is never treated as proof of identity.
- **Search:** `setSessionIdGenerator`, `sessionId`, `Random\(`, `AtomicInteger`, `currentTimeMillis`.
- **Ref:** transport-http.md §2, §5
- **Fix:** use a secure generator, key state as `<userId>:<sessionId>`, and authenticate every request.

### HTTP-07 · MEDIUM · HTTPS in deployment
- **Applies:** HTTP transport deployed beyond localhost.
- **Check:** TLS is terminated by the server or a trusted reverse proxy, and the deployment docs say which.
- **Ref:** transport-http.md §4 (transport security), §6
- **Fix:** document or configure TLS termination.

### HTTP-08 · LOW · Request limits and stream settings
- **Applies:** HTTP transport.
- **Check:**
  - `maxRequestBodySize` is appropriate;
  - engine timeouts are set;
  - behind nginx-like proxies, SSE responses disable buffering (`X-Accel-Buffering: no`);
  - long SSE streams use heartbeats (`sseHeartbeatConfig`, ≥0.15.0).
- **Search:** `maxRequestBodySize`, `sseHeartbeatConfig`, `X-Accel-Buffering`.
- **Ref:** transport-http.md §2–3
- **Fix:** configure the limits and add the header in the proxy or the route.

### HTTP-09 · LOW · Stateless mode considered
- **Applies:** HTTP transport, or HTTP planned.
- **Check:** if the server never sends server-to-client requests or notifications (no elicitation, sampling, roots, list_changed or resource updates), `mcpStatelessStreamableHttp` is simpler and closer to 2026-07-28.
- **Ref:** transport-http.md §6, spec-2026-07-28-delta.md §5
- **Fix:** use the stateless helper.

### HTTP-10 · INFO · HTTP readiness (stdio-only projects)
- **Applies:** stdio only, HTTP planned (config file or developer statement).
- **Check:** walk through the checklist in transport-http.md §6 and report the blockers.
- **Ref:** transport-http.md §6, transport-stdio.md §5

---

## AUTH: Authorization (HTTP)

### AUTH-01 · HIGH · Protected Resource Metadata and 401 challenges
- **Applies:** HTTP with OAuth-style bearer tokens.
- **Check:**
  - `/.well-known/oauth-protected-resource` (or the path-suffixed variant) is served with `authorization_servers`;
  - 401 responses include `WWW-Authenticate: Bearer resource_metadata="…"`.
  - If the project uses a gateway or API key instead, report this as MEDIUM (a deviation from the OAuth profile), or list it under accepted exceptions if the config file says so.
- **Search:** `oauth-protected-resource`, `WWW-Authenticate`, `resource_metadata`, `authorization_servers`.
- **Ref:** transport-http.md §4
- **Fix:** add a PRM route and challenge headers.

### AUTH-02 · CRITICAL · Token audience validated
- **Applies:** HTTP with bearer tokens.
- **Check:** JWT validation checks `aud` (or token introspection checks the resource) against the server's canonical URI, not only the signature and expiry. Tokens issued for other resources are rejected.
- **Search:** `withAudience`, `aud`, `verifier\(`, `validate \{`, `introspect`.
- **Ref:** transport-http.md §4
- **Fix:** add the audience check (`JWT.require(alg).withAudience(canonicalUri)`).

### AUTH-03 · CRITICAL · No token passthrough to upstream APIs
- **Applies:** HTTP server that calls upstream APIs.
- **Check:** the client's `Authorization` header or token is never forwarded to upstream services (GitLab, etc.). Upstream calls use their own credential.
- **Search:** `Authorization`, `bearerAuth\(`, `request\.headers\[`, `call\.request\.authorization\(\)`, `header\(HttpHeaders\.Authorization`.
- **Ref:** transport-http.md §4–5, security.md §4
- **Fix:** use a server-side upstream credential (service token), or a token obtained for the upstream through its own OAuth flow.

### AUTH-04 · HIGH · Correct status codes and scope challenges
- **Applies:** HTTP with auth.
- **Check:**
  - 401 for a missing, invalid or expired token;
  - 403 with `error="insufficient_scope"` and `scope="…"` for insufficient permissions;
  - 400 for malformed auth requests.
- **Search:** `HttpStatusCode\.(Unauthorized|Forbidden)`, `respond\(HttpStatusCode`.
- **Ref:** transport-http.md §4
- **Fix:** align the responses and headers.

### AUTH-05 · HIGH · Tokens never in URLs or logs
- **Applies:** any auth.
- **Check:** tokens are not accepted from query parameters, not logged, and not included in error messages.
- **Search:** `queryParameters\[".*token`, `parameters\["access_token`, `log.*(token|Authorization)`.
- **Ref:** transport-http.md §4, security.md §4
- **Fix:** accept the header only, and redact tokens in logs.

### AUTH-06 · MEDIUM · Minimal scopes
- **Applies:** OAuth scopes defined.
- **Check:**
  - `scopes_supported` is minimal;
  - there are no wildcard or omnibus scopes;
  - (2026-only, report as FUT-style INFO unless targeting 2026) `offline_access` not advertised; scope hierarchy honoured.
- **Ref:** transport-http.md §4–5
- **Fix:** trim the scopes, and use step-up challenges.

### AUTH-07 · HIGH · Per-user authorization on every request
- **Applies:** multi-user HTTP.
- **Check:** list **and** read operations filter by the caller's permissions. Reading by a guessed URI is also checked. There is no reliance on session possession.
- **Search:** read handlers; `principal`, `call\.principal`, `UserIdPrincipal`, `JWTPrincipal`.
- **Ref:** security.md §5, transport-http.md §5
- **Fix:** add authorization checks in list and read handlers.

---

## SEC: General security

### SEC-01 · CRITICAL · No hardcoded secrets
- **Applies:** always.
- **Check:** scan the sources, resources, tests, build files, `.env*` files and docs.
- **Search:** patterns from security.md §4.
- **Ref:** security.md §4
- **Fix:** move secrets to the environment or a secret store, and **rotate any secret that was committed**. Mask secrets in the report.

### SEC-02 · HIGH · No secrets or PII in logs and MCP log notifications
- **Applies:** always.
- **Check:**
  - no logging of tokens, headers or full requests;
  - `data class` configs holding secrets override `toString`;
  - `sendLoggingMessage` payloads contain no secrets, PII or internal details (spec MUST NOT).
- **Search:** `(?i)\blog\w*\.(trace|debug|info|warn|error).*\$\{?\w*(token|secret|password|authorization|header)`, `(?i)data class .*(token|secret|password)`, `sendLoggingMessage`.
- **Ref:** security.md §4, spec-core.md §7
- **Fix:** redact, and override `toString()`.

### SEC-03 · CRITICAL · Path traversal and path injection
- **Applies:** resource URIs, template variables or tool and prompt arguments map to file paths or upstream API paths.
- **Check:**
  - local file access uses `normalize()` plus a `startsWith(root)` containment check after decoding;
  - upstream API paths URL-encode their segments and validate identifiers.
- **Search:** `Path\.of\(`, `Paths\.get\(`, `File\(`, `resolve\(`, `"\$\{.*\}/`, `repository/files/`, `/raw`, `encodeURLPath`, `URLEncoder`.
- **Ref:** security.md §2.1, spec-primitives.md §2.6
- **Fix:** add a containment check, encoding and allow-pattern validation. Return not-found or invalid-params on rejection.

### SEC-04 · HIGH · SSRF controls on outbound requests
- **Applies:** the server fetches URLs influenced by client input.
- **Check:** outbound hosts are restricted to the configured upstream(s); redirects to other hosts are blocked; timeouts are set.
- **Search:** `HttpClient`, `\.get\(`, `URL\(`, `followRedirects`.
- **Ref:** security.md §2.3
- **Fix:** add a host allowlist from config, disable cross-host redirects, and set timeouts.

### SEC-05 · HIGH · Input validation beyond the schema
- **Applies:** tools, prompts, templates, completion.
- **Check:** handlers validate type, length, format and enumerations. Nothing is passed to shells or string-built queries.
- **Search:** `ProcessBuilder`, `Runtime\.getRuntime\(\)\.exec`, `"sh"`, `"-c"`, `createQuery\(".*\$`, `executeQuery\(".*\$`.
- **Ref:** security.md §2
- **Fix:** add validation helpers; use argument arrays and parameterized queries.

### SEC-06 · MEDIUM · Untrusted upstream content and prompt injection
- **Applies:** the server returns third-party or user-authored content (docs, READMEs, issues).
- **Check:**
  - content is returned as resource data or embedded resources;
  - prompt templates delimit untrusted content and label its origin;
  - control characters are stripped and the size is bounded;
  - there is no server-side execution of instructions found in content.
- **Ref:** security.md §3
- **Fix:** add delimiters and labels in prompts, a size cap and sanitization.

### SEC-07 · HIGH · Rate limiting and work bounds
- **Applies:** tools or completion (spec MUST); resources (practice).
- **Check:** there is a limit on invocations or concurrent upstream calls, a bound on result sizes and page sizes, and upstream timeouts. HIGH for tools or completion (spec MUST), especially over HTTP. For stdio single-user setups it may be lowered to MEDIUM or LOW with a stated reason. For resources only, MEDIUM (practice).
- **Search:** `Semaphore`, `RateLimit`, `HttpTimeout`, `requestTimeoutMillis`, `take\(`, `limit`.
- **Ref:** security.md §6, spec-primitives.md §4.6
- **Fix:** add a `Semaphore` or rate limiter, size caps, and `HttpTimeout`.

### SEC-08 · MEDIUM · Least-privilege upstream credentials
- **Applies:** upstream credentials.
- **Check:** the documented or required token scopes are minimal (for GitLab reading, `read_api` / `read_repository`). There are no admin tokens and no write scopes for read-only servers.
- **Search:** README, config docs, `scopes`, `PRIVATE-TOKEN`.
- **Ref:** security.md §4
- **Fix:** document and require the minimal scopes.

### SEC-09 · HIGH · TLS verification not disabled
- **Applies:** outbound HTTPS.
- **Search:** `X509TrustManager`, `checkServerTrusted`, `hostnameVerifier`, `trustAll`, `InsecureTrustManagerFactory`, `sslContext\.init\(null`.
- **Ref:** security.md §7
- **Fix:** use a truststore containing the internal CA.

### SEC-10 · MEDIUM · Readable scope explicitly restricted
- **Applies:** the server proxies a large internal system (GitLab groups and projects, a file share).
- **Check:** an allowlist or config defines what is exposed; the server does not expose "everything the token can read".
- **Ref:** security.md §2.1, §5
- **Fix:** add a configurable allowlist of groups, projects or paths.

---

## RES: Resources

### RES-01 · HIGH · Valid URIs with appropriate schemes
- **Applies:** resources.
- **Check:**
  - URIs and URI templates are RFC 3986 / RFC 6570 compliant;
  - they use a custom scheme (for example `gitlab://group/project/path`) or `file://` / `git://` as appropriate;
  - `https://` is used only when the client can fetch the resource directly (spec SHOULD);
  - there are no spaces or unencoded characters in URIs.
- **Search:** `addResource\(`, `addResourceTemplate\(`, `uri\s*=`, `uriTemplate`.
- **Ref:** spec-primitives.md §2.4
- **Fix:** define a URI scheme and encoding rules, and document them.

### RES-02 · MEDIUM · `mimeType` set explicitly and correctly
- **Applies:** resources.
- **Check:**
  - every `addResource` passes `mimeType`. The SDK default is `"text/html"`;
  - `TextResourceContents` / `BlobResourceContents` carry the right type (`text/markdown`, `text/plain`, `application/json`, `image/png`…);
  - a template's `mimeType` is set only if all matches share it.
- **Search:** `addResource\(` (check for a `mimeType` argument), `TextResourceContents\(`, `BlobResourceContents\(`.
- **Ref:** spec-primitives.md §2.3, §2.7
- **Fix:** pass the right `mimeType`, derived from the file extension where dynamic.

### RES-03 · MEDIUM · Text vs binary encoding
- **Applies:** resources that may be binary (images, PDFs, archives).
- **Check:** binary content uses `BlobResourceContents` with base64 (spec MUST "properly encoded"). Text is used only when the content is really text; there is no lossy UTF-8 decoding of binary.
- **Search:** `BlobResourceContents`, `Base64`, `readText\(`, `String\(bytes`.
- **Ref:** spec-primitives.md §2.3, §2.6
- **Fix:** detect binary and use a base64 blob.

### RES-04 · MEDIUM · Useful metadata for hosts and models
- **Applies:** resources and templates.
- **Check:**
  - `name` is set, with a `title` when useful;
  - `description` explains what the resource contains and when to use it;
  - `size` is set when known (it helps hosts estimate context usage);
  - annotations are valid: `audience` only `user` / `assistant`, `priority` in 0..1, ISO 8601 `lastModified`.
- **Ref:** spec-primitives.md §1, §2.3
- **Fix:** enrich the metadata.

### RES-05 · MEDIUM · Scales with the number of resources
- **Applies:** resources, when the set can be large (for example many repositories or files).
- **Check:**
  - large or open-ended sets are exposed through resource templates rather than enumerating everything at startup;
  - if `resources/list` can exceed a few hundred items, it is paginated through a custom `setRequestHandler<ListResourcesRequest>`;
  - cursors are opaque, and an invalid cursor gives -32602;
  - the end of the list omits `nextCursor` (never `""`).
- **Search:** `addResource\(` inside loops, `ListResourcesRequest`, `nextCursor`.
- **Ref:** spec-core.md §6, spec-primitives.md §2.3
- **Fix:** switch to templates, and paginate.

### RES-06 · MEDIUM · Template handlers validate variables and echo URIs
- **Applies:** resource templates.
- **Check:**
  - variables from the `Map<String,String>` are validated (see SEC-03);
  - returned contents use the requested URI (or a sub-resource URI);
  - not-found cases throw (see ERR-01).
- **Search:** `addResourceTemplate`.
- **Ref:** spec-primitives.md §2.3, §2.7
- **Fix:** add validation and consistent URIs.

### RES-07 · MEDIUM · Content size bounded
- **Applies:** resources returning upstream files.
- **Check:** content is capped or split. Kotlin-SDK clients cannot read messages above 16 MiB, and model context is far smaller. There is a clear truncation notice, or a pagination scheme via sub-resources.
- **Ref:** security.md §6, transport-stdio.md §3
- **Fix:** add a size cap with a notice, or split into sections.

### RES-08 · LOW · Upstream caching
- **Applies:** resources backed by remote systems.
- **Check:** repeated reads are cached with a TTL; the cache is thread-safe; invalidation is reasonable.
- **Ref:** security.md §6
- **Fix:** add a TTL cache, for example Caffeine or a `ConcurrentHashMap` with timestamps.

---

## PRM: Prompts

### PRM-01 · MEDIUM · Arguments declared and validated
- **Applies:** prompts.
- **Check:**
  - every argument used in the provider is declared in `arguments` with a `description` and a correct `required` flag;
  - required arguments are checked (see ERR-04);
  - values are parsed and validated. Values are always strings.
- **Search:** `addPrompt\(`, `PromptArgument\(`, `request\.arguments`.
- **Ref:** spec-primitives.md §3
- **Fix:** declare, validate and document the arguments.

### PRM-02 · MEDIUM · Safe templating of arguments and content
- **Applies:** prompts that interpolate arguments or upstream content into message text.
- **Check:** user and upstream text is delimited (fenced or quoted), labelled and length-capped. Large documents are attached as embedded resources or `resource_link`s rather than pasted.
- **Ref:** security.md §3, spec-primitives.md §3
- **Fix:** use delimiters and labels, and switch to embedded resources.

### PRM-03 · LOW · Prompt metadata quality
- **Applies:** prompts.
- **Check:** stable, unique `name`s (same character rules as tools recommended); `title` and `description` explain the purpose to the user who selects the prompt.
- **Ref:** spec-primitives.md §1, §3
- **Fix:** improve the names and descriptions.

### PRM-04 · MEDIUM · Well-formed messages
- **Applies:** prompts.
- **Check:**
  - `role` is `User` or `Assistant`, used deliberately;
  - embedded resources include a URI, a `mimeType` and either text or blob (spec MUST);
  - image and audio content is base64 with a MIME type (MUST).
- **Search:** `PromptMessage\(`, `EmbeddedResource`, `ImageContent`.
- **Ref:** spec-primitives.md §1.4, §3
- **Fix:** complete the content blocks.

### PRM-05 · LOW · Argument completion
- **Applies:** prompts or templates with enumerable arguments (project names, versions).
- **Check:** consider the `completions` capability with a `CompleteRequest` handler: at most 100 values, validated input, and no disclosure of items the caller cannot access.
- **Ref:** spec-core.md §11
- **Fix:** add a completion handler.

---

## TOOL: Tools

### TOOL-01 · MEDIUM · Tool names follow the spec pattern
- **Applies:** tools.
- **Check:** each name matches `^[A-Za-z0-9_.-]{1,128}$`, and names are unique and stable.
- **Search:** `addTool\(`, `name\s*=\s*"`.
- **Ref:** spec-primitives.md §4.2
- **Fix:** rename (this is a breaking change for clients; note it).

### TOOL-02 · MEDIUM · Input schemas are complete
- **Applies:** tools.
- **Check:**
  - the schema is present, with root type object (the SDK's `ToolSchema` does this);
  - every property has `type` and `description`;
  - the `required` list is correct;
  - enums or patterns constrain values where possible;
  - no-arg tools use `additionalProperties = false` (recommended);
  - `ToolSchema(...)` is called with **named arguments**. A positional first argument binds to `$schema`, not `properties`: report that as HIGH, because the schema is then wrong.
- **Search:** `ToolSchema\(`, `putJsonObject`, `required\s*=`.
- **Ref:** spec-primitives.md §4.1
- **Fix:** complete the schema.

### TOOL-03 · MEDIUM · Accurate tool annotations
- **Applies:** tools.
- **Check:** `readOnlyHint = true` for read-only tools. Destructive and idempotent hints are set for mutating tools, and `openWorldHint` reflects the external reach. Missing annotations default to "destructive, open-world".
- **Search:** `toolAnnotations`, `ToolAnnotations\(`.
- **Ref:** spec-primitives.md §4.3
- **Fix:** add the annotations.

### TOOL-04 · HIGH · Structured output conforms to `outputSchema`
- **Applies:** tools with `outputSchema` or `structuredContent`.
- **Check:**
  - whenever `structuredContent` is present, it conforms to `outputSchema` (MUST). Error results (`isError = true`) may omit it;
  - the 2025-11-25 root is `type: "object"`;
  - a serialized JSON `TextContent` is also returned (SHOULD).
- **Search:** `outputSchema`, `structuredContent`.
- **Ref:** spec-primitives.md §4.4
- **Fix:** align the output, or drop `outputSchema`.

### TOOL-05 · MEDIUM · Descriptions written for the model
- **Applies:** tools.
- **Check:** descriptions state what the tool does, when to use it, the input formats with examples, the limits (max results, size), and the lifetime of any returned handle. Avoid vague one-liners.
- **Ref:** spec-primitives.md §1.1, §4.7
- **Fix:** rewrite the descriptions.

### TOOL-06 · MEDIUM · Experimental tasks used deliberately
- **Applies:** `execution = ToolExecution(...)` or task APIs used.
- **Check:** `taskSupport` is set only when tasks are really implemented. Tasks are experimental in 2025-11-25 and moved to an extension in 2026.
- **Search:** `ToolExecution`, `taskSupport`, `tasks\s*=`.
- **Ref:** spec-primitives.md §4.1, spec-2026-07-28-delta.md §1
- **Fix:** remove it unless needed.

---

## UTIL: Utilities

### UTIL-01 · MEDIUM · MCP logging notifications used responsibly
- **Applies:** `sendLoggingMessage` used.
- **Check:**
  - the `logging` capability is declared (MUST; report as HIGH when missing);
  - rate-limited;
  - consistent `logger` names.
  - Secrets or PII in payloads → report under SEC-02.
- **Search:** `sendLoggingMessage`, `LoggingMessageNotification`.
- **Ref:** spec-core.md §7
- **Fix:** add redaction and throttling.

### UTIL-02 · MEDIUM · Progress notifications correct
- **Applies:** progress notifications sent.
- **Check:**
  - notifications are sent only when the request carried a `progressToken`;
  - `progress` strictly increases;
  - sending stops after completion;
  - the rate is limited.
- **Search:** `ProgressNotification`, `progressToken`.
- **Ref:** spec-core.md §8
- **Fix:** guard the sends, and use a monotonic counter.

### UTIL-03 · MEDIUM · Handlers are cancellable
- **Applies:** always.
- **Check:** long operations suspend cooperatively (`withContext(Dispatchers.IO)`, `ensureActive()` in loops). A cancellation SHOULD at the spec level; swallowed `CancellationException` is reported under KT-03.
- **Search:** `catch\s*\(\s*\w+\s*:\s*(Exception|Throwable)\s*\)`, `runCatching`, loops in handlers.
- **Ref:** spec-core.md §9, kotlin-sdk.md §6
- **Fix:** rethrow `CancellationException`, and add `ensureActive()`.

### UTIL-04 · MEDIUM · Completion handler bounded and safe
- **Applies:** completions capability.
- **Check:** at most 100 values; `hasMore` / `total` are coherent; input is validated; results are filtered by the caller's access.
- **Search:** `CompleteRequest`, `CompleteResult`.
- **Ref:** spec-core.md §11
- **Fix:** cap and filter.

### UTIL-05 · MEDIUM · Timeouts on outbound calls
- **Applies:** upstream calls.
- **Check:** HTTP and DB clients have connect, request and socket timeouts below the MCP request timeout (60 s by default in the SDK).
- **Search:** `HttpTimeout`, `requestTimeoutMillis`, `connectTimeout`, `callTimeout`.
- **Ref:** security.md §6, spec-core.md §1
- **Fix:** configure the timeouts.

---

## KT: Kotlin and coroutines

### KT-01 · HIGH · No blocking I/O on the default dispatcher
- **Applies:** SDK ≥ 0.13.0, where handlers run on `Dispatchers.Default`.
- **Check:** blocking calls inside handlers are wrapped in `withContext(Dispatchers.IO)`.
- **Search:** `\.execute\(\)`, `readText\(`, `readBytes\(`, `Files\.`, `URL\(.*\)\.(openStream|readText)`, `Thread\.sleep`, `\.get\(\)` on futures, `JdbcTemplate`, `DriverManager`.
- **Ref:** kotlin-sdk.md §6
- **Fix:** add `withContext(Dispatchers.IO)`, or use suspending clients (Ktor client).

### KT-02 · HIGH · Shared mutable state is thread-safe
- **Applies:** SDK ≥ 0.15.0 (concurrent handlers), or any HTTP transport.
- **Check:** caches, registries and counters touched by handlers use concurrent structures or a `Mutex`.
- **Search:** `mutableMapOf\(`, `HashMap\(`, `mutableListOf\(` in `object` or class fields, `var ` in singletons, `lateinit var`.
- **Ref:** kotlin-sdk.md §6
- **Fix:** use `ConcurrentHashMap`, `AtomicReference`, `Mutex`, or immutable swaps.

### KT-03 · HIGH · `CancellationException` is not swallowed (robustness: breaks cancellation, shutdown and structured concurrency)
- **Applies:** always.
- **Check:** broad catches and `runCatching` around suspend calls rethrow `CancellationException`.
- **Search:** `catch\s*\(\s*\w+\s*:\s*(Exception|Throwable)`, `runCatching\s*\{`.
- **Ref:** kotlin-sdk.md §6
- **Fix:** add `catch (e: CancellationException) { throw e }` before the broad catch, or narrow the catch.

### KT-04 · MEDIUM · No `runBlocking` or `GlobalScope` in request paths
- **Applies:** always.
- **Search:** `runBlocking` (acceptable only in `main`), `GlobalScope`.
- **Ref:** kotlin-sdk.md §6
- **Fix:** use suspend functions, or a structured scope owned by the server.

### KT-05 · MEDIUM · Dynamic registration is safe
- **Applies:** registrations after startup (SDK ≥ 0.15.0 throws on duplicates).
- **Check:** re-registration removes first, and concurrent updates are serialized.
- **Search:** `addTool|addResource|addPrompt` outside initialization, `remove(Tool|Resource|Prompt)`.
- **Ref:** kotlin-sdk.md §2
- **Fix:** remove then add, under a `Mutex`.

### KT-06 · LOW · HTTP client lifecycle
- **Applies:** outbound HTTP.
- **Check:** one shared client instance, closed on shutdown; not created per request.
- **Search:** `HttpClient\(` inside handlers or functions called per request.
- **Ref:** kotlin-sdk.md §6
- **Fix:** inject a singleton client.

### KT-07 · LOW · JSON built with kotlinx.serialization, not strings
- **Applies:** always.
- **Search:** string templates producing JSON (`"\{\\"`, `"""\{`).
- **Ref:** kotlin-sdk.md §3 (types built with the SDK and `buildJsonObject`); spec-core.md §4 (messages must be valid JSON-RPC)
- **Fix:** use `buildJsonObject { }` or `@Serializable` classes.

### KT-08 · LOW · Configuration externalized
- **Applies:** always.
- **Check:** upstream base URLs, allowlists, limits and timeouts come from env or config, with validated defaults.
- **Search:** hardcoded `https?://` hosts in `src/main`.
- **Ref:** security.md §2.3, §4 (allowlists, secrets from environment); transport-stdio.md §4
- **Fix:** add a config class loaded at startup and validated fail-fast.

---

## TEST: Tests and operability

### TEST-01 · MEDIUM · Automated MCP-level tests
- **Applies:** always.
- **Check:** tests exercise the server through an MCP client (`kotlin-sdk-testing` in-process transports, or `kotlin-sdk-client`). They cover list, read, get and call, plus error paths (not found, invalid params).
- **Search:** `src/test`, `kotlin-sdk-testing`, `\bClient\(`.
- **Ref:** kotlin-sdk.md §9
- **Fix:** add an in-process test suite.

### TEST-02 · LOW · stdout-cleanliness test for stdio
- **Applies:** stdio transport.
- **Check:** a test or script launches the built server, sends `initialize`, and asserts that every stdout line is valid JSON-RPC.
- **Ref:** build-and-run.md §3
- **Fix:** add the test, or document the manual check.

### TEST-03 · LOW · Run and configuration documented
- **Applies:** always.
- **Check:** the README shows the build command, the launch command and the required env vars, gives an example client configuration, and states the SDK and protocol versions.
- **Ref:** build-and-run.md §2; transport-stdio.md §2 (no `gradlew run`), §4 (env credentials)
- **Fix:** document these.

---

## FUT: Forward compatibility with MCP 2026-07-28 (always INFO unless the project targets 2026)

### FUT-01 · INFO · No reliance on connection or session state
- **Check:** handlers do not remember earlier requests per `sessionId` or connection (for example `onInitialized` state, per-session maps). Multi-step state uses explicit handles passed as arguments.
- **Search:** `sessionId`, `onInitialized`, `clientConnection\(`, maps keyed by session.
- **Ref:** spec-2026-07-28-delta.md §1, §5

### FUT-02 · INFO · Server-to-client requests isolated or avoided
- **Check:** `createMessage` (sampling, deprecated), `listRoots` (roots, deprecated), `createElicitation` and `ping()` are not adopted for new features, or are isolated behind one function. In 2026 they become MRTR `InputRequiredResult`.
- **Search:** `createMessage`, `listRoots`, `createElicitation`, `\.ping\(`.
- **Ref:** spec-2026-07-28-delta.md §1, §3

### FUT-03 · INFO · Error mapping centralized
- **Check:** not-found and invalid-params errors come from one helper, so the -32002 → -32602 switch is a one-line change.
- **Ref:** spec-2026-07-28-delta.md §2

### FUT-04 · INFO · No product features on MCP logging
- **Check:** MCP `notifications/message` is not relied on (it is deprecated in 2026); operational logging goes to stderr, OpenTelemetry or standard logs.
- **Ref:** spec-2026-07-28-delta.md §3

### FUT-05 · INFO · Resource change detection decoupled from `resources/subscribe`
- **Check:** if subscriptions exist, change detection is separate from the SDK subscribe API, ready for `subscriptions/listen`.
- **Ref:** spec-2026-07-28-delta.md §1

### FUT-06 · INFO · Results classified public vs user-specific
- **Check:** the project knows which list and read results depend on the caller, for the future `cacheScope` and `ttlMs`.
- **Ref:** spec-2026-07-28-delta.md §2

### FUT-07 · INFO · Deterministic registration order
- **Check:** tools, prompts and resources are registered in a stable order (not by iterating a `HashMap` / `HashSet`).
- **Search:** registration loops over `HashMap`, `HashSet`, `toSet\(\)`.
- **Ref:** spec-2026-07-28-delta.md §2

### FUT-08 · INFO · No hand-rolled 2026 protocol features
- **Check:** the project does not implement `server/discover`, `_meta` protocol parsing or MRTR on top of SDK ≤ 0.15.0. It should wait for SDK support.
- **Ref:** versions.md §3, spec-2026-07-28-delta.md §5
