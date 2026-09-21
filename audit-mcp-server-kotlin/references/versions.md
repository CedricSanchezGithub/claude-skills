# Versions, baselines and snapshot freshness

**Snapshot date: 2026-09-18.** Everything in `references/` reflects the MCP specification and the Kotlin SDK as of this date. Put this date in every report header.

## 1. MCP specification revisions

| Revision | Status on snapshot date | Era |
|---|---|---|
| `2026-07-28` | Latest published revision | "Modern": stateless, no `initialize` |
| `2025-11-25` | Previous revision | "Legacy": `initialize` handshake |
| `2025-06-18` | Older | Legacy |
| `2025-03-26` | Older (introduced Streamable HTTP) | Legacy |
| `2024-11-05` | Oldest (HTTP+SSE transport) | Legacy |

`2026-07-28` is a breaking redesign. It removes `initialize`, sessions, `ping`, `logging/setLevel`, `resources/subscribe` and server-initiated requests; see `spec-2026-07-28-delta.md`. A server MAY be "dual-era", serving legacy clients through `initialize` and modern clients through per-request `_meta`.

## 2. Kotlin SDK → protocol versions

Maven group `io.modelcontextprotocol`. Artifacts:

- `kotlin-sdk` (umbrella, client + server)
- `kotlin-sdk-server`
- `kotlin-sdk-client`
- `kotlin-sdk-core` (types, transitive)
- `kotlin-sdk-testing` (in-process transports for tests, since 0.9.0)

| SDK version | Latest protocol | Supported protocol versions | Toolchain (Kotlin / Ktor) |
|---|---|---|---|
| 0.4.0 – 0.6.x | 2024-11-05 | 2024-11-05, 2024-10-07 | 2.0–2.2 / 3.0–3.2 |
| 0.7.0 – 0.7.7 | 2025-03-26 | 2025-03-26, 2024-11-05 | 2.2.0 / 3.2.3 |
| 0.8.0 – 0.9.0 | 2025-06-18 | 2025-06-18, 2025-03-26, 2024-11-05 | 2.2.21 / 3.2.3 |
| 0.10.0 – 0.11.x | 2025-11-25 | 2025-11-25, 2025-06-18, 2025-03-26, 2024-11-05 | 2.2.21–2.3.10 / 3.2.3–3.3.3 |
| 0.12.0 – 0.14.0 | 2025-11-25 | same as above | 2.3.21 / 3.3.3–3.4.3 |
| **0.15.0** (latest release, 2026-07-28) | **2025-11-25** | same as above | **2.4.0 / 3.5.1** |

Facts to rely on:

- **No released Kotlin SDK version (≤ 0.15.0) supports protocol `2026-07-28`.** The unreleased main branch had started adding 2026 types, and had deprecated per-request log levels, but `2026-07-28` is not yet in `SUPPORTED_PROTOCOL_VERSIONS`.
- Negotiation in the SDK: if the client asks for a version in `SUPPORTED_PROTOCOL_VERSIONS`, the server echoes it back. Otherwise it answers with `LATEST_PROTOCOL_VERSION`. Developers do not implement negotiation themselves.
- JVM: minimum Java 11 since SDK 0.8.0 (Java 8 before that). The samples use `jvmToolchain(17)`.
- Ktor engines are **not** transitive. An HTTP server must add `ktor-server-netty` or `ktor-server-cio` itself.
- Release notes live on GitHub Releases only; the repository has no CHANGELOG file.

## 3. How to pick the audit baseline

1. **Primary baseline = the newest protocol revision the project's SDK version can negotiate** (table above). With SDK ≥ 0.10.0 this is `2025-11-25`. Rules in `checklist.md` are written against that baseline.
2. If the SDK is older than 0.10.0, audit against the SDK's own latest protocol, and report the upgrade as rule VER-01.
3. `2026-07-28` rules are reported as **forward-compatibility findings** (`FUT-*`, severity INFO). Do not report them as violations: the SDK cannot implement them yet, so hand-rolling them would be worse than waiting. The exception is when the project deliberately targets 2026-07-28 (the config file says so, or the code handles `server/discover`). Then treat `spec-2026-07-28-delta.md` as normative.
4. If the project pins an SDK **newer than 0.15.0**, the snapshot cannot describe it. Say so explicitly in the report, audit against what is known, and recommend refreshing the skill (`SOURCES.md`).

## 4. Snapshot freshness rule

- If the current date is known and is more than **6 months** after the snapshot date, add this warning at the top of the report: "Reference snapshot is older than 6 months; newer MCP revisions or SDK releases may exist. Refresh the skill (see SOURCES.md)."
- Never try to check for newer versions online. The skill is offline by design.
- When the code uses an API, field or method that the references do not mention, report it as **"Not covered by the reference snapshot"** instead of guessing whether it is right.
