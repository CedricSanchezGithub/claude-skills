# Streamable HTTP transport and authorization (server side)

These rules apply when the project has an HTTP entry point, or plans one; in that case report them as readiness notes. The baseline is 2025-11-25; 2026-07-28 differences are marked **[2026]**.

## 1. Security rules (identical in 2025-11-25 and 2026-07-28)

> 1. Servers **MUST** validate the `Origin` header on all incoming connections to prevent DNS rebinding attacks.
>    * If the `Origin` header is present and invalid, servers **MUST** respond with HTTP 403 Forbidden.
> 2. When running locally, servers **SHOULD** bind only to localhost (127.0.0.1) rather than all network interfaces (0.0.0.0).
> 3. Servers **SHOULD** implement proper authentication for all connections.
>
> Without these protections, attackers could use DNS rebinding to interact with local MCP servers from remote websites.

## 2. Streamable HTTP, 2025-11-25 (what the Kotlin SDK implements)

- The server has a single MCP endpoint supporting POST and GET, plus DELETE for session termination.
- POST body: one JSON-RPC request, notification or response.
  - A notification or response gets **202 Accepted** with no body.
  - A request gets either `Content-Type: application/json` or `text/event-stream` (SSE).
- GET: the server either returns an SSE stream or **405 Method Not Allowed**.
- > The server **MUST** send each of its JSON-RPC messages on only one of the connected streams.
- Sessions (optional):
  - The session ID is assigned in the `Mcp-Session-Id` header of the `initialize` response.
  - It "**SHOULD** be globally unique and cryptographically secure".
  - It "**MUST** only contain visible ASCII characters (0x21 to 0x7E)".
  - After termination, the server **MUST** answer that ID with 404.
- Resumability (optional): SSE event IDs are globally unique per session, clients resume with `Last-Event-ID`, and the server **MUST NOT** replay messages from other streams.
- `MCP-Protocol-Version` header: when it is missing and the version cannot be inferred, assume `2025-03-26`. An invalid or unsupported value gets **400**.
- Deprecated: the HTTP+SSE transport from 2024-11-05 (deprecated since 2025-03-26). "New implementations **SHOULD NOT** adopt it."

**[2026]** Removed:

- the GET stream, sessions / `Mcp-Session-Id`, and `Last-Event-ID` resumability;
- server-sent JSON-RPC requests.

Added or changed:

- closing the response stream now counts as cancellation;
- new required headers `Mcp-Method` and `Mcp-Name`, validated against the body. A mismatch gets 400 with `-32020 HeaderMismatch`;
- an unknown method gets **404** with `-32601`;
- GET and DELETE SHOULD return 405;
- the server SHOULD send `X-Accel-Buffering: no` on SSE responses.

## 3. Kotlin SDK HTTP idioms

Preferred, at the application level (SDK ≥ 0.9.0 with a `path` param; DNS-rebinding protection on by default since 0.13.0):

```kotlin
embeddedServer(CIO, host = "127.0.0.1", port = port) {
    mcpStreamableHttp(path = "/mcp") { buildServer() }        // stateful, sessions + SSE
    // or: mcpStatelessStreamableHttp(path = "/mcp") { buildServer() }  // JSON responses, GET → 405
}.start(wait = true)
```

Parameters: `enableDnsRebindingProtection: Boolean = true`, `allowedHosts`, `allowedOrigins`, `eventStore` (stateful only), and `sseHeartbeatConfig` (0.15.0).

- **Default protection allows only `localhost`, `127.0.0.1` and `[::1]`.** Deploying behind a real hostname requires `allowedHosts` and `allowedOrigins`. **Disabling protection (`enableDnsRebindingProtection = false`) is a finding** unless another layer validates `Origin`.
- `allowedOrigins` is compared by hostname only. With custom `allowedHosts` and `allowedOrigins == null`, origin validation is skipped.
- On SDK **0.8.4 – 0.12.x** the default was `enableDnsRebindingProtection = false`, so the protection must be enabled explicitly.
- **Manual transport** (`StreamableHttpServerTransport(Configuration(...))` inside your own `routing { post/get/delete }`) **bypasses** the default protection unless you `install(DnsRebindingProtection) { allowedHosts = …; allowedOrigins = … }` on the route. **In the plugin, `allowedOrigins` defaults to `null`, which disables Origin validation.** Only the helpers default origins to localhost, so a manual install must set `allowedOrigins` explicitly to satisfy the Origin MUST. The `Configuration(enableDnsRebindingProtection = …)` fields are deprecated.
- The helpers install `ContentNegotiation(McpJson)` and `SSE` themselves. Installing `ContentNegotiation` yourself only produces a warning.
- The request body limit is 4 MiB by default (`maxRequestBodySize`); larger bodies get 413.
- Session IDs: the default generator is `Uuid.random()`, which is secure. A custom `setSessionIdGenerator { … }` must stay cryptographically random.
- Legacy SSE transport: `Route.mcp { }`, `Application.mcp { }`, `SseServerTransport`. Deprecated protocol-wise; the README says "Prefer Streamable HTTP for new projects."
- CORS for browser clients: the README sample uses `anyHost()` with the comment "restrict to specific origins in production". `anyHost()` in a deployed server is a finding.
- **No auth is built in.** Add Ktor `Authentication` (`bearer`, `jwt`) and wrap the MCP routes with `authenticate("…") { … }`. The official `simple-streamable-server` sample does this with manual routes.

## 4. Authorization (HTTP only; optional in the spec, but if you do auth, do it this way)

> Authorization is **OPTIONAL** for MCP implementations. When supported: Implementations using an HTTP-based transport **SHOULD** conform to this specification.

A protected MCP server is an **OAuth 2.1 resource server**.

- **Protected Resource Metadata (RFC 9728):**
  - > MCP servers **MUST** implement OAuth 2.0 Protected Resource Metadata … **MUST** include the `authorization_servers` field containing at least one authorization server.
  - Discovery: `WWW-Authenticate: Bearer resource_metadata="…"` on 401, **and/or** a well-known URI: `/.well-known/oauth-protected-resource[/<mcp-path>]`.
- **Token validation:**
  - > MCP servers **MUST** validate that access tokens were issued specifically for them as the intended audience (RFC 8707). Invalid or expired tokens **MUST** receive a HTTP 401 response.
  - > MCP servers **MUST** only accept tokens that are valid for use with their own resources. / **MUST NOT** accept or transit any other tokens.
- **No token passthrough:**
  - > If the MCP server makes requests to upstream APIs … The MCP server **MUST NOT** pass through the token it received from the MCP client.
  - Upstream calls use a separate credential, such as a service token or a token obtained through OAuth for the upstream.
- **Status codes:**
  - 401 when authorization is required or the token is invalid;
  - 403 for invalid scopes or insufficient permissions;
  - 400 for a malformed authorization request.
- **Insufficient scope:**
  - Answer `403` with `WWW-Authenticate: Bearer error="insufficient_scope", scope="…", resource_metadata="…"`.
  - Include a `scope` parameter in challenges, SHOULD.
- **[2026]**
  - Servers **MUST** honour scope hierarchies.
  - Servers **SHOULD NOT** list `offline_access` in challenges or in `scopes_supported`.
- Tokens are sent only in the `Authorization: Bearer` header, **never in the query string**.
- Transport security: authorization server endpoints MUST be served over HTTPS, and redirect URIs MUST be localhost or HTTPS. Remote MCP endpoints carrying bearer tokens therefore need TLS, whether terminated by the server or by a trusted reverse proxy.
- The canonical server URI is the audience value, for example `https://mcp.example.com/mcp`: lowercase scheme and host, no fragment, and no trailing slash unless it is meaningful.

**Enterprise note:** a corporate gateway or SSO reverse proxy that authenticates before the MCP server is a legitimate architecture. The audit should then verify that:

- the MCP server cannot be reached while bypassing the gateway (binding, network policy);
- the identity forwarded by the gateway is trusted only from the gateway;
- per-user authorization is still applied to what each user can list and read.

Report deviations from the OAuth profile under AUTH-01 as MEDIUM, not as violations. If the config file lists AUTH-01 as an accepted exception, with a justification and an expiry date, list it in report §8 instead of as a finding.

## 5. Security best practices for HTTP servers (MCP security guide)

- **Token passthrough** is an anti-pattern. It circumvents rate limiting and auditing, breaks trust boundaries, and exposes data to exfiltration.
- **Session hijacking (2025-11-25):**
  - > MCP servers that implement authorization **MUST** verify all inbound requests. MCP Servers **MUST NOT** use sessions for authentication.
  - > MCP servers **MUST** use secure, non-deterministic session IDs.
  - > MCP servers **SHOULD** bind session IDs to user-specific information … `<user_id>:<session_id>` … the user ID is derived from the user token and not provided by the client.
- **[2026] State handle hijacking:** the same rules apply to state handles minted by tools. "**MUST NOT** treat possession of a state handle as authentication."
- **Local server compromise:** local HTTP servers are reachable by any local process and by browsers via DNS rebinding. Prefer stdio for local use; otherwise require a token or use a unix socket.
- **Confused deputy:** this applies only if the server is an **OAuth proxy** to a third-party API with a static client ID. Then it needs per-client consent, exact `redirect_uri` matching, and single-use `state`.
- **Scope minimization:** publish a minimal `scopes_supported`, elevate with targeted challenges, and avoid wildcard scopes such as `*` or `full-access`.

## 6. HTTP readiness checklist (for stdio-first projects)

- [ ] A single `buildServer()` factory is shared by the stdio and HTTP mains.
- [ ] No per-user state in singletons. Upstream credentials are resolvable per request or per user when HTTP arrives.
- [ ] Authorization checks can be applied per request (for example, a filter on list and read by caller identity).
- [ ] Handlers are thread-safe (see `kotlin-sdk.md` §6). Over HTTP, many sessions run concurrently.
- [ ] The HTTP plan uses `mcpStreamableHttp` / `mcpStatelessStreamableHttp`, not legacy SSE.
- [ ] The plan includes: bind to 127.0.0.1 in dev, keep DNS-rebinding protection, auth in front, HTTPS in deployment, and restricted CORS.
- [ ] Prefer **stateless** Streamable HTTP when the server needs no server-to-client requests or notifications. It is simpler and closer to 2026-07-28.
