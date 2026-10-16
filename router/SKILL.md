---
name: router
description: >
  Routes any EPLAN-related request to the right skill and documentation index:
  detects the product (EPLAN Electric P8 vs EPLAN EEC Pro) and the task, then reads
  the matching skill before acting. Use for EPLAN C# scripting, the EPLAN API,
  actions, Remote Client automation, macros, parts databases; for EEC Pro typicals,
  mechatronic models, the formula language, configurations and generation pipelines;
  and to pick the right doc index (P8 2026 semantic, P8 2027 keyword, EEC Pro
  semantic). Start here when unsure which EPLAN skill applies.
---

# Router — EPLAN skill dispatcher

Entry point for EPLAN work. It fingerprints the product, classifies the task, and
names the **minimal** set of skills to read before acting. It dispatches — it does
not re-teach the APIs.

## When to use

- At the **start of any EPLAN request** — writing or debugging a script, API code,
  a Remote Client app, an EEC Pro typical/configuration/generation pipeline — to
  decide which skill(s) and which doc index apply.
- When the user says "EPLAN" without specifying the product.

**When *not* to use:** once the right skill is loaded and the task sits squarely in
it, work from that skill. Re-route only when the task pivots to the other product.

## 1. Product detection

| Product | File signals | Phrase signals | Skill |
|---|---|---|---|
| **EPLAN Electric P8** | `.elk`, `.ema`, `.emp`, `.edz`, `.zw*`, `*.cs` with `[Start]`/`[DeclareAction]`, `Eplan.EplApi.*` | script, action, API, Remote Client, parts database, pages, macros, symbols, ribbon, Cogineer | `skills/eplan-p8/eplan-scripting` |
| **EPLAN EEC Pro** | `Typical *.xlsm` workbooks, EEC project/model dirs | typical, mechatronic model, configuration, formula, variant, condition, generator, Job Server, Form-UI | `skills/eec-pro/eec-pro-development` |

Ambiguous: an EEC Pro *generation* task that produces P8 output uses **both** —
eec-pro-development for the model/conditions side, eplan-scripting for the
resulting macros/pages/API side. The boundary is spelled out in the EEC skill.

## 2. Task → skill

- **Write/fix C# for EPLAN P8** (script, add-in, actions, events, ribbon, context
  menus) → `eplan-scripting` → its `references/script-basics.md`,
  `actions-reference.md`, `core-classes.md`.
- **Read/write EPLAN data** (parts DB, project model, pages, properties, symbols)
  → `eplan-scripting` → `references/api-data-access.md`.
- **Drive EPLAN from outside** (Remote Client, headless, ports, Cogineer) →
  `eplan-scripting` → `references/remoting.md`.
- **Decode/generate a typical workbook into a P8 project** → `eplan-scripting` →
  `references/eec-typicals.md` (the insertion side) + `eec-pro-development` (the
  model/conditions side).
- **Hangs, timeouts, weird behavior in EPLAN automation** → `eplan-scripting` →
  `references/pitfalls.md` first, always.
- **EEC Pro model/typical/formula/config/generation work** → `eec-pro-development`
  → `references/doc-categories.md` for which doc category to query.
- **"Which doc index do I search?"** → §3.

## 3. Doc-index selection

Three public indexes, three different strengths — pick deliberately:

| Index | Product / version | Search mode | Use it for |
|---|---|---|---|
| `eecpro-rag` · `rageecpro.covaga.xyz` | EEC Pro 2026 | Semantic (bge) | All EEC Pro docs; filter with `category` (36 of them) |
| `eplan-rag` · `rag2026.covaga.xyz` | P8 docs 2026 | Semantic (bge, ~57k vectors) | P8 questions phrased without exact identifiers |
| `eplan-wiki-2027` · `rag2027.covaga.xyz` | P8 docs 2027 | Keyword/FTS5 + bm25 | Exact names — actions, classes, methods, error codes |

The three remote indexes are maintained in [eplan-cloudflare-rags](https://github.com/Canary-ECAD/eplan-cloudflare-rags).

Measured head-to-head: FTS5 wins exact-name lookups; semantic wins when the query
shares no vocabulary with the docs. When both P8 indexes are available and you know
the identifier, hit 2027 first.

All expose `POST /search {"query", "topK"}` + `GET /stats`/`/health` over REST, and
the same content over MCP when the `.mcp.json` servers are configured. The local
`eplan` action server (from `eplan-mcp`, not auto-installed) can additionally
*execute* actions/scripts on a running EPLAN and introspect the API live — when it
is connected, prefer live introspection over any doc index.
