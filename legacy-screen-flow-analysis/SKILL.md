---
name: legacy-screen-flow-analysis
description: Analyzes how a screen, page, or view in a legacy mobile codebase behaves between user arrival and stable render, then produces a structured markdown report covering the lifecycle sequence, state management, data fetching, business rules, and side effects. Defaults to Flutter/Dart conventions (StatefulWidget, initState, Provider/Bloc/Riverpod) but adapts to other mobile frameworks (Android/Kotlin, iOS/Swift, React Native) when told. Use when the user wants to understand, document, reverse-engineer, or map an existing screen's behavior before refactoring, migrating, or rewriting it in a new stack or language. Do not use for writing new features, fixing unrelated bugs, or general-purpose code review.
---

# Legacy Screen Flow Analysis

Document, without modifying, what a legacy mobile screen does between user arrival and stable render. Output feeds a migration/rewrite effort, so accuracy matters more than completeness.

## Step 1 — Collect inputs

Required before starting:
- Screen name
- Entry-point file path (root widget/view/controller)
- Framework (default: Flutter/Dart; ask if ambiguous)

Optional but useful: route/navigation trigger.

If any required input is missing, ask the user instead of guessing.

## Step 2 — Check code access

- If you have direct file/repo access, use it to explore.
- If not, ask the user to paste the content of: the root widget/view, the controller/bloc/provider/viewmodel, the repository/service called on load, and the routing file.
- Never infer behavior from a file you have not actually read. Anything unverifiable goes into "Open questions" in the output, not into the analysis.

## Step 3 — Analyze

Work through each point below. Skip points that genuinely don't apply to the framework in use.

1. **Entry point** — widget/view type, defining file, how navigation reaches it (route, arguments passed).
2. **Arrival sequence** — chronological trace of everything that runs on load (constructor, `initState`/`viewDidLoad`/`onCreate`/equivalent, first render, post-frame callbacks, `FutureBuilder`/`StreamBuilder` or async loads). State what each step actually triggers, not just that it runs.
3. **State management** — pattern in use (setState, Provider, Bloc/Cubit, Riverpod, GetX, MVVM, Redux, etc.), where state initializes, what fires automatically on load.
4. **Data flow** — every call triggered on load (API, local DB, cache, key-value storage): trigger, function/file, data returned, success/error/loading handling.
5. **Business rules** — conditions that change what's displayed (status, flags, user data, feature flags, A/B).
6. **Side effects** — analytics, permission requests, auto-redirects, notifications, local writes.
7. **Dependencies** — packages, services, DI/locators involved.
8. **Gray areas** — dead code, duplication, contradictions, `TODO`/`FIXME`. Report file + function/method + line number, without speculating on intent.

## Step 4 — Write the report

Use the structure in `references/report-template.md`.

If file creation is available:
1. Create `docs/screen-flows/` if it does not exist (`mkdir -p docs/screen-flows/`).
2. Save the report as `docs/screen-flows/<screen-name>-analysis.md`.

If file creation is not available, return the markdown directly in the response.

## Rules

- Cite file + function/method + line number for every non-trivial claim.
- Never invent or extrapolate behavior. Uncertain points go in "Open questions."
- Stay descriptive, not evaluative — no judgment on code quality.
- Always write the report in **French**, regardless of the language of the request or the codebase.
