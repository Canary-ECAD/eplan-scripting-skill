---
name: eec-pro-development
description: Develop with EPLAN EEC Pro 2026 — the mechatronic model and typicals (variants, conditions), the formula language, configuration interfaces (Form-UI, Excel/CSV/XML/IMX import), commands and C# scripting, and generation into EPLAN Electric P8, Pro Panel, PLC, Graph2D and text/Office targets, locally or through the Job Server. Use when building or debugging EEC Pro libraries, typical workbooks, configurations or generation pipelines, or decoding what an EEC-generated project did.
---

# EPLAN EEC Pro development

EEC Pro generates engineering deliverables — EPLAN Electric P8 schematics, Pro Panel
layouts, PLC programs, Graph2D documents, Word/Office files — from a **mechatronic
model plus a configuration**, instead of drawing them by hand. This skill teaches the
mental model and the workflow; the *details* (every formula function, every command,
every parameter) live in a searchable documentation index — see "Looking things up".

## The pipeline

```
mechatronic model  +  configuration values  ──►  generation  ──►  target output
(typicals, params,      (GUI, Form-UI,         (resolve          (P8 project, Pro
 conditions,             Excel/CSV/XML/IMX,     conditions,       Panel, PLC, text)
 formulas, commands)     SAP, manual entry)     insert macros)
```

1. **The mechatronic model** describes the product independent of any target system:
   functions, parameters, and the rules connecting them. Tutorials: `tutmechatronic`.
2. **Typicals** are the reusable building blocks. A typical wraps target artifacts
   (macro files, document templates) behind **variants** and **conditions** — e.g.
   "include this macro only when `CS=L` and `TNEUTRAL_R` is `TT` or `TNS`". Condition
   expressions are written in the formula language; filenames encode them as
   `_CS=L`, `Y(...)` for AND, `O(...)` for OR.
3. **Configuration** assigns concrete values to the parameters — through the GUI, a
   custom **Form-UI** (`refformui`), bulk import (`refexcel`, `refcsv`, `refxml`,
   `refimx`), or SAP (`eecsap`).
4. **Generation** evaluates the conditions against the configuration and produces
   the target artifacts — in-process, or queued on a **Job Server**
   (`eecjobserver`, `refjobserver`, `tutjobs`) for unattended/distributed runs.

For the ECAD module the output is a "typical workbook" — one sheet per mounting
location listing the page/window macros to insert, already filtered to the resolved
configuration. The full field-proven decode of that workbook (column layout, `####`
group markers, why you must *not* re-evaluate the conditions) lives in the companion
skill: `../eplan-p8/eplan-scripting/references/eec-typicals.md`.

## Where the work is

- **Formula language** (`refformulas` — the single biggest doc category, ~317 pages):
  computing parameter values, conditions, string building. Functions and operators
  are exact identifiers — look them up, never invent them.
- **Commands** (`refcommands`): the automation verbs EEC runs during generation and
  scripting (open, import, assign, generate, export…).
- **Scripting** (`refscripting` + `eecscripting`): custom logic inside EEC Pro.
- **Form-UI** (`refformui`): building the configuration dialogs users fill in.
- **Administration** (`admin` — ~304 pages): installation, system configuration,
  VMArgs (EEC Pro runs on a JVM — startup tuning lives here).
- **Per-target modules**: `eececad` (EPLAN P8), `eecpropanel`, `eecplc` (incl. the
  Step7 workflow), `eecgraph2d`, `eectext`/`reftextgen` (document generation to
  Word), `eecoffice`, `eecsap`.

## Looking things up — never guess identifiers

EEC Pro identifiers (formula functions, commands, parameters, VMArgs) are exact and
many are documented only in the official help. The index source lives in
[eplan-cloudflare-rags](https://github.com/Canary-ECAD/eplan-cloudflare-rags/tree/main/cloudflare-rag-eecpro).
Resolve identifiers against the doc index:

1. **`eecpro-rag` MCP** (if configured — this repo ships it in `.mcp.json`):
   `eecpro_search(query, topK, category?)` — semantic search over the full EEC Pro
   2026 documentation (~6,760 vectors, 36 categories). Pass `category` to scope the
   search — the category map is in `references/doc-categories.md`. `eecpro_stats`
   reports index health.
2. **Same index over REST**, no MCP needed:

   ```bash
   curl -X POST https://rageecpro.covaga.xyz/search -H "Content-Type: application/json" \
        -d '{"query": "formula function to read a parameter value", "topK": 5, "category": "refformulas"}'
   ```

   `GET /stats` and `GET /health` are also exposed.
3. Query in **English**, prefer several narrow queries, and use the `category`
   filter whenever you know which subsystem owns the answer — the index is small
   enough (~1,648 source pages) that an unfiltered query can pull a same-named
   concept from the wrong module.

## Relationship to the P8 skill

EEC Pro's main output *is* an EPLAN Electric P8 project. The moment the task crosses
to the P8 side — inserting the generated macros, renaming pages to the workbook's
structure, scripting the EPLAN API, driving EPLAN remotely — load
`../eplan-p8/eplan-scripting/`. Typical boundary:

- Configurator/model/conditions/formulas side → this skill.
- `.elk` project, `.ema`/`.emp` macros, `Eplan.EplApi.*`, Remote Client → `eplan-scripting`.

## Golden rules

1. **A generated workbook is already condition-filtered.** Do not re-evaluate `_CS=L`
   / `Y()`/`O()` conditions against the parameters — EEC resolved them when it wrote
   the sheet. (Field-proven: `eec-typicals.md`.)
2. **Never guess a formula function, command or parameter name.** Search the index
   first; identifiers are exact and the docs are the authority.
3. **Conditions, parameters and variants are model data, not code decoration.**
   Changing them changes what generation produces — understand the mechatronic model
   before editing.
4. **Generation is a batch, not a preview.** Run it against scratch/test targets
   until the output is verified; a Job Server run is not a dry run.
5. **EEC Pro ≠ EEC One ≠ EPLAN P8 scripting.** Same vendor, different products and
   different extensibility surfaces. When a docs hit describes P8's API
   (`Eplan.EplApi.*`), that is the *target* system's API — switch skills.
