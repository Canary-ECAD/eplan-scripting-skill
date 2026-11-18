# Macros by product number

Was Bochica's S16 and issue bochica#2 ("PLC generator: organise and create control schemes with
any SKU"). Paused there on 2026-11-12 as the most complex item; the API spike was done.

## The idea

A user brings a macro project for a product family. Its macros are drawn, but nothing says
which ones a given product needs. With the agent, approving each step, they write the rules:
how the product number reads (segments and their values), and which macro, variant and value
set each value takes. The macro project is checked against the rules, and its gaps are fixed in
approved batches. Then a concrete product number builds its schematic in a project, the same
way every time, and the result is checked.

| Piece | What it is | Who does it |
|---|---|---|
| Product number | Segments (`MS-400-7K5-B`: family, voltage, power, option), each with its values | The user with the agent; a rules file |
| Rules | For each value (or combination), the macro, its variant and the value set it takes | The same rules file |
| Linter | The macro project against the rules: missing, unused, contradictory, by severity | A tool; the same report each time |
| Fixes | Gaps in the macro project, in batches the user approves, after a backup | A tool; the agent proposes |
| Generation | One product number → a plan → the pages in a project | A tool; deterministic |

Made-up example family (for tests and demos):

```text
MS-400-7K5-B
│  │   │   └─ option:  B = brake, N = none
│  │   └───── power:   2K2, 4K0, 7K5 (kW)
│  └───────── voltage: 230, 400 (V)
└──────────── family:  MS (motor starter)
```

Its "dirty" macro project has a window macro per power size, the voltage as value sets on
placeholder objects, the brake as a macro variant, and nothing that ties them to the code.

## API findings (spike, read with `eplan_api_describe`)

Everything needed is public API:

- **Macro boxes** (`Eplan.EplApi.DataModel.MacroBox`): `MacroName` (the macro file), `Variant`,
  `RepresentationType`, `TypeOfUsage`, `InsertionPoint`, `ReferencedPlacements`, `Properties`;
  `Create(page)` and `AddReference` make one. They tell EPLAN which objects of the macro project
  form which macro and variant.
- **Placeholder objects** (`Graphics.PlaceHolder`): value sets are **records** in the API
  (`GetRecordNames`, `AddRecord`, `DeleteRecord`), each with **variables** (`GetVariableNames`,
  `AddVariable`, `GetValue`, `SetValue`). `ApplyRecord(name)` is EPLAN's "Assign value set".
  `IsVariableValid` and `DeleteUnusedVariables` help a linter. Properties carry
  `PLACEHOLDER_NAME`, `_DESCRIPTION`, `_VALUESET_NAME`, `_LASTUSEDVALUESET`.
- **Window macros** (`MasterData.WindowMacro`): `Open(file, project, representationType, variant)`,
  `GetVariants(representationType)`, `ExistVariant`, `PlaceHolders`, `Objects`, `Size`;
  `Create(file, representationType, variant, placements, description)` writes a macro file.
- **Inserting** (`HEServices.Insert`): `WindowMacro(macro or .ema, representationType, variant,
  page, point, moveKind, numerationMode)`, `PageMacro(.emp, insertAfterPage, project, overwrite,
  numerationMode)`, `SymbolMacro`. Insertion takes a macro **file**, so generation needs `.ema` /
  `.emp` files: generated from the macro project into a folder the user picks, or the ones the
  macro boxes already name.

Still open from the spike: reading a whole macro project (every box and placeholder) in one call
and how long it takes; inserting a window macro with a chosen variant and then applying a value
set, on a scratch project; where `eplan_generate_macros` writes.

## What exists in eplan-mcp

`eplan_generate_macros`, `eplan_prepare_macros`, `eplan_update_macros`, `eplan_live_insert_macro`,
`eplan_live_batch` (op `macro`), `eplan_set_property` / `eplan_set_project_property`,
`eplan_live_read_page`, `eplan_live_verify_page`, `eplan_live_check_page`, `eplan_check_project`,
the convention profiles (`eplan_profile_*`), `eplan_live_objects_read` / `_edit`, and the script
host's library scripts for macro boxes (`macrobox_snapshot`, `macrobox_restore`, `macrobox_links`,
`macro_prepare_inspect`). Missing: reading placeholders and value sets, and inserting a macro
with a chosen variant and value set. Those are script-host library scripts or new tools.

## As a skill

A `product-macros` skill would hold:

- the order: read the macro project, agree the code with the user, write the rules file, lint,
  fix in batches, generate one sample number, check it;
- the rules file format (one JSON per family, `<family>.product.json`: segments, values with a
  label per language, rules as a `when` of segment values that must all match, and what each
  takes: macro, variant, value set, page, order; plus the project properties the number fills);
- the linter's checks, by severity: a rule names a macro, variant or value set that does not
  exist (serious); a segment value no rule covers (serious); two matching rules that disagree
  (serious); a placeholder value set with an empty variable (medium); a macro no rule uses, a
  macro box without a description (minor);
- the guardrails: the macro project changes only through fixes, after a backup; generation
  writes only into the project the user names, adds pages and never deletes; nothing goes into
  the master data folders; sheet sizes and plot frames are the user's;
- what never to do: place macros by hand instead of generating, change the macro project
  outside a batch, pick sheet formats.

The deterministic parts (decode a number, build the plan, lint, generate) belong in scripts or
tools, not in the model. Later: EEC Pro as another generator; rules imported from a spreadsheet
the user already keeps.
