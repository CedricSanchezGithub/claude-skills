# MCP server primitives (baseline: 2025-11-25, with [2026] notes)

This file covers resources, resource templates, subscriptions, prompts, tools, content blocks, annotations, and what a server must respect when it uses elicitation, sampling or roots. Quotes are verbatim spec text.

---

## 1. Shared metadata

### 1.1 `name` vs `title`

- `name` is the programmatic identifier. `title` is the optional display name for humans.
- For tools, display precedence is `title`, then `annotations.title`, then `name`.
- `description` "can be used by clients to improve the LLM's understanding … It can be thought of like a "hint" to the model." Descriptions are therefore functional, not decoration.

### 1.2 Icons (2025-11-25 and later)

`icons?: Icon[]` on Implementation, Resource, ResourceTemplate, Prompt and Tool. Each icon is `{src, mimeType?, sizes?: ["48x48"|"any"], theme?: "light"|"dark"}`, where `src` is an HTTP(S) URL or a `data:` URI. Clients must support PNG and JPEG. SVG may contain script, so consumers must take precautions with it.

### 1.3 Annotations (resources, templates, content blocks)

- `audience`: array of `"user"` and/or `"assistant"`, the only valid values.
- `priority`: number from 0.0 to 1.0. 1 means "effectively required" and 0 means "entirely optional".
- `lastModified`: ISO 8601 string, for example `"2025-01-12T15:00:58Z"`.

### 1.4 Content blocks

`ContentBlock = TextContent | ImageContent | AudioContent | ResourceLink | EmbeddedResource`

- `text`: `{type:"text", text}`
- `image` / `audio`: `{type, data: base64, mimeType}`. "The image data **MUST** be base64-encoded and include a valid MIME type." The same applies to audio.
- `resource_link`: a `Resource` plus `type:"resource_link"`. "Resource links returned by tools are not guaranteed to appear in the results of resources/list".
- `resource` (embedded): `{type:"resource", resource: TextResourceContents | BlobResourceContents}`. It **MUST** include a valid URI, the appropriate MIME type, and either text or base64 blob.

---

## 2. Resources

Resources are **application-driven**: the host decides what enters the context. The model does not pull resources on its own unless the client builds that feature. Each resource is identified by a URI (RFC 3986).

### 2.1 Capability

> Servers that support resources **MUST** declare the `resources` capability.

- `subscribe`: "whether the client can subscribe to be notified of changes to individual resources."
- `listChanged`: "whether the server will emit notifications when the list of available resources changes."
- Both are optional. `{}` is valid.

### 2.2 Methods

| Method | Notes |
|---|---|
| `resources/list` | paginated (`cursor` / `nextCursor`) |
| `resources/read` | params `uri`; result `contents: (TextResourceContents \| BlobResourceContents)[]` |
| `resources/templates/list` | returns `resourceTemplates` |
| `resources/subscribe` / `resources/unsubscribe` | only if `subscribe` is true. **[2026]** Replaced by `subscriptions/listen` |
| `notifications/resources/list_changed` | > servers that declared the `listChanged` capability **SHOULD** send a notification |
| `notifications/resources/updated` | `{uri}` sent to subscribers when a subscribed resource changes |

### 2.3 Data types

```ts
interface Resource { uri; name; title?; description?; mimeType?; annotations?; size?; icons?; _meta? }
// size: "raw resource content, in bytes (before base64 encoding or any tokenization), if known.
//        This can be used by Hosts to display file sizes and estimate context window usage."
interface ResourceTemplate { uriTemplate /* RFC 6570 */; name; title?; description?; mimeType?; annotations?; icons? }
// mimeType on a template: "should only be included if all resources matching this template have the same type."
interface TextResourceContents { uri; mimeType?; text }   // "must only be set if the item can actually be represented as text"
interface BlobResourceContents { uri; mimeType?; blob }   // base64
```

- "Resource templates allow servers to expose parameterized resources using URI templates (RFC 6570). Arguments may be auto-completed through the completion API."
- **[2026]** "Servers **MAY** return multiple resource contents in response to a single `resources/read` request", for example for a directory.

### 2.4 URI schemes

> This list is not exhaustive—implementations are always free to use additional, custom URI schemes.

- **`https://`**: > Servers **SHOULD** use this scheme only when the client is able to fetch and load the resource directly from the web on its own—that is, it doesn't need to read the resource via the MCP server. For other use cases, servers **SHOULD** prefer to use another URI scheme, or define a custom one, even if the server will itself be downloading resource contents over the internet.
- **`file://`**: filesystem-like resources, which need not map to a physical filesystem. The server MAY use `inode/directory` for directories.
- **`git://`**: Git version control integration.
- > Custom URI schemes **MUST** be in accordance with RFC3986.

Audit implication: a server that proxies an internal system, such as GitLab, should use a custom scheme (`gitlab://…`) or `git://`. It should not expose the internal `https://` URL, which the client usually cannot fetch.

### 2.5 Errors

2025-11-25:

> Servers **SHOULD** return standard JSON-RPC errors for common failure cases: Resource not found: `-32002`; Internal errors: `-32603`

**[2026]**:

> If the requested resource does not exist, servers **MUST** return a JSON-RPC error with code `-32602` (Invalid Params). … Servers **MUST NOT** return an empty `contents` array for a non-existent resource.

Treat "empty `contents` for a missing resource" as a finding at both baselines. It is ambiguous under 2025 and a MUST violation under 2026.

### 2.6 Security considerations

> 1. Servers **MUST** validate all resource URIs
> 2. Access controls **SHOULD** be implemented for sensitive resources
> 3. Binary data **MUST** be properly encoded
> 4. Resource permissions **SHOULD** be checked before operations
> 5. **[2026]** Servers **MUST** sanitize file paths to prevent directory traversal attacks when serving `file://` resources

### 2.7 Kotlin SDK specifics

- `addResource(uri, name, description, mimeType = "text/html") { request -> ReadResourceResult(...) }`: **the default `mimeType` is `"text/html"`.** Always pass it explicitly.
- `addResourceTemplate(uriTemplate, name, description?, mimeType?) { request, variables -> ... }` (SDK ≥ 0.10.0). An exact URI wins over a template; otherwise the most specific template wins.
- Unknown URI with no matching template: the SDK throws `McpException(RESOURCE_NOT_FOUND)` (-32002). Inside a template handler, "not found" is **the developer's job**. Throw `McpException(RPCError.ErrorCode.RESOURCE_NOT_FOUND, "Resource not found", data = buildJsonObject { put("uri", uri) })` rather than returning empty contents.
- `resources.subscribe = true` makes the SDK register subscribe/unsubscribe handlers. The server must then call `sendResourceUpdated(...)` when data changes, or the flag is a lie.
- `resources.listChanged = true` makes the SDK emit `list_changed` automatically on `addResource` / `removeResource`.

---

## 3. Prompts

Prompts are **user-controlled**: "exposed from servers to clients with the intention of the user being able to explicitly select them for use." **[2026]** adds: "This refers to who decides when the prompt is used, not who authors its content."

- Capability: > Servers that support prompts **MUST** declare the `prompts` capability. `listChanged` is optional.
- `prompts/list` is paginated. `prompts/get` takes `name` and `arguments: {[key]: string}`. **Argument values are always strings.**
- `Prompt { name; title?; description?; arguments?: PromptArgument[]; icons? }`
- `PromptArgument { name; title?; description?; required? }`
- `GetPromptResult { description?; messages: PromptMessage[] }`
- `PromptMessage { role: "user"|"assistant"; content: ContentBlock }`. `content` is a **single** block, not a list.
- Prompt messages MAY include `resource_link` and embedded resources, which is useful for attaching documentation without pasting it.

Errors:

> Servers **SHOULD** return standard JSON-RPC errors for common failure cases: Invalid prompt name: `-32602`; Missing required arguments: `-32602`; Internal errors: `-32603`

> Servers **SHOULD** validate prompt arguments before processing

> Implementations **MUST** carefully validate all prompt inputs and outputs to prevent injection attacks or unauthorized access to resources.

Kotlin SDK:

- `addPrompt(name, description?, arguments?) { request -> GetPromptResult(...) }`. Arguments are read with `request.arguments?.get("x")`.
- **The SDK does not enforce `required = true`.** The handler must check for missing required arguments and throw `McpException(RPCError.ErrorCode.INVALID_PARAMS, "Missing required argument: x")`.
- Unknown prompt name: in SDK 0.15.0 this surfaces as `IllegalArgumentException`, which becomes -32603 instead of -32602. It is fixed on the unreleased main branch. This is an SDK limitation: report it as INFO, not as a project defect.

---

## 4. Tools

Tools are **model-controlled**: the model discovers and invokes them.

### 4.1 Capability and definition

- > Servers that support tools **MUST** declare the `tools` capability.
- `Tool { name; title?; description?; inputSchema; outputSchema?; annotations?; icons?; execution? /* 2025 only */; _meta? }`
- `inputSchema`:
  - "**MUST** be a valid JSON Schema object (not `null`)" with root `type: "object"`.
  - It defaults to 2020-12.
  - For tools with no parameters, `{ "type": "object", "additionalProperties": false }` is **Recommended**.
- `outputSchema` (2025-11-25): root `type: "object"`. **[2026]** Any JSON Schema, and `structuredContent` may be any JSON value.
- `execution.taskSupport`: `"forbidden"` (default), `"optional"` or `"required"`. This is for experimental tasks in 2025-11-25 only. **[2026]** Removed; tasks are now an extension.

### 4.2 Tool names

> * Tool names **SHOULD** be between 1 and 128 characters in length (inclusive).
> * Tool names **SHOULD** be considered case-sensitive.
> * The following **SHOULD** be the only allowed characters: uppercase and lowercase ASCII letters (A-Z, a-z), digits (0-9), underscore (_), hyphen (-), and dot (.)
> * Tool names **SHOULD NOT** contain spaces, commas, or other special characters.
> * Tool names **SHOULD** be unique within a server.

Regex: `^[A-Za-z0-9_.-]{1,128}$`. Since 0.12.0 the Kotlin SDK logs a warning for invalid names.

### 4.3 ToolAnnotations (hints; clients treat them as untrusted)

| Field | Meaning | Default |
|---|---|---|
| `title` | display name | none |
| `readOnlyHint` | does not modify its environment | false |
| `destructiveHint` | may perform destructive updates (meaningful only if not read-only) | true |
| `idempotentHint` | repeated calls with the same args have no extra effect (meaningful only if not read-only) | false |
| `openWorldHint` | interacts with an open world of external entities | true |

A read-only documentation lookup should say `readOnlyHint = true`. Otherwise clients assume it is destructive and open-world.

### 4.4 Results

- `CallToolResult { content: ContentBlock[]; structuredContent?; isError? }`
- > For backwards compatibility, a tool that returns structured content SHOULD also return the serialized JSON in a TextContent block.
- If `outputSchema` is present: > Servers **MUST** provide structured results that conform to this schema.
- > Servers that use embedded resources **SHOULD** implement the `resources` capability.

### 4.5 Errors: two mechanisms

1. **Protocol errors** (JSON-RPC error): unknown tool, malformed request, server errors. Example: `{"code": -32602, "message": "Unknown tool: invalid_tool_name"}`.
2. **Tool execution errors** (`isError: true` in the result): API failures, **input validation errors** (such as a date in the wrong format or a value out of range), and business logic errors.
   > Tool Execution Errors contain actionable feedback that language models can use to self-correct and retry with adjusted parameters.

Kotlin SDK behaviour (`Server.handleCallTool`, 0.15.0):

- Unknown tool: `CallToolResult(isError = true, "Tool X not found")`.
- **Any exception thrown by a handler is caught and returned as `"Error executing tool X: ${e.message}"` with `isError = true`.** Exception messages, which may contain URLs, SQL, stack details, tokens or internal hostnames, therefore reach the client and the model. Handlers should catch expected failures and return a curated message.

### 4.6 Security considerations

> Servers **MUST**: Validate all tool inputs; Implement proper access controls; Rate limit tool invocations; Sanitize tool outputs

### 4.7 [2026] additions to keep in mind

- `tools/list` **SHOULD** use a deterministic order, and **MUST NOT** vary per connection or as a side effect of other requests.
- There is an `x-mcp-header` schema annotation with strict constraints. Never put it on sensitive params.
- Stateful tools (non-normative guidance): return an explicit opaque handle; validate the caller's authorization on every call; state its lifetime in the description; an expired or unknown handle is a tool execution error.

---

## 5. Server use of client features (elicitation, sampling, roots)

These apply only if the server calls `createElicitation`, `createMessage` or `listRoots` (Kotlin: methods on `ClientConnection` or `ServerSession`).

### 5.1 Elicitation (2025-11-25)

- Check the client capability first. > Servers **MUST NOT** send elicitation requests with modes that are not supported by the client. An empty `elicitation: {}` means form mode only.
- > Servers **MUST NOT** use form mode elicitation to request sensitive information such as passwords, API keys, access tokens, or payment credentials. / Servers **MUST** use URL mode for interactions involving such sensitive information
- Form schemas are flat objects of primitive properties only (string, number/integer, boolean, enum).
- Handle all three actions, `accept`, `decline` and `cancel`. **[2026]** "**MUST** handle cases where the user declines or cancels".
- URL mode:
  - **MUST NOT** put credentials or PII in the URL.
  - **MUST NOT** use a pre-authenticated URL.
  - **SHOULD** use HTTPS outside development.
  - **MUST** verify that the user who opens the URL is the one who started the elicitation.
  - **MUST NOT** rely on URL elicitation to authorize users for the MCP server itself.
  - **MUST NOT** pass third-party credentials to the client.
- State: > State **MUST NOT** be associated with session IDs alone. State storage must be protected. For remote servers, user identity must come from MCP authorization (for example the `sub` claim).
- `notifications/elicitation/complete` and `-32042 URLElicitationRequiredError` exist in 2025-11-25 only. **[2026]** Both are removed; elicitation happens through MRTR `InputRequiredResult`.

### 5.2 Sampling

- Check that the client declared `sampling` before calling `createMessage`. Tool-enabled sampling requires `sampling.tools`.
- Avoid `includeContext: "thisServer"|"allServers"` unless the client declares `sampling.context`.
- In tool loops, every `tool_use` must be answered by a matching `tool_result`, and a message holding tool results must hold only tool results.
- **[2026]** Sampling is **deprecated**: "New implementations **SHOULD NOT** adopt it; … migrate to integrating directly with LLM provider APIs."

### 5.3 Roots

- Check the `roots` capability before calling `listRoots`. Root URIs are `file://`.
- Servers **SHOULD** handle roots becoming unavailable, respect root boundaries, and validate paths against roots.
- **[2026]** Roots are **deprecated**: "pass directories or files via tool parameters, resource URIs, or server configuration". Roots were never an access-control mechanism.
