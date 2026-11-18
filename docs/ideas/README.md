# Ideas for future skills

Three EPLAN workflows that started as enhancement issues in Bochica, the desktop app that is
now archived (Claude Cowork covers the same ground). They are kept here as candidates for new
skills. Each would be a skill that drives tools the `eplan` MCP server
([eplan-mcp](https://github.com/Canary-ECAD/eplan-mcp)) already has or will get, not an app feature.

| Idea | Issue | State | Note |
|---|---|---|---|
| Macros by product number ("PLC generator") | [#4](https://github.com/Canary-ECAD/eplan-scripting-skill/issues/4) (was bochica#2) | Planned in detail, API spike done | [product-macros.md](product-macros.md) |
| Eyes: PDF to schematic ("primitive drawing") | [#5](https://github.com/Canary-ECAD/eplan-scripting-skill/issues/5) (was bochica#3) | Outline | [pdf-to-schematic.md](pdf-to-schematic.md) |
| The project as code ("Nuna") | [#6](https://github.com/Canary-ECAD/eplan-scripting-skill/issues/6) (was bochica#4) | Outline; builds on product-macros | [project-as-code.md](project-as-code.md) |

The original planning stays readable in the archived repo:
[BACKLOG.md](https://github.com/Canary-ECAD/bochica/blob/main/docs/planning/BACKLOG.md#eplan-workflows)
and [S16-product-macros.md](https://github.com/Canary-ECAD/bochica/blob/main/docs/planning/sprints/S16-product-macros.md).

## What carries over from Bochica's design

- **Pre-made tools compute, the skill guides.** A skill must not have the model place macros or
  draw pages one call at a time when a deterministic tool can do it from a plan. The skill holds
  the order of steps and the questions to ask; the tools (in eplan-mcp, or scripts for its
  script host) do the work, the same way every time.
- **Rules are data the user owns:** a file in their working folder that the agent writes with
  them and they approve.
- **Human in the loop at every write:** show the plan, take a backup, apply in approved batches,
  read back, stop on a difference.
- **Scratch first:** writes go to a scratch clone (`eplan_scratch_project_create`) until the user
  approves the real project. Never write into `$(MD_MACROS)` or other shared master data folders.
- **Made-up sample data only** in tests and demos: these repositories are public.
