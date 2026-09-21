# Security guidance for MCP servers (transport-independent)

This file combines spec MUSTs (quoted) with established server hygiene. Hygiene items are labelled **[practice]** and are not spec text. HTTP- and OAuth-specific rules are in `transport-http.md`.

## 1. Spec MUSTs that apply to every server

| Primitive | Requirement |
|---|---|
| Resources | "Servers **MUST** validate all resource URIs"; "Binary data **MUST** be properly encoded"; **[2026]** "**MUST** sanitize file paths to prevent directory traversal attacks" |
| Prompts | "Implementations **MUST** carefully validate all prompt inputs and outputs to prevent injection attacks or unauthorized access to resources." |
| Tools | "Servers **MUST**: Validate all tool inputs; Implement proper access controls; Rate limit tool invocations; Sanitize tool outputs" |
| Completion | "**MUST**: Validate all completion inputs; Implement appropriate rate limiting; Control access to sensitive suggestions; Prevent completion-based information disclosure" |
| Logging | "Log messages **MUST NOT** contain: Credentials or secrets; Personal identifying information; Internal system details that could aid attacks" |
| Elicitation | no secrets through form mode; URL mode for secrets; bind to the user's identity (see `spec-primitives.md` §5.1) |

## 2. Input handling

### 2.1 Path traversal and path injection

Resource URIs, template variables, tool arguments and prompt arguments are **attacker-controlled**. The model can be steered by injected content, so treat them as untrusted even when the client is your own.

- Local files: resolve against a fixed root and verify containment:

  ```kotlin
  val root = Path.of(baseDir).toRealPath()
  val target = root.resolve(userPath).normalize()
  require(target.startsWith(root)) { "Path outside allowed root" }   // then map to McpException / isError
  ```

  Also reject absolute paths, NUL bytes, and encoded traversal (`%2e%2e`, `%2f`) **after** decoding.
- Upstream APIs (for example GitLab `/projects/:id/repository/files/:file_path`): **URL-encode each path segment** (`URLEncoder.encode(path, UTF_8)` or Ktor `encodeURLPathPart`). Validate identifiers against an allow-pattern, such as `^[\w.-]+(/[\w.-]+)*$` for project paths. Never build upstream URLs with raw string templates, or `..`, `?` and `#` in the input can change the target endpoint.
- **[practice]** Restrict readable scope explicitly, for example with an allowlist of groups or projects in config. Do not rely on "whatever the service token can see".

### 2.2 Injection

- **[practice]** Never pass user input to `ProcessBuilder` / `Runtime.exec` through a shell (`sh -c`). Use argument arrays and an allowlist of commands.
- **[practice]** Use parameterized queries only: no string-built SQL, JPQL or GraphQL with user input.
- **[practice]** Validate types, lengths and enumerations in the handler. Do not trust the JSON schema alone, because clients may skip validation.

### 2.3 SSRF

**[practice]** When a server fetches URLs derived from client input:

- allow only the configured upstream base URL(s);
- block redirects to other hosts;
- block private, link-local and metadata ranges (`169.254.169.254`) unless the target is the one configured internal host;
- set connect and read timeouts.

The MCP guide's SSRF section targets OAuth discovery URLs; this generalizes it.

## 3. Output handling and prompt injection

- Content returned to the client ends up in a model context. Documentation, READMEs, issues and code fetched from an upstream system can contain **prompt-injection text** ("ignore previous instructions…").
- The spec says tools **MUST** "Sanitize tool outputs" and prompts **MUST** validate outputs. Reasonable server measures **[practice]**:
  - return upstream content as data (resource contents, embedded resources), not merged into instruction-like prompt text;
  - clearly delimit untrusted content inside prompt templates, and state its origin;
  - strip or escape control characters; bound the size;
  - use `annotations.audience` where helpful;
  - never follow instructions found in fetched content from inside the server.
- **Error messages** are output too. Do not leak stack traces, internal hostnames, SQL, tokens or file-system paths. The Kotlin SDK forwards `e.message` of any exception thrown by a tool handler, and of any non-`McpException` thrown by other handlers.

## 4. Secrets

- **[practice]** No hardcoded secrets in code, resources, tests or build files. Search patterns:
  - GitLab tokens: `glpat-[0-9A-Za-z_-]{20,}`, `gldt-`, `glrt-`
  - GitHub tokens: `ghp_`, `github_pat_`
  - AWS keys: `AKIA[0-9A-Z]{16}`
  - private keys: `-----BEGIN (RSA |EC |OPENSSH )?PRIVATE KEY-----`. The pattern starts with `-`, so pass it with `rg -e '…'` or after `--`.
  - bearer tokens: `Bearer [A-Za-z0-9._-]{20,}`
  - password assignments: `(password|passwd|secret|token|apikey|api_key)\s*[:=]\s*"[^"$]{6,}"` (case-insensitive)
- Credentials come from environment or a secret store. On stdio: > retrieve credentials from the environment.
- **[practice]** Never log secrets: no request or header dumps, no `Authorization` values, no URLs carrying tokens. Mask secrets in `toString()` of config classes (a Kotlin `data class` prints every field).
- **[practice]** Least privilege for upstream credentials. For GitLab documentation reading, `read_api` or `read_repository` is enough; not `api`, not admin, not write scopes.
- **In the audit report, never print a full secret.** Show at most the first 4 characters followed by `…`.

## 5. Access control

- **[practice]** Decide who may read what.
  - With a single shared service token, every user of the MCP server sees everything that token sees. Acceptable for stdio with a personal token; a finding for multi-user HTTP.
  - Per-user filtering must be applied on list **and** read. Read-by-guessed-URI must also be checked, not only listing.
- **[2026]** List results may vary by authorization, but never per connection. `cacheScope: "private"` is needed for user-specific results. Plan for this.

## 6. Availability and abuse

- Spec: tools **MUST** be rate-limited, completion **MUST** be rate-limited, and logging **SHOULD** be rate-limited.
- **[practice]** Bound work per request: result size, page size, recursion depth and upstream fan-out.
  - Messages above 16 MiB cannot be read by Kotlin-SDK peers (frame cap on the reading side).
  - HTTP bodies above 4 MiB get 413 by default.
- **[practice]** Timeouts on every upstream call (Ktor `HttpTimeout`, OkHttp timeouts). A hung upstream must not hang the MCP request forever.
- **[practice]** Cache upstream reads with a TTL to avoid hammering the upstream. Make the cache thread-safe.

## 7. TLS and upstream trust

**[practice]** Never disable certificate validation. Search patterns:

- `TrustManager` with empty `checkServerTrusted`
- `hostnameVerifier { _, _ -> true }`
- `trustAllCerts`
- `InsecureTrustManagerFactory`
- `sslContext.init(null, arrayOf(trustAll)`

For an internal CA, import it into a truststore instead.

## 8. Supply chain

- **[practice]**
  - Pin dependency versions: no `+` or `latest.release`.
  - Use the Ktor BOM.
  - Use Gradle dependency verification or locking where the organization supports it.
  - Resolve from the organization's Maven mirror.
- Vulnerability scanning is not possible offline. Recommend running the organization's scanner (OWASP dependency-check, Snyk, Dependabot-equivalent) in CI.
