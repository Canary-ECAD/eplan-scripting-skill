# EPLAN agent skills — for EPLAN Electric P8 & EEC Pro

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Agent-agnostic [skills](https://agentskills.io) for EPLAN development — install
once, and the **router** loads the right skill for whatever you're automating:
**EPLAN Electric P8** (C# scripting, the EPLAN API, Remote Client) or **EPLAN EEC
Pro** (typicals, the mechatronic model, the formula language, generation).

Teaches your agent the traps that only show up in production — silent compile
failures, the command-blocking issue, dispose discipline, the CS0234 namespaces
that are reachable anyway — plus how to drive EEC Pro's model → configuration →
generation pipeline. Distilled from working code and live testing against a real
installation, not from the manual alone.

---

## Install

### Any agent (skills CLI)

The [`skills` CLI](https://github.com/vercel-labs/skills) installs into 60+
agents (Claude Code, Cursor, Codex, Copilot, Windsurf, …):

```bash
npx skills add Canary-ECAD/eplan-scripting-skill
```

### As a Claude Code plugin

The plugins are published through the single Canary E-cad marketplace,
`eplan-tools`, which lives in [eplan-mcp](https://github.com/Canary-ECAD/eplan-mcp):

```
/plugin marketplace add Canary-ECAD/eplan-mcp
/plugin install eplan@eplan-tools            # router + all skills + doc RAGs
/plugin install eplan-scripting@eplan-tools  # Electric P8 only
/plugin install eec-pro@eplan-tools          # EEC Pro only
```

Update later with `/plugin marketplace update eplan-tools`.

> Coming from the old `eplan-skills` marketplace or the `eplan-development` skill?
> Remove them (`/plugin marketplace remove eplan-skills`) and install from
> `eplan-tools` as above. The P8 skill is now called `eplan-scripting`.

### Manually

```bash
git clone https://github.com/Canary-ECAD/eplan-scripting-skill
```

```powershell
# Windows - personal skills, available in every project.
# ~/.agents/skills is the universal dir most agents read; use ~/.claude/skills for Claude Code.
xcopy /E /I eplan-scripting-skill\skills\eplan-p8\eplan-scripting "$env:USERPROFILE\.agents\skills\eplan-scripting"
xcopy /E /I eplan-scripting-skill\skills\eec-pro\eec-pro-development "$env:USERPROFILE\.agents\skills\eec-pro-development"
```

```bash
# macOS / Linux — universal dir (Amp, Codex, Cline, …). Use ~/.claude/skills for Claude Code.
cp -r eplan-scripting-skill/skills/eplan-p8/eplan-scripting ~/.agents/skills/
cp -r eplan-scripting-skill/skills/eec-pro/eec-pro-development ~/.agents/skills/
cp -r eplan-scripting-skill/router ~/.agents/skills/router
```

For a single project, copy into `<your-project>/.agents/skills/` (or
`.claude/skills/`) instead.

---

## The MCP servers

The skills are **host-agnostic** — useful on their own. Paired with the remote
[documentation RAGs](https://github.com/Canary-ECAD/eplan-cloudflare-rags), the agent can
*query the EPLAN documentation*. The local action server in
[eplan-mcp](https://github.com/Canary-ECAD/eplan-mcp) can *drive a running EPLAN*.

This repo ships a [`.mcp.json`](.mcp.json) with the three **remote doc RAGs**
from [eplan-cloudflare-rags](https://github.com/Canary-ECAD/eplan-cloudflare-rags)
(already deployed, no local data needed):

| Server | Endpoint | Covers | Search |
|---|---|---|---|
| `eplan-rag` | `rag2026.covaga.xyz/mcp` | EPLAN P8 docs 2026 | Semantic |
| `eplan-wiki-2027` | `rag2027.covaga.xyz/mcp` | EPLAN P8 docs 2027 | Keyword (FTS5) |
| `eecpro-rag` | `rageecpro.covaga.xyz/mcp` | EEC Pro 2026 docs | Semantic, 36 categories |

Or register them by hand:

```bash
claude mcp add eplan-rag       --transport http https://rag2026.covaga.xyz/mcp
claude mcp add eplan-wiki-2027 --transport http https://rag2027.covaga.xyz/mcp
claude mcp add eecpro-rag      --transport http https://rageecpro.covaga.xyz/mcp
```

(Older setups: `claude mcp add eecpro-rag -- npx mcp-remote https://rageecpro.covaga.xyz/mcp`,
with `cmd /c` before `npx` on Windows.)

The **local action server** (`eplan`, ~200 tools: run actions/scripts inside a
running EPLAN, introspect the API live) installs separately — it needs Python +
`pythonnet` on the EPLAN machine. See
[eplan-mcp](https://github.com/Canary-ECAD/eplan-mcp#local-eplan-automation-p8).

---

## What's inside

```
eplan-scripting-skill/
├── .mcp.json                   # the three remote doc-RAG MCP servers
├── router/
│   └── SKILL.md                # detects P8 vs EEC Pro + task → loads the right skill
└── skills/
    ├── eplan-p8/
    │   └── eplan-scripting/
    │       ├── SKILL.md        # the three dev models, lookup order, golden rules
    │       └── references/     # script-basics, actions, core-classes,
    │                           # api-data-access, e3d-installation-spaces,
    │                           # eec-typicals, remoting, pitfalls, integration
    └── eec-pro/
        └── eec-pro-development/
            ├── SKILL.md        # model → configure → generate pipeline, golden rules
            └── references/
                └── doc-categories.md   # the 36-category map of the EEC Pro doc index
```

Each `SKILL.md` is the entry point; the agent pulls in only the reference file
that matches the task, so it costs little context until needed.

## Coverage

- **EPLAN Electric P8 2022–2027** — with the version-specific notes called out:
  ribbon API since 2022, remoting defaults 2023 vs "Remote Client Access" + gRPC
  in 2025, the `...Netu` assemblies in 2027, .NET Framework 4.8.1 targeting.
- **EPLAN EEC Pro 2026** — mechatronic model, typicals/variants/conditions,
  formula language, Form-UI, import formats, scripting, commands, Job Server, and
  all generation targets (ECAD/P8, Pro Panel, PLC, Graph2D, text/Office, SAP).

## Pairs well with

- **[eplan-mcp](https://github.com/Canary-ECAD/eplan-mcp)** — the local MCP
  server that lets the agent *execute* inside a running EPLAN.
- **[eplan-cloudflare-rags](https://github.com/Canary-ECAD/eplan-cloudflare-rags)** —
  the three remote documentation RAGs configured in `.mcp.json`.
- **[eplan-ctxmenu-kit](https://github.com/covagashi/eplan-ctxmenu-kit)** — a
  worked example of the context-menu material: adding right-click entries and
  reading the row the user clicked.

## License

MIT — see [LICENSE](LICENSE).

> **Disclaimer:** this is an independent community project by [Canary E-cad](https://github.com/Canary-ECAD). It is not affiliated with, endorsed by, or supported by EPLAN GmbH & Co. KG. EPLAN, EPLAN Electric P8 and EPLAN EEC Pro are trademarks of their respective owner.
