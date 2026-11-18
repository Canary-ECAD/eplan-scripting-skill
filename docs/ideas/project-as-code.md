# The project as code ("Nuna")

Was issue bochica#4 ("Nuna: infrastructure as code. Any linter finds a fail, it gets solved.").
Planned to follow product-macros, whose rules file and linter are its first piece.

## The idea

In the way of Terraformer and Terraform: read an existing EPLAN project into a text file, lint
it, show a plan (the difference between the file and the project) and apply it.

The file holds the structure, the devices with their part numbers, the PLC I/O and the logical
connections. Page layout is not declared in it.

## Reading

- **Live**, through eplan-mcp: `eplan_live_query_functions`, `eplan_live_read_connections`,
  `eplan_live_plc_io_list`, `eplan_live_read_terminals`, `eplan_export_parts_list`, the
  structure from the page and function properties.
- **Without EPLAN**, from an `.epdz`: Canary-ECAD/ecad-view's `e3d-core` (TypeScript, no UI)
  already reads its `manifest.db` and AutomationML (designations, part numbers, eCl@ss,
  positions, bridges). It would come in as a pinned dependency.

## What does not fit

EPLAN is neither declarative nor idempotent, and page layout is not declared. So a first apply
only adds and updates, never deletes, and works on a scratch clone until the user approves the
real project.

## As a skill

The skill would hold the file format, the lint rules (the product-macros linter is the first
set), and the plan/apply order with its guardrails: backup, approved batches, read back, stop on
a difference. Reading, diffing and applying are tools or scripts, not model work.
