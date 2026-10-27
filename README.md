# EPLAN agent skill — for EPLAN Electric P8

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)

Agent-agnostic [skills](https://agentskills.io) for **EPLAN Electric P8** development: C# scripting, the
EPLAN API and Remote Client. The **router** picks the right reference and documentation index for the task.

Teaches your agent the traps that only show up in production:
- silent compile failures;
- the command-blocking issue;
- dispose discipline;
- the CS0234 namespaces that are reachable anyway.

Distilled from working code and live testing against a real installation, not from the manual alone.

> **EPLAN EEC Pro** (mechatronic model, typicals, formulas, Job Server, generation) now has its own skill:
> **[eec-pro-skill](https://github.com/Canary-ECAD/eec-pro-skill)**. The two pair well, because EEC Pro's
> output is a P8 project, but each installs on its own.

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
/plugin install eplan@eplan-tools            # router + P8 skill + P8 doc RAGs
/plugin install eplan-scripting@eplan-tools  # Electric P8 skill only
/plugin install eec-pro@eplan-tools          # EEC Pro (from eec-pro-skill)
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
```

```bash
# macOS / Linux — universal dir (Amp, Codex, Cline, …). Use ~/.claude/skills for Claude Code.
cp -r eplan-scripting-skill/skills/eplan-p8/eplan-scripting ~/.agents/skills/
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

This repo ships a [`.mcp.json`](.mcp.json) with the two **remote P8 doc RAGs**
from [eplan-cloudflare-rags](https://github.com/Canary-ECAD/eplan-cloudflare-rags).
They are already deployed, so no local data is needed:

| Server | Endpoint | Covers | Search |
|---|---|---|---|
| `eplan-rag` | `rag2026.covaga.xyz/mcp` | EPLAN P8 docs 2026 | Semantic |
| `eplan-wiki-2027` | `rag2027.covaga.xyz/mcp` | EPLAN P8 docs 2027 | Keyword (FTS5) |

Or register them by hand:

```bash
claude mcp add eplan-rag       --transport http https://rag2026.covaga.xyz/mcp
claude mcp add eplan-wiki-2027 --transport http https://rag2027.covaga.xyz/mcp
```

The EEC Pro doc index (`eecpro-rag`) ships with [eec-pro-skill](https://github.com/Canary-ECAD/eec-pro-skill).

The **local action server** (`eplan`, ~200 tools: run actions/scripts inside a
running EPLAN, introspect the API live) installs separately — it needs Python +
`pythonnet` on the EPLAN machine. See
[eplan-mcp](https://github.com/Canary-ECAD/eplan-mcp#local-eplan-automation-p8).

---

## What's inside

```
eplan-scripting-skill/
├── .mcp.json                   # the two remote P8 doc-RAG MCP servers
├── router/
│   └── SKILL.md                # classifies the task → right reference + doc index; EEC Pro → eec-pro-skill
└── skills/
    └── eplan-p8/
        └── eplan-scripting/
            ├── SKILL.md        # the three dev models, lookup order, golden rules
            └── references/     # script-basics, actions, core-classes,
                                # api-data-access, e3d-installation-spaces,
                                # eec-typicals, remoting, pitfalls, integration
```

Each `SKILL.md` is the entry point; the agent pulls in only the reference file
that matches the task, so it costs little context until needed.

## Coverage

- **EPLAN Electric P8 2022–2027** — with the version-specific notes called out:
  ribbon API since 2022, remoting defaults 2023 vs "Remote Client Access" + gRPC
  in 2025, the `...Netu` assemblies in 2027, .NET Framework 4.8.1 targeting.

## Pairs well with

- **[eec-pro-skill](https://github.com/Canary-ECAD/eec-pro-skill)** — the EPLAN EEC Pro skill: model,
  typicals, formulas, Job Server, generation into P8.

- **[eplan-mcp](https://github.com/Canary-ECAD/eplan-mcp)** — the local MCP
  server that lets the agent *execute* inside a running EPLAN.
- **[eplan-cloudflare-rags](https://github.com/Canary-ECAD/eplan-cloudflare-rags)** —
  the remote documentation RAGs (the P8 ones are configured in `.mcp.json`).
- **[eplan-ctxmenu-kit](https://github.com/covagashi/eplan-ctxmenu-kit)** — a
  worked example of the context-menu material: adding right-click entries and
  reading the row the user clicked.

## License

MIT — see [LICENSE](LICENSE).

> **Disclaimer:** this is an independent community project by [Canary E-cad](https://github.com/Canary-ECAD). It is not affiliated with, endorsed by, or supported by EPLAN GmbH & Co. KG. EPLAN and EPLAN Electric P8 are trademarks of their respective owner.
