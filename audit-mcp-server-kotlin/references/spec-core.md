# MCP core rules for servers (baseline: 2025-11-25)

Server-side normative text from the MCP specification, revision 2025-11-25. Quotes are verbatim. Where 2026-07-28 differs, a **[2026]** note says how; the full delta is in `spec-2026-07-28-delta.md`.

RFC 2119 keywords: **MUST / MUST NOT** = violation if broken; **SHOULD / SHOULD NOT** = deviation that needs a justification; **MAY** = optional.

---

## 1. Lifecycle (2025-11-25)

> The initialization phase **MUST** be the first interaction between client and server.

> The server **MUST** respond with its own capabilities and information.

Example `initialize` result:

```json
{"jsonrpc":"2.0","id":1,"result":{
  "protocolVersion":"2025-11-25",
  "capabilities":{"logging":{},"prompts":{"listChanged":true},
    "resources":{"subscribe":true,"listChanged":true},"tools":{"listChanged":true}},
  "serverInfo":{"name":"ExampleServer","title":"Example Server Display Name","version":"1.0.0",
    "description":"An example MCP server providing tools and resources"},
  "instructions":"Optional instructions for the client"}}
```

- > The server **SHOULD NOT** send requests other than pings and logging before receiving the `initialized` notification.
- > Both parties **MUST**: Respect the negotiated protocol version; Only use capabilities that were successfully negotiated
- Shutdown on stdio: the client closes stdin, waits, then sends SIGTERM and finally SIGKILL. > The server **MAY** initiate shutdown by closing its output stream to the client and exiting.
- Shutdown on HTTP: "shutdown is indicated by closing the associated HTTP connection(s)."

Timeouts:

> Implementations **SHOULD** establish timeouts for all sent requests … the sender **SHOULD** issue a cancellation notification for that request and stop waiting for a response.
> … implementations **SHOULD** always enforce a maximum timeout, regardless of progress notifications …

**Kotlin SDK:** the handshake and version negotiation are handled by `ServerSession`. The server code only supplies `Implementation(...)`, `ServerOptions(capabilities = ...)` and optional `instructions`. The default request timeout in `ProtocolOptions` is 60 s.

**[2026]** No `initialize`. Capabilities are advertised by `server/discover`; client version and capabilities arrive in `_meta` on every request.

## 2. Version negotiation (2025-11-25)

> If the server supports the requested protocol version, it **MUST** respond with the same version. Otherwise, the server **MUST** respond with another protocol version it supports. This **SHOULD** be the *latest* version supported by the server.

> If using HTTP, the client **MUST** include the `MCP-Protocol-Version: <protocol-version>` HTTP header on all subsequent requests to the MCP server.

The SDK implements this. Audit only for custom code that overrides `initialize` handling, which is rare and suspicious.

## 3. Capabilities

Server capabilities: `prompts`, `resources`, `tools`, `logging`, `completions`, `tasks` (experimental), `experimental`, `extensions`.

Sub-capabilities:

- `listChanged` on prompts, resources and tools.
- `subscribe` on resources only.

Per-feature MUSTs, the same in both revisions:

- "Servers that support tools **MUST** declare the `tools` capability". The same applies to `resources` and `prompts`.
- "Servers that support completions **MUST** declare the `completions` capability"
- "Servers that emit log message notifications **MUST** declare the `logging` capability"
- Architecture: "Implemented server features must be advertised in the server's capabilities".

Client capabilities the server must respect (2025-11-25): `roots`, `sampling` (optionally with `tools` and `context`), `elicitation` (`form` and/or `url`), `tasks`. A server **MUST NOT** use a client feature the client did not declare. For example:

> Servers **MUST NOT** send elicitation requests with modes that are not supported by the client.

> Servers **MUST NOT** send tool-enabled sampling requests to Clients that have not declared support for tool use via the `sampling.tools` capability.

## 4. JSON-RPC messages

- > All messages between MCP clients and servers **MUST** follow the JSON-RPC 2.0 specification.
- > Requests **MUST** include a string or integer ID. … the ID **MUST NOT** be `null`.
- > The request ID **MUST NOT** have been previously used by the requestor within the same session.
- > Result responses **MUST** include the same ID as the request they correspond to.
- > Error responses **MUST** include an `error` field with a `code` and `message`. / Error codes **MUST** be integers.
- The error `message` "SHOULD be limited to a concise single sentence."
- > Notifications … The receiver **MUST NOT** send a response. … Notifications **MUST NOT** include an ID.

**[2026]** Every result must carry `resultType` (`"complete"` or `"input_required"`), and servers must not send JSON-RPC requests at all.

## 5. Error codes

| Code | Meaning | Server usage |
|---|---|---|
| -32700 | Parse error | transport / SDK |
| -32600 | Invalid request | SDK |
| -32601 | Method not found | unknown method; completion not supported (both revisions). **[2026]** explicitly also "a method gated behind a server capability the server did not advertise" |
| -32602 | Invalid params | unknown tool (protocol error), unknown prompt, missing required prompt args, invalid cursor, invalid log level |
| -32603 | Internal error | unexpected server failures |
| -32002 | Resource not found | 2025-11-25 `resources/read` (SHOULD). **[2026]** replaced by -32602 (MUST) |
| -32042 | URL elicitation required | 2025-11-25 only. **[2026]** removed |

**[2026]** The range -32000 to -32019 is legacy: new code **SHOULD NOT** use it. The range -32020 to -32099 is reserved for the spec: implementations **MUST NOT** emit undefined codes from it. Defined codes are -32020 HeaderMismatch, -32021 MissingRequiredClientCapability and -32022 UnsupportedProtocolVersion. Application-specific codes **SHOULD** be allocated outside -32768..-32000.

Kotlin SDK constants (`RPCError.ErrorCode`): `CONNECTION_CLOSED` -32000, `REQUEST_TIMEOUT` -32001, `RESOURCE_NOT_FOUND` -32002, `UNSUPPORTED_PROTOCOL_VERSION` -32022, `URL_ELICITATION_REQUIRED` -32042, `PARSE_ERROR`, `INVALID_REQUEST`, `METHOD_NOT_FOUND`, `INVALID_PARAMS`, `INTERNAL_ERROR`. Throw `McpException(code, message, data)` from handlers to produce a JSON-RPC error.

## 6. Pagination

Paginated operations: `resources/list`, `resources/templates/list`, `prompts/list`, `tools/list`.

- The cursor is an opaque string. "Page size is determined by the server, and clients **MUST NOT** assume a fixed page size".
- The response has an optional `nextCursor` when more results exist. The request carries `params.cursor`.
- > Servers **SHOULD**: Provide stable cursors; Handle invalid cursors gracefully
- > Invalid cursors **SHOULD** result in an error with code -32602 (Invalid params).
- Server implication: to signal the end, **omit** `nextCursor`. Never send `""`, because in 2026 an empty string is a valid cursor.

**Kotlin SDK:** the built-in list handlers return everything with `nextCursor = null`. Real pagination means overriding the handler, for example `session.setRequestHandler<ListResourcesRequest>(Method.Defined.ResourcesList) { request, _ -> ... }`. The README says: "Treat cursors as opaque—don't parse or persist them across sessions."

## 7. Logging (MCP `notifications/message`)

- Capability: > Servers that emit log message notifications **MUST** declare the `logging` capability
- Levels (RFC 5424): debug, info, notice, warning, error, critical, alert, emergency.
- The client MAY send `logging/setLevel`; after that, the server sends only that level and above. The Kotlin SDK filters automatically in `sendLoggingMessage`.
- Notification: `{"jsonrpc":"2.0","method":"notifications/message","params":{"level":"error","logger":"database","data":{...}}}`
- > Servers **SHOULD**: Rate limit log messages; Include relevant context in data field; Use consistent logger names; Remove sensitive information
- > Log messages **MUST NOT** contain: Credentials or secrets; Personal identifying information; Internal system details that could aid attacks

**[2026]** The Logging feature is **deprecated**. `logging/setLevel` is removed and the log level is requested per request through `_meta`. Migration path: "Log to `stderr` for stdio transports; use OpenTelemetry for observability".

## 8. Progress

- Token is `params._meta.progressToken`. > Progress tokens **MUST** be a string or integer value … **MUST** be unique across all active requests.
- > The `progress` value **MUST** increase with each notification, even if the total is unknown.
- > Progress notifications **MUST** only reference tokens that: Were provided in an active request; Are associated with an in-progress operation
- The receiver MAY choose not to send progress at all, and may send it at any frequency and omit `total`.
- > Both parties **SHOULD** implement rate limiting to prevent flooding / Progress notifications **MUST** stop after completion

Notification: `{"jsonrpc":"2.0","method":"notifications/progress","params":{"progressToken":"abc123","progress":50,"total":100,"message":"..."}}`

## 9. Cancellation

- `notifications/cancelled` with `requestId` and optional `reason`.
- > Receivers of cancellation notifications **SHOULD**: Stop processing the cancelled request; Free associated resources; Not send a response for the cancelled request
- > Both parties **MUST** handle these race conditions gracefully
- 2025-11-25 HTTP: "Disconnection **SHOULD NOT** be interpreted as the client cancelling its request." **[2026]** The reverse: on Streamable HTTP, closing the response stream **is** cancellation.

**Kotlin SDK:** a request is cancelled by cancelling the handler's coroutine. Handlers are cancellable only when they suspend cooperatively and do not swallow `CancellationException` (see `kotlin-sdk.md` §6).

## 10. Ping (2025-11-25)

> The receiver **MUST** respond promptly with an empty response.

The SDK answers `ping` automatically. A server MAY ping the client (`ClientConnection.ping()`). **[2026]** `ping` is removed.

## 11. Completion

- > Servers that support completions **MUST** declare the `completions` capability.
- Method `completion/complete`, with `ref` (`ref/prompt` with a `name`, or `ref/resource` with a `uri`), `argument {name, value}` and optional `context.arguments`.
- "Maximum 100 items per response" (`values`), plus optional `total` and `hasMore`.
- Errors: -32601 when not supported, -32602 for an invalid prompt name or missing arguments, -32603 for internal errors.
- > Implementations **MUST**: Validate all completion inputs; Implement appropriate rate limiting; Control access to sensitive suggestions; Prevent completion-based information disclosure

**Kotlin SDK:** declare `completions = ServerCapabilities.Completions`, then `session.setRequestHandler<CompleteRequest>(Method.Defined.CompletionComplete) { request, _ -> CompleteResult(...) }`. The server side does **not** verify that `completions` is declared when a `CompleteRequest` handler is installed. Check this yourself (LIFE-01).

## 12. `_meta` key naming (both revisions)

- The optional prefix is dot-separated labels followed by `/`. Labels start with a letter, end with a letter or digit, and may contain hyphens inside.
- Prefixes whose second label is `modelcontextprotocol` or `mcp` are **reserved**. Do not invent `io.modelcontextprotocol/...` keys.
- Use reverse-DNS prefixes for custom keys, for example `com.acme/trace-id`.

## 13. JSON Schema (tools, elicitation)

- Default dialect 2020-12. > Implementations **MUST** support at least 2020-12.
- **[2026]** "Implementations **MUST NOT** automatically dereference `$ref` values that resolve to a network URI."
