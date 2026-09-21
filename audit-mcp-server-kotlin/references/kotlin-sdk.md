# Kotlin MCP SDK reference (server side, as of 0.15.0)

Signatures below were read from the SDK source at tag 0.15.0. Packages:

- types: `io.modelcontextprotocol.kotlin.sdk.types.*` (since 0.8.0)
- server: `io.modelcontextprotocol.kotlin.sdk.server.*`
- protocol internals: `io.modelcontextprotocol.kotlin.sdk.shared.*`

## 1. Gradle (Kotlin DSL)

```kotlin
// gradle/libs.versions.toml
// [versions] mcp-kotlin = "0.15.0"   ktor = "3.5.x"
// [libraries]
// mcp-kotlin-server = { group = "io.modelcontextprotocol", name = "kotlin-sdk-server", version.ref = "mcp-kotlin" }
// ktor-bom = { group = "io.ktor", name = "ktor-bom", version.ref = "ktor" }

dependencies {
    implementation(platform(libs.ktor.bom))
    implementation(libs.mcp.kotlin.server)
    implementation("io.ktor:ktor-server-cio")      // only for HTTP; engines are NOT transitive
    implementation("org.slf4j:slf4j-simple:…")     // logs to stderr by default: safe for stdio
    testImplementation("io.modelcontextprotocol:kotlin-sdk-testing:$mcp")  // in-process transports (≥0.9.0)
    testImplementation("io.modelcontextprotocol:kotlin-sdk-client:$mcp")
}
kotlin { jvmToolchain(17) }   // SDK targets JVM 11+
```

Where to find the SDK version in a project:

- `gradle/libs.versions.toml`
- `build.gradle.kts` (`io.modelcontextprotocol:kotlin-sdk`)
- `gradle.properties`
- `buildSrc/` or convention plugins

## 2. Server construction

```kotlin
val server = Server(
    serverInfo = Implementation(name = "gitlab-docs", version = BuildConfig.VERSION, title = "GitLab Docs"),
    options = ServerOptions(
        capabilities = ServerCapabilities(
            resources = ServerCapabilities.Resources(listChanged = false, subscribe = false),
            prompts = ServerCapabilities.Prompts(listChanged = false),
            // tools = ServerCapabilities.Tools(listChanged = false),
            // logging = ServerCapabilities.Logging,
            // completions = ServerCapabilities.Completions,
        ),
    ),
    instructions = "Read-only access to internal library documentation hosted on GitLab.",
)
```

- `Implementation(name, version, title?, websiteUrl?, icons?)`
- `ServerOptions(capabilities, enforceStrictCapabilities = true, resourceTemplateMatcherFactory = …, handlerCoroutineContext = Dispatchers.Default)`
- `enforceStrictCapabilities = true` (the server default) makes the SDK **throw** when server code calls a client feature the client did not declare. That covers sampling, roots and elicitation, plus URL-mode elicitation without `elicitation.url` and tool-enabled sampling without `sampling.tools`. It throws instead of sending, so the spec is not violated, but an unguarded call fails at runtime; inside a tool, the exception text reaches the client. Setting it to `false` removes this safety net.
- **Not checked by the SDK:** a `setRequestHandler<CompleteRequest>` without the `completions` capability declared. The server then serves completions while breaking a spec MUST.
- Registration **throws `IllegalStateException` if the matching capability is not declared**. A missing capability therefore shows up at startup, not silently.
- `listChanged = true` makes the SDK emit `notifications/*/list_changed` to every session on add/remove. `resources.subscribe = true` registers subscribe/unsubscribe handlers.
- **Duplicate registration throws `IllegalArgumentException` since 0.15.0.** Re-registering a tool, prompt or resource requires `remove*` first.

## 3. Registration APIs

```kotlin
fun addTool(name: String, description: String, inputSchema: ToolSchema = ToolSchema(), title: String? = null,
            outputSchema: ToolSchema? = null, toolAnnotations: ToolAnnotations? = null,
            execution: ToolExecution? = null, meta: JsonObject? = null,
            handler: suspend ClientConnection.(CallToolRequest) -> CallToolResult)
fun addPrompt(name: String, description: String? = null, arguments: List<PromptArgument>? = null,
              promptProvider: suspend ClientConnection.(GetPromptRequest) -> GetPromptResult)
fun addResource(uri: String, name: String, description: String, mimeType: String = "text/html",   // ← default!
                readHandler: suspend ClientConnection.(ReadResourceRequest) -> ReadResourceResult)
fun addResourceTemplate(uriTemplate: String, name: String, description: String? = null, mimeType: String? = null,
                        readHandler: suspend ClientConnection.(ReadResourceRequest, Map<String, String>) -> ReadResourceResult)
// plus addTools/addPrompts/addResources, removeTool/removePrompt/removeResource/removeResourceTemplate
```

Types commonly used:

- `ToolSchema(schema: String? /* $schema */ = null, properties: JsonObject? = null, required: List<String>? = null, defs: JsonObject? /* $defs */ = null)`. `type` is always `"object"`. **Always use named arguments**: `schema` comes first, so a positional `ToolSchema(props)` binds the properties to `$schema`. A positional call is a finding (TOOL-02).
- `ToolAnnotations(title?, readOnlyHint?, destructiveHint?, idempotentHint?, openWorldHint?)`
- `CallToolResult(content: List<ContentBlock>, isError: Boolean? = null, structuredContent: JsonObject? = null, meta: JsonObject? = null)`
- `TextContent(text, annotations?, meta?)`
- `PromptArgument(name, description, required)`
- `GetPromptResult(messages, description?)`
- `PromptMessage(role = Role.User, content = TextContent(...))`
- `ReadResourceResult(contents = listOf(TextResourceContents(text, uri, mimeType)))`, or `BlobResourceContents` for binary.
- Errors: `McpException(code: Int, message: String, data: JsonElement? = null, cause: Throwable? = null)` with `RPCError.ErrorCode.*`.

Handler receiver `ClientConnection` (≥0.9.0):

- `sessionId`
- `notification(n, relatedRequestId?)`
- `sendLoggingMessage(...)`
- `sendResourceUpdated(...)`
- `sendResourceListChanged()`, `sendToolListChanged()`, `sendPromptListChanged()`
- `ping()`, `createMessage(...)` (sampling), `listRoots(...)`, `createElicitation(...)`

`currentRequestHandlerExtra()` gives `requestId` and `sendNotification(...)`, for example progress tied to the request's own stream. The exact progress-token accessor on request `meta` varies: verify it in the IDE before proposing code.

## 4. Error behaviour of the SDK (important for the audit)

| Situation | What the client receives |
|---|---|
| Tool handler throws any `Exception` | `CallToolResult(isError = true, "Error executing tool X: ${e.message}")`. **The message leaks** |
| Unknown tool | `CallToolResult(isError = true, "Tool X not found")` |
| Other handler throws `McpException` | JSON-RPC error with its code, message and data |
| Other handler throws another exception | JSON-RPC `-32603` with `cause.message`. **The message leaks** |
| Params fail to deserialize | `-32602` |
| Unknown resource URI (no template match) | `McpException(RESOURCE_NOT_FOUND = -32002)` |
| Unknown prompt name | 0.15.0: `IllegalArgumentException` becomes `-32603` (SDK bug, fixed on main to `-32602`) |
| Missing required prompt argument | **not checked by the SDK.** The handler must check |
| `CancellationException` | rethrown (cooperative cancellation) |

Recommended pattern:

```kotlin
addTool(name = "search_docs", description = "…", inputSchema = …,
        toolAnnotations = ToolAnnotations(readOnlyHint = true, openWorldHint = false)) { request ->
    val query = request.arguments?.get("query")?.jsonPrimitive?.contentOrNull
        ?: return@addTool CallToolResult(listOf(TextContent("Missing required argument 'query'.")), isError = true)
    try {
        val hits = withContext(Dispatchers.IO) { docs.search(query) }
        CallToolResult(listOf(TextContent(render(hits))))
    } catch (e: CancellationException) {
        throw e
    } catch (e: UpstreamNotFound) {
        CallToolResult(listOf(TextContent("No documentation found for '$query'.")), isError = true)
    } catch (e: Exception) {
        log.error(e) { "search_docs failed" }                 // full details to stderr/log only
        CallToolResult(listOf(TextContent("Documentation backend unavailable, retry later.")), isError = true)
    }
}
```

## 5. Transports

- stdio: see `transport-stdio.md` §3.
- HTTP: see `transport-http.md` §3.
- Connecting: `server.createSession(transport)`. `connect()` was removed in 0.10.0.

## 6. Concurrency and coroutines (common defects)

- Since **0.13.0**, handlers run on `Dispatchers.Default` (before that, IO). **Blocking I/O in a handler starves the CPU pool.** Wrap blocking calls in `withContext(Dispatchers.IO) { … }`. Blocking calls include JDBC, `java.net.URL.readText()`, `HttpURLConnection`, blocking OkHttp `execute()`, `File.readText()`, `Files.*` and `Thread.sleep`.
- Since **0.15.0**, handlers run **concurrently**. Shared mutable state must be thread-safe:
  - use `ConcurrentHashMap`, `AtomicReference`, `Mutex`, or immutable snapshots;
  - avoid `mutableMapOf()` / `HashMap` / `var` fields in shared objects mutated from handlers.
- **Swallowing cancellation:** `catch (e: Exception)`, `catch (e: Throwable)` and `runCatching { }` around suspending calls also catch `CancellationException`. Rethrow it, or catch narrower types. Otherwise cancellation (the client's `notifications/cancelled`, a closed stream, shutdown) silently fails.
- Do not use `runBlocking` inside handlers, and do not use `GlobalScope.launch` for work tied to a request.
- Create one shared Ktor `HttpClient` (with `HttpTimeout`) and close it on shutdown. Do not create a client per request.

## 7. Deprecated and removed APIs (use to flag outdated code)

The deprecation policy is WARNING in release N, ERROR in N+1, removed in N+2.

| Old usage (search for it) | Replacement | Status |
|---|---|---|
| `server.connect(transport)` | `server.createSession(transport)` | removed 0.10.0 |
| imports `io.modelcontextprotocol.kotlin.sdk.Tool`, `.Implementation`, `.CallToolResult` … (no `.types`) | `io.modelcontextprotocol.kotlin.sdk.types.*` | moved 0.8.0, removed 0.10.0 |
| `McpError` | `McpException` | removed 0.10.0 |
| `Tool.Input` / `Tool.Output` | `ToolSchema` | removed 0.10.0 |
| `ErrorCode.*`, `JSONRPCError` | `RPCError.ErrorCode.*`, `RPCError` | 0.8.0 |
| `LoggingLevel.debug`, `Role.user` (lower-case) | `LoggingLevel.Debug`, `Role.User` | 0.8.0 |
| `PromptMessageContent` | `ContentBlock` | 0.8.0 |
| `EmptyRequestResult` | `EmptyResult` | 0.8.0 |
| `CreateElicitationRequest/Result` | `ElicitRequest` / `ElicitResult` | 0.8.0 |
| `ResourceReference` | `ResourceTemplateReference` | 0.7.0 |
| `Routing.mcp(…)` | `Route.mcp(…)` | 0.9.0 (breaking) |
| `Application.MCP { }` | `Application.mcp { }` (legacy SSE; prefer Streamable HTTP) | removed |
| `StreamableHttpServerTransport(enableJsonResponse = …, …)` / no-arg constructor | `StreamableHttpServerTransport(Configuration(...))` | deprecated |
| `Configuration(enableDnsRebindingProtection/allowedHosts/allowedOrigins)` | `install(DnsRebindingProtection)` or the helper params | WARNING 0.13.0 |
| `StdioServerTransport(inputStream = …, outputStream = …)` or positional 2 args | `StdioServerTransport(input = …, output = …) { }` | WARNING 0.13.0 |
| `mcpStatelessStreamableHttp(…, eventStore = …)` | overload without `eventStore` | ERROR 0.15.0 |
| `mcpWebSocket(options, handler)`, `mcpWebSocketTransport` | `mcpWebSocket { Server }` | WARNING |
| `ElicitRequestParams(message, requestedSchema)` | `ElicitRequestFormParams` / `ElicitRequestURLParams` | 0.11.0 |
| `LegacyTitledEnumSchema` | `TitledSingleSelectEnumSchema` | deprecated |
| Constructing `RequestHandlerExtra(...)` | `currentRequestHandlerExtra()` | internal since 0.15.0 |

## 8. Behaviour changes worth mentioning when a project upgrades

| From → to | Change |
|---|---|
| < 0.8.0 → 0.8.0 | Java 11 minimum; types package moved; nested `params` objects |
| → 0.9.0 | handler receiver `ClientConnection`; `Route.mcp`; `kotlin-sdk-testing` |
| → 0.10.0 | protocol 2025-11-25; all deprecated-ERROR symbols removed; `addResourceTemplate`; tasks types |
| → 0.11.0 | sealed `ElicitRequestParams`; URL elicitation; `maxRequestBodySize`; auto `ContentNegotiation` |
| → 0.12.0 | tool-name validation warnings; `$schema` on tool schemas; protocol-version header validation |
| → 0.13.0 | **DNS-rebinding protection ON by default**; handlers on `Dispatchers.Default`; stdio back-pressure; 16 MiB frame cap; stdio 2-arg constructor deprecated |
| → 0.14.0 | binary-incompatible (recompile); URL elicitation required exception; POST body limits; completion gated by capability **on the client side only** (the server does not check that `completions` is declared) |
| → 0.15.0 | **concurrent handlers**; duplicate registration throws; SSE heartbeats; Kotlin 2.4 / Ktor 3.5 |

## 9. Testing support

- The `kotlin-sdk-testing` artifact (≥0.9.0) provides in-process channel transports. Connect a real `Client` to the `Server` in a unit test and assert on `listResources`, `readResource`, `getPrompt` and `callTool`, including error paths.
- Stdout cleanliness can be tested by launching the built distribution with piped stdin/stdout. Send one `initialize` line and check that every stdout line parses as JSON-RPC.
