# Eyes: PDF to schematic

Was Bochica's S17 (outline) and issue bochica#3 ("Primitive tools - drawing"). The wish: that
Claude, or any model, can take a schematic PDF and redraw it in EPLAN, with EPDZ projects as
guidance for its "eyes".

## What exists

- The live drawing primitives in eplan-mcp (Canary-ECAD/eplan-mcp#34): create pages, place
  symbols, devices, terminals, PLC points, corners and T-nodes, connect pins, read back
  (`eplan_live_*`), plus `eplan_live_render_page` and page exports to PDF and graphics files.
- `eplan_pdf_inspect` reads a PDF without EPLAN.
- Hosts such as Claude Code and Claude Cowork already pass images and PDFs to the model, which
  was the missing piece in Bochica.

## The approach

Vision alone guesses positions. Give the model **exact positions** from the page itself, as text
with coordinates:

- a PDF exported from a CAD tool is vector: its texts (device tags such as `-K1`, pin numbers),
  lines and boxes come out with exact coordinates, no OCR needed;
- a scanned PDF or a photo needs OCR. Windows has an offline OCR engine that returns each word
  with its box, so nothing has to be bundled;
- symbol shapes (a relay coil, a contact) are found by the vision model, anchored to those texts
  and lines, and compared against the EPLAN symbol library (`eplan_live_symbol_catalog`, exact
  catalog lookup; do not trust a symbol's short name).

Loop: extract the page (texts, lines, symbols with positions) → map to EPLAN symbols → draw with
the primitives on a scratch clone → render the page → compare with the source → correct.

## Measuring it

Made-up sample projects (EPDZ) exported to PDF give pairs with a known answer: redraw the PDF
and compare devices and connections with `eplan_live_read_page` against the original. Only
made-up projects: the repositories are public.

## Open questions

- What does an EPDZ give as "eyes": the reference structure to learn from, or test cases?
- Scope of a first version: one page, one symbol library?
- Does the extraction belong in eplan-mcp (a tool returning texts and lines with coordinates)
  or in the skill as a script the host runs?
