# Build, run and verify: instructions to hand to the developer

The audit is read-only. **Give these commands to the developer. Do not run them yourself** unless the developer explicitly asks you to in the current conversation. In that case, show the exact command first and run only what was asked.

Nothing here needs internet access, except Gradle dependency resolution: that uses the organization's Maven mirror or the Gradle cache (`--offline` if the cache is warm).

Adapt the module name, main class and application name to the project. Find them in `build.gradle.kts`: the `application { mainClass… }`, `shadow` and `rootProject.name` entries.

## 1. Build and tests

```bash
./gradlew clean build            # compile + unit tests
./gradlew test --tests '*Mcp*'   # MCP-level tests only, if named that way
./gradlew build --warning-mode all   # surfaces deprecated SDK API usage (compiler warnings)
./gradlew dependencies --configuration runtimeClasspath | grep -Ei 'modelcontextprotocol|ktor|logback|log4j|slf4j'
```

The last command shows the resolved SDK version and **which logging backend is actually on the runtime classpath**, which rule STDIO-02 needs.

## 2. Produce a launchable artifact (for MCP clients)

```bash
./gradlew installDist            # → build/install/<app>/bin/<app>
# or, with the shadow plugin:
./gradlew shadowJar              # → build/libs/<app>-all.jar
```

Example client configuration (the exact format depends on the client):

```json
{ "command": "/abs/path/build/install/<app>/bin/<app>",
  "args": [],
  "env": { "GITLAB_URL": "https://gitlab.internal", "GITLAB_TOKEN": "<set in client secret store>" } }
```

Do not launch the server with `./gradlew run`: Gradle writes to stdout and startup is slow.

## 3. Manual stdio smoke test (checks protocol and stdout cleanliness, rules STDIO-01 / STDIO-03)

Prerequisite: `./gradlew installDist` (§2). Outputs go to a temporary directory, so the project tree is not modified.

```bash
APP="$PWD/build/install/<app>/bin/<app>"
OUT="$(mktemp -d)"; cd "$OUT"
( printf '%s\n' \
  '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"audit","version":"0"}}}' \
  '{"jsonrpc":"2.0","method":"notifications/initialized"}' \
  '{"jsonrpc":"2.0","id":2,"method":"resources/list","params":{}}' \
  '{"jsonrpc":"2.0","id":3,"method":"resources/templates/list","params":{}}' \
  '{"jsonrpc":"2.0","id":4,"method":"prompts/list","params":{}}' \
  '{"jsonrpc":"2.0","id":5,"method":"resources/read","params":{"uri":"does-not-exist://x"}}' ;
  sleep 5 ) | "$APP" > stdout.jsonl 2> stderr.log
echo "exit code: $?"

# Every stdout line must be a JSON-RPC message:
python3 - <<'EOF'
import json,sys
bad=0
for i,l in enumerate(open("stdout.jsonl"),1):  # run from $OUT
    try:
        m=json.loads(l); assert m.get("jsonrpc")=="2.0"
    except Exception as e:
        bad+=1; print(f"line {i} NOT JSON-RPC: {l[:120]!r}")
print("OK" if bad==0 else f"{bad} bad line(s)")
EOF
```

What to check:

- The initialize result shows `protocolVersion` and the capabilities.
- Response `id` 5 is an **error**: `-32002` at the 2025-11-25 baseline. It must not be an empty `contents`.
- The process **exits by itself** shortly after stdin closes (rule STDIO-03). If the command hangs, the keep-alive is wrong.
- `stderr.log` contains the logs, and no secrets.
- Omit the list calls for primitives the server does not declare. They return `-32601`, which is correct.

Tools, if present:

```json
{"jsonrpc":"2.0","id":6,"method":"tools/list","params":{}}
{"jsonrpc":"2.0","id":7,"method":"tools/call","params":{"name":"<tool>","arguments":{}}}
```

The second call should return `isError: true` with a helpful message when required arguments are missing.

## 4. Manual HTTP smoke test (Streamable HTTP, when it exists)

```bash
URL=http://127.0.0.1:3000/mcp
curl -si -X POST "$URL" -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"audit","version":"0"}}}'
# note the Mcp-Session-Id response header (stateful mode), then send
# notifications/initialized and further requests with headers
#   Mcp-Session-Id: <id>   MCP-Protocol-Version: 2025-11-25

# DNS-rebinding / Origin protection (rule HTTP-01) → expect 403
curl -si -X POST "$URL" -H 'Origin: http://evil.example' -H 'Content-Type: application/json' \
  -H 'Accept: application/json, text/event-stream' -d '{"jsonrpc":"2.0","id":1,"method":"ping"}'

# Unauthenticated access (rule HTTP-03 / AUTH-01) → expect 401 with WWW-Authenticate
curl -si -X POST "$URL" -H 'Content-Type: application/json' -H 'Accept: application/json, text/event-stream' \
  -d '{"jsonrpc":"2.0","id":1,"method":"initialize","params":{"protocolVersion":"2025-11-25","capabilities":{},"clientInfo":{"name":"audit","version":"0"}}}'
```

## 5. Graphical inspection (optional)

The official MCP Inspector runs through `npx @modelcontextprotocol/inspector`, so it needs the npm registry (or an internal npm mirror). If npm is not available, the manual tests above cover the essentials.

## 6. After applying fixes

1. `./gradlew clean build --warning-mode all`: the build is green, and no new deprecation warnings appear.
2. Rerun §3 (and §4 if HTTP is involved).
3. Test with the real MCP client.
4. For fixes that touch secrets (SEC-01), **rotate the exposed credential**. Removing it from code is not enough, because it stays in the git history.
