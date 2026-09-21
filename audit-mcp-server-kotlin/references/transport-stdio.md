# stdio transport (server side)

## 1. Spec rules (identical core in 2025-11-25 and 2026-07-28)

- "The client launches the MCP server as a subprocess."
- "The server reads JSON-RPC messages from `stdin` and writes JSON-RPC messages to `stdout`."
- > Messages are delimited by newlines, and **MUST NOT** contain embedded newlines.
- > The server **MAY** write UTF-8 strings to `stderr` for any logging purposes including informational, debug, and error messages.
- > The server **MUST NOT** write anything to its `stdout` that is not a valid MCP message.
- JSON-RPC messages **MUST** be UTF-8 encoded.
- Shutdown: the client closes stdin, waits, then sends SIGTERM and finally SIGKILL. > The server **MAY** initiate shutdown by closing its output stream to the client and exiting.
- **[2026]** > Servers **SHOULD** exit promptly when their standard input is closed or reads return end-of-file. This is good practice at any baseline.
- **[2026]** > The server **MUST NOT** write JSON-RPC *requests* to `stdout`. Cancellation on stdio: the client sends `notifications/cancelled`, and the server "**SHOULD** stop work … as soon as practical and **MUST NOT** send any further messages for it."
- Authorization: > Implementations using an STDIO transport **SHOULD NOT** follow this [OAuth] specification, and instead retrieve credentials from the environment.
- Security best practices: stdio is the recommended transport for local servers ("Use the `stdio` transport to limit access to just the MCP client").

## 2. What pollutes stdout in a Kotlin/JVM server

Any of the following corrupts the protocol stream in stdio mode. The client usually fails with a JSON parse error or silently drops the server.

| Source | Search pattern (regex) | Safe alternative |
|---|---|---|
| `println` / `print` | `\bprintln\(`, `\bprint\(` | logger → stderr, or `System.err.println` |
| `System.out` direct use | `System\.out\b`, `System\.\`out\`` (ignore the one passed to `StdioServerTransport`) | stderr |
| Logback console appender | `ConsoleAppender` in `logback.xml` / `logback-test.xml` without `<target>System.err</target>` | Logback's ConsoleAppender **defaults to System.out**, so set `<target>System.err</target>` |
| Logback / Log4j2 **without any config file** | `logback-classic` or `log4j-core` on the runtime classpath and no `logback*.xml` / `log4j2*.xml` in `src/main/resources` | both default configurations log to the console on **stdout**, so add a config with a stderr target |
| Log4j2 console appender | `<Console` without `target="SYSTEM_ERR"` in `log4j2.xml` | The Log4j2 Console appender defaults to `SYSTEM_OUT`, so set `target="SYSTEM_ERR"` |
| slf4j-simple | `org.slf4j.simpleLogger.logFile=System.out` in `simplelogger.properties` | default is `System.err`, which is fine |
| java.util.logging | custom `StreamHandler(System.out, …)` | `ConsoleHandler` (stderr by default) |
| Ktor / other banners | `embeddedServer(` started in stdio mode, ASCII banners, `showBanner` | do not start HTTP engines in stdio mode, or log them to stderr |
| Child processes | `ProcessBuilder(...).inheritIO()`, `redirectOutput(ProcessBuilder.Redirect.INHERIT)` | capture or redirect child output; never inherit stdout |
| Pretty-printed JSON | custom `Json { prettyPrint = true }` used for MCP messages | use the SDK's `McpJson` |
| Gradle `run` wrapper | clients launching `./gradlew run` (Gradle prints to stdout) | launch the built jar or installed distribution: `java -jar`, `build/install/<app>/bin/<app>` |

`e.printStackTrace()` writes to stderr, so it does not break the protocol, but it is still poor logging practice.

## 3. Kotlin SDK stdio idioms (0.13.0 – 0.15.0)

Preferred construction (current API, named arguments):

```kotlin
import kotlinx.io.asSink
import kotlinx.io.asSource
import kotlinx.io.buffered

val transport = StdioServerTransport(
    input = System.`in`.asSource().buffered(),
    output = System.out.asSink().buffered(),
)
```

- `StdioServerTransport(inputStream = …, outputStream = …)` and the **positional two-argument call** `StdioServerTransport(a, b)` resolve to the constructor **deprecated in 0.13.0**. It compiles with a warning. The fix is to use named `input =` / `output =`, optionally with a builder block `{ scope = … }`.
- Keeping the process alive (idiomatic, from the official `weather-stdio-server` sample):

```kotlin
runBlocking {
    val session = server.createSession(transport)
    val done = Job()
    session.onClose { done.complete() }
    done.join()
}
```

- **Pitfall:** waiting on `server.onClose { … }` instead of `session.onClose { … }`. `Server.onClose` fires only on `Server.close()`, not when stdin reaches EOF, so the process may never exit when the client disconnects.
- **Other keep-alive anti-patterns:** `while (true) { delay(...) }`, `Thread.sleep(Long.MAX_VALUE)`, `awaitCancellation()` with no EOF handling, `System.exit` inside handlers.
- `server.connect(transport)` was removed in 0.10.0 (renamed `createSession`).
- Since 0.13.0, when stdin reaches EOF the transport drains its output and then fires `onClose`.
- Since 0.13.0 there is a 16 MiB frame cap (`TooLongFrameException`). It applies to the **reading** side: incoming messages on the server, and server responses read by a Kotlin-SDK client. Very large resource contents must therefore be bounded or split, or the client may fail to read them.

## 4. Credentials in stdio mode

- Read upstream credentials, such as a GitLab token, from **environment variables** set in the client's server configuration.
- Do not pass secrets as command-line arguments: they are visible in `ps` and in shell history.
- Do not hardcode secrets or commit them in `application.conf` / `.properties`.
- Fail fast, with a clear message on **stderr**, when a required variable is missing.

## 5. Readiness for a later HTTP transport

A stdio server is "HTTP-ready" when:

- server construction (`Server(...)` plus all `add*` registrations) lives in one factory function, reused by a stdio `main` and a future HTTP `main`;
- handlers do not touch `System.in` or `System.out`, or rely on "one process = one user";
- per-user data such as tokens and permissions is not stored in global singletons. Over HTTP, many users share one process;
- configuration comes from env or a config file that works for both entry points.

See `transport-http.md` §6.
