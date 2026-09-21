# Report template

Produce the report in this structure, in the language the developer uses in the conversation. Rule IDs, code and spec terms stay in English. Keep the evidence short: file:line plus at most about 5 lines of code. **Mask secrets** (first 4 characters + `…`).

```markdown
# MCP Server Audit: <project name>

| | |
|---|---|
| Date | <YYYY-MM-DD> |
| Skill reference snapshot | 2026-09-18 |
| Audit baseline | MCP 2025-11-25 (via Kotlin SDK <x.y.z>) |
| Forward-compat target | MCP 2026-07-28 (INFO only) |
| Scope | <modules/paths audited> |
| Config file | <path or "none – defaults used"> |
| Mode | Static read-only review (nothing built or executed) — or list the commands the developer asked you to run |

> ⚠️ <only if applicable> Reference snapshot older than 6 months: refresh the skill (SOURCES.md).
> ⚠️ <only if applicable> SDK version newer than the snapshot knows: findings may be outdated.

## 1. Server profile
- **Entry point(s):** `<file>` (stdio) / `<file>` (HTTP)
- **SDK / Kotlin / Ktor / JVM:** …
- **Declared capabilities:** resources{listChanged=…, subscribe=…}, prompts{…}, tools{…}, logging, completions
- **Inventory:** N resources, N templates (`uriTemplate`s…), N prompts (names…), N tools (names…)
- **Transports:** current … / planned …
- **Upstream systems & credentials:** e.g. GitLab REST via `GITLAB_TOKEN` env
- **Logging backend on runtime classpath:** …
- **Tests:** …

## 2. Summary
| Severity | Count |
|---|---|
| CRITICAL | n |
| HIGH | n |
| MEDIUM | n |
| LOW | n |
| INFO | n |

**Top priorities:** 1–3 sentences naming the most important IDs.

## 3. Findings
<!-- Sort by severity, then category. One block per finding. -->

### [SEVERITY] RULE-ID: short title
- **Where:** `path/File.kt:123` (+ other locations)
- **Evidence:**
  ```kotlin
  // ≤ ~5 relevant lines
  ```
- **Why it matters:** 1–3 sentences, cite the spec (MUST/SHOULD) or reference section.
- **Proposed fix:** concrete change (short snippet or description). Note any breaking impact on clients.
- **Effort:** S / M / L

## 4. Passed checks
Compact list of rule IDs verified OK (e.g. `STDIO-04, STDIO-05, RES-02, …`).

## 5. Not applicable
Rule IDs skipped and why (e.g. `TOOL-*: no tools`, `AUTH-*: stdio only`).

## 6. Not verifiable statically
Items that need running the server or knowing the deployment (e.g. real classpath logging config, reverse-proxy TLS), with the command from build-and-run.md that would verify them.

## 7. Forward compatibility (MCP 2026-07-28)
FUT-* observations, grouped; no action required now.

## 8. Accepted exceptions (from config)
Rule ID, justification, expiry. Flag expired exceptions as findings again.

## 9. Next steps
1. Reply with the rule IDs you want fixed (e.g. "fix STDIO-01, ERR-01, RES-02"), or "fix all CRITICAL/HIGH".
2. After fixes: commands from build-and-run.md §6.
```

Rules for writing findings:

- **One finding per rule per root cause.** If the same defect appears in many places, list the locations in one finding.
- If a check depends on something outside the code, such as the deployment, put it in §6. Do not guess.
- Every "Why it matters" statement must be traceable to a reference file. If no reference supports a claim, do not make it.
