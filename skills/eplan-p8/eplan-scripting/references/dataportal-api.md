# EPLAN Data Portal web API

`https://dataportal.eplan.com/api`. OpenAPI 3.0 description at `/api/doc` (an
HTML page; the spec is the JSON inside its `<script id="swagger-data">`, there
is no `doc.json`). Answers are JSON:API (`application/vnd.api+json`): `data`,
`attributes`, `relationships`, `included`, `meta`. Everything below was checked
against the live API on **2026-11-19**; where the published spec says otherwise,
the spec is wrong.

This is one of the two supported surfaces to the Data Portal. The other is the
add-in's actions inside EPLAN (`EDPCloudAddPartsToBasketAction`, in
`parts-database.md`). The web API **finds** parts and downloads loose files;
only the add-in's download brings a part with its macros into the parts
database.

## Auth: most of it needs no token

| Needs nothing | Needs a Personal Access Token |
|---|---|
| `GET/POST /partscheck`, `GET/POST /parts`, `GET /parts/{id}` | every `/download/*` route |
| `GET /macros/{id}`, `GET /macro_variants/{id}` | `/basketitems` (the download list), `DELETE /download_list` |
| `/manufacturers`, `/eplan_catalog_entries`, `/eclass_catalog_entries` | `/configurators/parts`, `/token/adapter`, `/configurators/token` |
| `/data_types`, `/standards`, `/statistics`, `/suggestions/*` | |

Without a token a protected route answers `401 {"errors":[{"title":"Missing JWT token"}]}`.

The token header is `Authorization: Bearer PAT:<token>`. A PAT is made by a
person in their ePulse user profile, for the Data Portal specifically, and
cannot belong to the ePulse Community organisation; it expires. Keep it in an
environment variable or a secret store, never in a script, a project or a log.
(`Bearer <token>` without `PAT:` is the short-lived browser token.)

## Part number → Data Portal id

`POST /partscheck`, `Content-Type: text/plain`, **one value per line** (the
spec's example uses newlines; commas are not separators here). A value can be a
full EPLAN part number (`SIE.5SL6316-7`), a type number or an order number.

```
POST /partscheck?exact=0
SIE.5SL6316-7
9343210
NOPE.XYZ
```

```json
{"data":[
 {"type":"partscheckresult","attributes":{"search":"SIE.5SL6316-7","partid":454767,
   "partnr":"SIE.5SL6316-7","typenr":"5SL6316-7","ordernr":"5SL6316-7","data_supplier":"SIE",
   "url":"https://dataportal.eplan.com/part-details/SIE/SIE.5SL6316-7/id-454767", ...}},
 {"type":"partscheckresult","attributes":{"search":"9343210","partid":422506,"partnr":"RIT.9343210", ...}},
 {"type":"partscheckmissingresult","attributes":{"search":"NOPE.XYZ"}}]}
```

- Always 200; a miss is a `partscheckmissingresult`, not a 404 (the `GET
  /partscheck/{value}` form does 404).
- `partid` is what `EDPCloudAddPartsToBasketAction` takes as `PARTID<i>`, and
  `partnr` is the `PARTNR<i>` to pair with it (a bare order number comes back
  as `MFR.TYPE`).
- `allow_multiple_results=true` returns every match; the same hit can appear
  twice (it matched on two numbers), so de-duplicate on `(search, partid)`.
- `exact=1`: case sensitive, no trimming.
- A hit whose `data_supplier` differs from your database's manufacturer is a
  different part with the same number. A miss doesn't prove absence: some
  catalogs store numbers with different punctuation.

## Search: `GET /parts`

| Parameter | Works as |
|---|---|
| `search` | free text, max 100 characters |
| `filter[manufacturer]` | the **short name** (`SIE`, `RIT`, `PXC`), not the numeric id |
| `filter[data_types]` | comma list from `commercial_data, function_template, logic_macro, 2d_graphical_data, connection_point_pattern, drilling_pattern, 3d_graphical_data, dxf_data`. **The spec's `filter[features]` is refused** (`400 invalid-filter-type`), as are `data_type`, `standard`, `eplan_catalog` |
| `filter[catalog_eplan]` | a product-group id from `/eplan_catalog_entries` (`200070000`; the spec's 10-digit `0200070000` also works) |
| `filter[catalog_eclass]`, `filter[catalog_manufacturer]` | ECLASS id, manufacturer catalog id |
| `filter[eplan_data_standard][ge]=1` | operators `gt lt ge le eq ne` on single-value filters |
| `sort` | `part_number, order_number, type_number, download_count, manufacturer.short_name, manufacturer.long_name`; `-` prefix = descending |
| `page[offset]`, `page[limit]` | default 0 / 10; `meta.page.total` is the hit count |

Texts (`designation`, `description`) come per language:
`{"en_US": …, "de_DE": …, "fr_FR": …, "es_ES": …}`; `attributes.languages`
lists which. `relationships.data_types` says what the part carries.

`POST /parts` with a text/plain body `ID1,ID2,…` (comma separated here) returns
several parts by id.

## One part, its macros and the 3D file

`GET /parts/{id}?include=manufacturer` gives the attributes
(`graphic_macro_path`, `group_symbol_macro_path`, `eds_version`, `lifecycle`,
`identcode`, `basketable`…) and relationships: `product_top_group /
product_group / product_sub_group` (catalog ids), `eclass_level1..4`,
`graphic_macro` (3D), `group_symbol_macro` (2D), `picture_file`,
`accessories`, `dependent_parts`, `variants`, `function_templates`.

The E3D chain is three hops:

1. `GET /parts/454767` → `relationships.graphic_macro.data.id` = `19832608`
2. `GET /macros/19832608` → `attributes.name` `5SLx3xx-x_3D`,
   `relationships.macro_variants.data[].id` = `69290004`
3. `GET /macro_variants/69290004` → `representation_type` **10** (3D; the 2D
   group-symbol macro's variant is 6). Then
   `GET /download/e3d_data/69290004` (token) returns the `.e3d`.

Whether a macro is 3D is the representation type of its variant, not a name
ending in `_3D` (that is only a convention).

## Downloads (token)

| What | Route |
|---|---|
| E3D of one macro variant | `GET /download/e3d_data/{macroVariantId}` |
| DXF, one / several parts | `GET /download/dxf_data/part/{partId}` / `POST /download/dxf_data/part` |
| Commercial data CSV, one / several | `GET /download/commercial_data/part/{partId}` / `POST /download/commercial_data/part` |
| Same for the download list | `GET /download/{dxf_data\|commercial_data}/basket`; `POST …/basket` with `ITEM_ID,ITEM_ID` |

The file name is in `Content-Disposition`. Measured with a real PAT on
2026-11-19:

- E3D: `<macro>.emav<variant>r<reptype>.e3d` (`5SLx3xx-x_3D.emav0r10.e3d`,
  171 KB, binary).
- DXF of one part or one download-list item: `MAC_<macro>.zip` holding
  `dxf/<macro>/Panel layout/<macro>_A6.dxf`, `commercialdata.csv` and
  `readme.txt`; several parts come as one zip (`EDataPortalDXF.zip`).
- Commercial data, one or several parts: always `commercial_data.csv`
  (quoted header row: part number, manufacturer short/full name, type number,
  order number, product group, descriptions 1–3, long description, links), so
  name your copies yourself or the second download overwrites the first.
- A part without `3d_graphical_data` (e.g. `SIE.6ES7512-1SM03-0AB0`) has no
  `graphic_macro` at all; its `group_symbol_macro` can carry many variants
  (representation types 1, 2, … for the multi-line, overview, panel views),
  whose `description` is an EPLAN multilanguage string
  (`de_DE@…;en_US@…;`). None of this imports into the parts database: an
`.e3d` goes through the 3D import, a DXF through the DXF import.

## The download list (`/basketitems`, token)

The account's download list: the same one EPLAN's Data Portal window shows
when signed in to that account. JSON:API bodies:

```json
POST /basketitems
{"data":[{"type":"basketitems","attributes":{"count":1,"description":"…"},
          "relationships":{"part":{"data":{"id":"454767","type":"parts"}}}}]}
```

Checked live: `GET /basketitems?include=part` lists items with the part in
`included`; `POST` answers with the new item ids; removing by item id brought
the list back to what it was. `DELETE /basketitems` takes `{"data":[{"type":"basketitems","id":"12"}]}` —
the **item** id from `GET /basketitems`, not the part id. `409` = list full.
`DELETE /download_list` empties it. Removed parts still in the list are
reported under `meta.inactive_parts`.

## Reference lists

- `/suggestions/combined?search=siemens` → `manufacturer_values:
  [{"SIE":"Siemens"}, …]`: the way from a name to the short name.
- `/manufacturers` (625; `short_name`, `long_name`, `allow_dxf`),
  `/eplan_catalog_entries` (top groups `200000000` Electrical engineering,
  `300000000`, …; `?filter[catalog_eplan]=<id>` lists the children),
  `/data_types`, `/standards` (`iec`, `nfpa`, `nfpa_mm`). Counts are in
  `meta.facets_counts`, keyed by id.
- `/statistics`: `parts_count` (2.4 M), `manufacturers_count`.

## From an EPLAN script (C# 5, .NET Framework on 2026)

`System.Net` and Newtonsoft are available to scripts. Force TLS 1.2 on .NET
Framework; keep the call off long loops (it blocks EPLAN's UI thread, see
`pitfalls.md`).

```csharp
using System;
using System.Net;
using System.Text;
using Eplan.EplApi.Scripting;

public class DataPortalLookup
{
    [Start]
    public void Run()
    {
        ServicePointManager.SecurityProtocol |= SecurityProtocolType.Tls12;
        using (WebClient wc = new WebClient())
        {
            wc.Encoding = Encoding.UTF8;
            wc.Headers[HttpRequestHeader.Accept] = "application/vnd.api+json";
            wc.Headers[HttpRequestHeader.ContentType] = "text/plain";
            string json = wc.UploadString("https://dataportal.eplan.com/api/partscheck",
                                          "SIE.5SL6316-7\nRIT.9343210");
            Newtonsoft.Json.Linq.JObject res = Newtonsoft.Json.Linq.JObject.Parse(json);
            foreach (Newtonsoft.Json.Linq.JToken hit in res["data"])
            {
                if ((string)hit["type"] != "partscheckresult") continue;
                string partnr = (string)hit["attributes"]["partnr"];
                string partid = (string)hit["attributes"]["partid"];
                // partnr + partid are PARTNR<i> / PARTID<i> for EDPCloudAddPartsToBasketAction
            }
        }
    }
}
```

For a token route add
`wc.Headers[HttpRequestHeader.Authorization] = "Bearer PAT:" + Environment.GetEnvironmentVariable("EPLAN_DATAPORTAL_PAT");`.

## From the EPLAN MCP server

The `eplan` MCP server wraps this API as `eplan_dataportal_api_find_parts`,
`_search_parts`, `_get_part`, `_catalog` (no token, no EPLAN needed),
`_download` and `_download_list` (token in the server's
`EPLAN_DATAPORTAL_PAT`). `eplan_dataportal_add_to_download_list` looks a
missing `dataportal_id` up through `/partscheck` itself.
