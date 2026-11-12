---
name: eplan-scripting
description: Develop EPLAN Electric P8 scripts, API extensions, and remote-control applications. Use when writing C# scripts for EPLAN (actions, event handlers, ribbon), accessing the EPLAN API (parts database, projects, pages), building external apps that drive EPLAN via Remote Client, or debugging EPLAN automation issues (blocking, threading, dispose). Covers EPLAN 2022–2027.
---

# EPLAN Electric P8 Development

Comprehensive guide for developing with EPLAN Electric P8: scripting (C#), the EPLAN API, and remote automation. Distilled from working production code and curated examples.

## The three development models

| Model | Runs | License | Use for |
|---|---|---|---|
| **Scripting** | Inside EPLAN (compiled on load, C# subset) | None extra | Automating actions, UI additions (ribbon), event hooks, file exports |
| **API extension** | Inside EPLAN (compiled DLL) or offline app | API license | Deep data access: parts DB, project object model, pages, properties |
| **Remote Client** | External process (WPF/console) driving a running EPLAN | RemoteClient DLLs | Orchestration apps, headless build pipelines, Cogineer generation |

**Scripts can reach the ENTIRE `Eplan.EplApi.*` surface — no API licence, no add-in.** The restriction is compile-time only: `using` works for a fixed set (`Base`, `ApplicationFramework`, `Gui`, `MasterData`, `Scripting`), anything else is CS0234 — but every namespace is reachable at runtime through reflection. Measured on 2025.0.3: 26 namespaces, 606 public types, and all 6 non-preloaded `Eplan.EplApi.*.dll` load on demand with zero refusals. Never conclude a namespace is unavailable because its `using` fails — see `references/api-data-access.md`.

## Reference files — read the one that matches the task

- **`references/script-basics.md`** — Script structure, entry-point attributes (`[Start]`, `[DeclareAction]`, `[DeclareEventHandler]`, `[DeclareRegister]`), deployment, scripting limitations.
- **`references/actions-reference.md`** — Executing actions with `CommandLineInterpreter` + `ActionCallingContext`; catalog of common actions (backup, export PDF, reports, labels, edit, selectionset…) with their parameters.
- **`references/core-classes.md`** — `Progress`, `Decider`, `PathMap` variables, `Settings`, `MultiLangString`, ribbon/context menus, `QuietModeStep`, system messages (`BaseException`, `SysMessagesCollection`).
- **`references/api-data-access.md`** — Parts database (`MDPartsManagement`), reading *and writing* part properties (`AddPart`/`RemovePart`, the `MDPropertyValue` constructor that doesn't exist, the `AmbiguousMatchException` every name-based property lookup hits), user-defined properties, multilanguage string parsing, resolving `$(MD_DOCUMENTS)`-style paths. Also: why `using Eplan.EplApi.DataModel;`/`...HEServices;` don't compile in scripts (CS0234) and how to reach that object model anyway. Also: `SymbolLibrary`/`Symbol` enumeration — why walking `Symbol(lib, int)` by index must `continue`, never `break`, on a construction miss (SymbolIds are sparse, not contiguous), and why a symbol's real identity comes from `Symbol.Properties.SYMB_DESC`/`FUNC_CATEGORY`, never from its short name or IEC-letter prefix.
- **`references/parts-database.md`** — The parts database in bulk: property numbers and API names (22009 `ARTICLE_NOTE` is the dialog's "Description", not `ARTICLE_DESCR1`), the `XPamExportXml` format (empty properties left out, `xx_XX@text;` multilanguage values, `$(MD_MACROS)` paths), import modes 0–3 and how a mode 1 update leaves everything else alone, the `Remove*` methods that delete with no undo, and the Data Portal: the add-in's actions per version (`EDPCloudAddPartsToBasketAction`, `PARTNR<i>` + `PARTID<i>`), what a download writes and skips, and the documented web API.
- **`references/eec-typicals.md`** — Generating a project from an EEC One “typical” workbook without EEC: reading the sheet (it is already condition-filtered), inserting page/window macros through `Insert`, renaming pages to the sheet's structure via `PagePropertyList`, and the page-index collision that bites on the second cabinet. (For the EEC Pro side — the mechatronic model, conditions, formula language — load the `eec-pro-development` skill.)
- **`references/e3d-installation-spaces.md`** — The version-proof runtime-reflection recipe for reaching `Eplan.EplApi.DataModel`/`HEServices` from a script (`LockingStep`, `FindType()` assembly scanning, EPLAN 2027's `...Netu`-suffixed assemblies), applied to creating 3D installation spaces and inserting window macros headlessly.
- **`references/remoting.md`** — `EplanRemoteClient`: server discovery, dynamic ports, headless launch, version gotchas (2023 vs 2025), the .NET runtime and API assembly names per version (2026 vs 2027), which addresses the port binds, executing actions and scripts remotely, Cogineer generation from Excel.
- **`references/pitfalls.md`** — CRITICAL: the command-blocking issue (message loop / monitor thread), `using`/`Dispose` discipline, sequential execution model, error-handling rules, and the compile errors that masquerade as timeouts, including which C# each version's script engine actually accepts (#9). Also: closing and opening a specific project without touching the user's (#11), and why an older project "does not match any existing software version" until it is upgraded (#12).
- **`references/integration-patterns.md`** — Connecting EPLAN to the outside: HTTP servers, SignalR real-time messaging, forwarding EPLAN system errors to external services.

## When you don't know something: look it up, never guess

Action names, action parameters and API signatures are exact, case-sensitive
identifiers, and many are undocumented in the manual. **Resolve them against a
source before writing them.** In order of preference:

1. **Ask EPLAN itself.** If the session can reach a running EPLAN — through an
   MCP server (e.g. the `eplan` action server from
   [eplan-mcp](https://github.com/Canary-ECAD/eplan-mcp)), a Remote Client,
   or a script you can execute — introspection
   beats any document: enumerate the registered actions and their parameters,
   and reflect over `Eplan.EplApi.*` to confirm a member exists before calling
   it. See `references/actions-reference.md` and `references/api-data-access.md`.
2. **Search the EPLAN documentation** with whatever docs-search tool the host
   provides. If there is none, two public endpoints index the EPLAN P8 help and
   need no auth (they are one deployment of this idea, not a dependency of this
   skill — any equivalent index works). They are also exposed as the MCP servers
   `eplan-rag` (2026, semantic) and `eplan-wiki-2027` (2027, keyword) — both
   preconfigured in this repo's `.mcp.json`. Their source and deployment
   instructions live in [eplan-cloudflare-rags](https://github.com/Canary-ECAD/eplan-cloudflare-rags):

   ```bash
   # Keyword / full-text (SQLite FTS5 + bm25), EPLAN 2027 docs.
   # Try this FIRST for anything with a real or guessable exact name -- an
   # action name, a class/method/property, an error code. Measured
   # head-to-head on real queries: FTS5 wins that case, because it doesn't
   # fragment a page into disconnected chunks the way an embedding index does.
   curl -X POST https://rag2027.covaga.xyz/search      -H "Content-Type: application/json"      -d '{"query": "FindAction", "topK": 5}'

   # Semantic (Vectorize + bge-base, ~57k vectors), EPLAN 2026 docs.
   # Use when the query shares no real vocabulary with the docs at all --
   # e.g. describing a UI behavior without knowing what it is called.
   curl -X POST https://rag2026.covaga.xyz/search      -H "Content-Type: application/json"      -d '{"query": "export project to PDF parameters", "topK": 5}'
   ```

   Both expose the same shape: `POST /search` with `{"query": "...", "topK": N}`,
   plus `GET /stats` and `GET /health`.
3. **The official help**, `https://www.eplan.help/`, when you need prose rather
   than an identifier.

Query in English, and prefer several narrow queries over one broad one
("action to renumber devices", "PagePropertyList page type values").

## Golden rules (violating these causes real production failures)

1. **EPLAN actions are pseudo-asynchronous.** After `oCLI.Execute(...)` in an external/long-running context, code can hang forever without an active message loop. See `references/pitfalls.md` before writing any multi-step automation.
2. **Dispose everything.** `ActionCallingContext`, `EplanRemoteClient`, temp clients — wrap in `using` or dispose in `finally`.
3. **Operations are sequential.** Each EPLAN action must fully finish before the next; never fire actions in parallel against one instance.
4. **Never use empty `catch {}`.** Log with `BaseException(msg, MessageLevel.X).FixMessage()` (inside EPLAN) or a logger (outside).
5. **EPLAN 2025 remoting requires "Remote Client Access"** enabled in EPLAN options; transport is gRPC (default port 49152, dynamic). In 2023 it was on by default.
6. **Match the runtime of the EPLAN you load into.** EPLAN 2026 and older run on **.NET Framework 4.8.1** and ship the managed API as `Eplan.EplApi.*u.dll`; EPLAN 2027 runs on **.NET 8** and ships it as `Eplan.EplApi.*Netu.dll` (its `*u.dll` are native twins - never reference them). A .NET 8 add-in or host cannot load in 2026 at all (`TypeLoadException: System.Runtime.Loader.AssemblyLoadContext ... mscorlib 4.0`, measured live on 2026.0.3), however cleanly it compiles against 2026's DLLs. Pick the reference set from the `Bin` folder: `*Netu.dll` when any exist, else `*u.dll`. See `references/remoting.md`.
7. **Verify action names/parameters before use** — many are undocumented and all are case-sensitive. Ask a live EPLAN, or search the docs; never write one from memory.
8. **Never `using Eplan.EplApi.DataModel;`/`...HEServices;` in a script** — that statement doesn't compile in EPLAN's script engine (CS0234, a fixed assembly set). Reach that object model via runtime reflection instead, and never hardcode the assembly name: on 2027 the managed assemblies are `...DataModelNetu`/`HEServicesNetu` and the un-suffixed names are the mixed-mode *native* twins, so a hardcoded old name throws `BadImageFormatException` there — but this is **not** a clean version cutoff: measured on 2025.0.3 the loaded managed assembly *is* `Eplan.EplApi.DataModelu` and nothing refuses to load. Resolve the type by scanning `AppDomain.CurrentDomain.GetAssemblies()` first, and only then try candidate names. See `references/e3d-installation-spaces.md`.
9. **Never `RegisterScript` a one-shot `[Start]` script.** That's for installing persistent hooks (`[DeclareAction]`/`[DeclareEventHandler]`/`[DeclareRegister]`); a `[Start]`-only script has none, so registering it first just produces a spurious EPLAN warning and wastes two remote-API round-trips. Call `ExecuteScript` alone. See `references/pitfalls.md` #10.
10. **Write generated scripts to the oldest C# your fleet needs — on 2026 that is C# 5: no `?.`, no `$"..."`, no `{ ["k"] = v }`, no `nameof`.** A compile error is *silent*: `ExecuteScript` returns success, the script never runs, and a caller waiting on its output just sees a timeout — so it gets misread as a hung or blocked EPLAN. **On any script timeout, read EPLAN's system-message tree for the `CS####` line before suspecting anything else.** (2027's engine accepts more than 2026's, so probe rather than assume either way.) See `references/pitfalls.md` #9.
11. **Address projects by path, never by GUI focus.** The "current project" is whatever the user last clicked, opening a project does not necessarily move focus to it, and `XPrjActionProjectClose` closes what the GUI has *selected* - with a navigator multi-selection it closed a real project together with the scratch clone it was meant to close (measured live on 2026.0.3). Resolve the project object from `ProjectManager.OpenProjects` by `ProjectLinkFilePath`, act on that object (`Project.Close()` to close it), and compare the open-project list before and after. See `references/pitfalls.md` #11.
