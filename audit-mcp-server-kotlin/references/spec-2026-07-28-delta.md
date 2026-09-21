# MCP 2026-07-28: what changes for servers (forward-compatibility reference)

The Kotlin SDK (≤ 0.15.0) does **not** support this revision yet. Use this file for:

- `FUT-*` findings (severity INFO): design choices that will make migration harder;
- full normative checks, only when the project explicitly targets 2026-07-28.

Terminology from the spec: **Modern** = 2026-07-28 and later (per-request `_meta`); **Legacy** = 2025-11-25 and earlier (`initialize`); **Dual-era** = a server that supports both.

## 1. Major changes (from the official changelog, condensed)

1. **Stateless protocol.** The `initialize` / `notifications/initialized` handshake is removed.
   - Every request carries `_meta["io.modelcontextprotocol/protocolVersion"]` and `_meta["io.modelcontextprotocol/clientCapabilities"]`, both required. A request missing either is rejected with `-32602` (HTTP 400).
   - Servers SHOULD put `_meta["io.modelcontextprotocol/serverInfo"]` in every result.
   - > Servers **MUST NOT** rely on prior requests over the same connection to establish context (e.g., capabilities, protocol version, client identity).
   - > State that needs to span multiple requests … **MUST** be referenced by an explicit identifier the client passes on each request.
2. **`server/discover` is MUST-implement.** It returns `supportedVersions`, `capabilities`, `instructions?`, serverInfo in `_meta`, and the caching fields `ttlMs` and `cacheScope`.
3. **Unsupported version:** `UnsupportedProtocolVersionError` code **-32022** with `data.supported` and `data.requested`.
4. **Sessions removed** from Streamable HTTP. There is no `Mcp-Session-Id`, and list results MUST NOT vary per connection.
5. **`subscriptions/listen`** replaces `resources/subscribe`, `resources/unsubscribe` and the HTTP GET stream.
   - Clients opt in per type: `toolsListChanged`, `promptsListChanged`, `resourcesListChanged`, `resourceSubscriptions`.
   - The server MUST first send `notifications/subscriptions/acknowledged`, and every notification on the stream carries `_meta["io.modelcontextprotocol/subscriptionId"]`.
   - The server MUST NOT send types the client did not request.
6. **Removed:** `ping`, `logging/setLevel`, `notifications/roots/list_changed`.
   - The log level is requested per request (`_meta["io.modelcontextprotocol/logLevel"]`).
   - The server MUST NOT emit `notifications/message` for a request without it.
7. **Tasks** moved out of core into the extension `io.modelcontextprotocol/tasks`. `execution.taskSupport` is gone from `Tool`.
8. **MRTR (Multi Round-Trip Requests).** Servers **MUST NOT** send JSON-RPC requests.
   - To obtain elicitation, sampling or roots, a server returns `InputRequiredResult` (`resultType: "input_required"`, `inputRequests`, `requestState`) from `tools/call`, `prompts/get` or `resources/read` only.
   - The client retries with `inputResponses`.
   - `requestState` is attacker-controlled. If it influences authorization, resource access or logic, it MUST be integrity-protected (HMAC/AEAD) and verified, and it SHOULD include the principal, an expiry and the originating request.
9. **`resultType` required** on all results: `"complete"` or `"input_required"`.
10. **No SSE resumability** (`Last-Event-ID` is removed). Closing the response stream means cancellation on HTTP.

## 2. Minor changes relevant to servers

- `tools/list` **SHOULD** have a deterministic order.
- New HTTP headers `Mcp-Method` and `Mcp-Name` are required and validated against the body; a mismatch gets 400 with **-32020 HeaderMismatch**. There is an optional `x-mcp-header` tool-parameter annotation with strict constraints.
- **Caching hints are required** on complete results of `server/discover`, `tools/list`, `prompts/list`, `resources/list`, `resources/templates/list` and `resources/read`:
  - `ttlMs` (integer ≥ 0);
  - `cacheScope` (`"public"` or `"private"`). `"public"` may be shared across users by intermediaries, so user-specific data MUST be `"private"`. Use the same `cacheScope` across all pages of a list.
- **Resource not found:** `-32602` (MUST), no longer `-32002`. **MUST NOT** return an empty `contents` array.
- New resources MUST: sanitize `file://` paths against directory traversal.
- `inputSchema` allows any 2020-12 keyword (root `type: "object"` still required). `outputSchema` and `structuredContent` may be any JSON value. Network `$ref` MUST NOT be auto-dereferenced.
- Elicitation: `notifications/elicitation/complete`, `elicitationId` and `-32042` are removed. Correlate retries through `requestState`.
- Error code policy:
  - -32000..-32019 is legacy; new code SHOULD NOT use it;
  - -32020..-32099 is spec-reserved;
  - defined codes: -32020 HeaderMismatch, -32021 MissingRequiredClientCapability (with `data.requiredCapabilities`), -32022 UnsupportedProtocolVersion.
- Implementations of 2026-07-28 **MUST NOT** emit -32002 or -32042.
- OpenTelemetry trace-context keys in `_meta`: `traceparent`, `tracestate`, `baggage`.
- `extensions` field in capabilities.

## 3. Deprecated features (registry as of 2026-07-28)

| Feature | Deprecated in | Earliest removal | Migration |
|---|---|---|---|
| Roots | 2026-07-28 | first revision on or after 2027-07-28 | pass directories or files through tool parameters, resource URIs or server configuration |
| Sampling | 2026-07-28 | first revision on or after 2027-07-28 | integrate directly with LLM provider APIs |
| Logging (`notifications/message`) | 2026-07-28 | first revision on or after 2027-07-28 | log to stderr for stdio; use OpenTelemetry for observability |
| Dynamic Client Registration | 2026-07-28 | first revision on or after 2027-07-28 | Client ID Metadata Documents |
| `includeContext: "thisServer"/"allServers"` | 2025-11-25 | follows Sampling | omit or use `"none"` |
| HTTP+SSE transport | 2025-03-26 | three months after SEP-2596 reaches Final | Streamable HTTP |

> A Deprecated feature remains part of the specification but is scheduled for removal: new implementations **SHOULD NOT** adopt it, and existing implementations **SHOULD** migrate.

## 4. Backward compatibility

- A server MAY be dual-era:
  - `initialize` selects legacy semantics, scoped to the stdio process or the HTTP session;
  - a request with modern `_meta` is served statelessly.
- A modern-only server **SHOULD** list its supported versions in any error it returns to `initialize`.
- On stdio, dual-era clients probe with `server/discover` first. A legacy server answers with some error or a timeout, and the client falls back to `initialize`.
- On HTTP, a modern server must return **recognizable JSON-RPC error bodies** on 400. An empty 400 makes clients misdetect it as legacy.

## 5. What a Kotlin project can do *now* (before SDK support)

These are the recommendations behind the `FUT-*` rules.

1. Do not build features on connection or session state: per-session maps keyed by `sessionId`, or "remember what the client said earlier". Pass explicit identifiers.
2. Keep list results independent of the connection. Filtering by authenticated identity is fine.
3. Avoid adopting **sampling** and **roots**, which are deprecated. Isolate any `createElicitation` call behind one function so it can move to MRTR.
4. Avoid building product features on MCP logging notifications. Log to stderr (stdio) and to OpenTelemetry or standard logging (HTTP).
5. Centralize error mapping, so resource-not-found (-32002 → -32602) and other code changes happen in one place.
6. Classify every list and read result as user-specific or public. This becomes `cacheScope`.
7. Register tools, prompts and resources in a stable, deterministic order.
8. Prefer `mcpStatelessStreamableHttp` for the future HTTP transport when no server-to-client requests or notifications are needed.
9. Do not hand-roll `server/discover`, `_meta` parsing or MRTR on top of the SDK. Wait for SDK support and upgrade.
