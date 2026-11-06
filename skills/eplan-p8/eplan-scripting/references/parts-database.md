# Parts Database: properties, XML export/import, deletion, Data Portal

Facts for checking and editing a parts database from outside the parts
management dialog. Everything here was measured on **EPLAN 2025.0.3**
unless a line says otherwise. Reading parts one by one through
`MDPartsManagement` is in `api-data-access.md`; this page is the bulk side.

## Part properties: numbers and API names

The numbers and names come from the API documentation EPLAN installs next to
its binaries (`Eplan.EplApi.DataModelu.xml` in the `Bin` folder). In the XML
export the same property is an attribute named `P_` + the API name.

| Number | API name | Export attribute | What it holds |
|---|---|---|---|
| 22001 | `ARTICLE_PARTNR` | `P_ARTICLE_PARTNR` | Part number |
| 22056 | `ARTICLE_ERPNR` | `P_ARTICLE_ERPNR` | ERP number |
| 22258 | `ARTICLE_DISCONTINUED` | `P_ARTICLE_DISCONTINUED` | Discontinued, `0` or `1` |
| 22007 | `ARTICLE_MANUFACTURER` | `P_ARTICLE_MANUFACTURER` | Manufacturer, its **short name** (not the long one) |
| 22009 | `ARTICLE_NOTE` | `P_ARTICLE_NOTE` | The field the parts management dialog shows as **"Description"** (multilanguage) |
| 22145 | `ARTICLE_GROUPSYMBOLMACRO` | `P_ARTICLE_GROUPSYMBOLMACRO` | Schematic macro (a path) |
| 22045 | `ARTICLE_PICTUREFILE` | `P_ARTICLE_PICTUREFILE` | Picture file (a path) |

**`ARTICLE_NOTE` is not `ARTICLE_DESCR1`.** The dialog's "Description" box is
22009 `ARTICLE_NOTE`; `ARTICLE_DESCR1/2/3` are separate properties. A check
for "parts without a description" that reads `ARTICLE_DESCR1` reports the
wrong parts.

Property numbers are what parameters such as `PROPERTYID<i>` of the
`partsmanagementapi` action take (below), so keep the number next to the name.

## The `XPamExportXml` format

Export with the `partsmanagementapi` action. `PARTNUMBER1:*` exports every
part; `PARTNUMBER1`, `PARTNUMBER2`… pick some. The import side of the same
format is called `XPamImportXml`.

```csharp
using (ActionCallingContext acc = new ActionCallingContext())
{
    acc.AddParameter("TYPE", "EXPORT");
    acc.AddParameter("EXPORTFILE", @"C:\temp\parts.xml");
    acc.AddParameter("FORMAT", "XPamExportXml");
    acc.AddParameter("PARTNUMBER1", "*");
    new CommandLineInterpreter().Execute("partsmanagementapi", acc);
}
```

A full export of about 14,000 parts took about 21 s and wrote about 80 MB.
Read a file that size as a stream, not into one string. It is
machine-written: no DTD, no CDATA, attribute values in double quotes.

Shape, with made-up parts:

```xml
<?xml version="1.0" encoding="utf-8" ?>
<partsmanagement version="2.9.4" build="0" type="1" length-unit="mm" weight-unit="kg" count="2">
 <part P_ARTICLE_PARTNR="DEMO.CT-100" P_ARTICLE_ERPNR="100001" P_ARTICLE_DISCONTINUED="0"
       P_ARTICLE_MANUFACTURER="DEMO" P_ARTICLE_NOTE="en_US@Contactor 3 kW;es_ES@Contactor 3 kW;"
       P_ARTICLE_GROUPSYMBOLMACRO="$(MD_MACROS)\Demo\CT-100.ema"
       P_ARTICLE_PICTUREFILE="$(MD_IMG)\Demo\CT-100.jpg">
  <freeproperty pos="1" P_ARTICLE_FREE_DATA_DESCRIPTION="??_??@Coil 24 V DC;" />
  <variant P_ARTICLE_VARIANT="0">
   <functiontemplate pos="1" functiondefcategory="1" functiondefgroup="1" functiondefid="1" />
  </variant>
 </part>
 <part P_ARTICLE_PARTNR="DEMO.TB-10" P_ARTICLE_DISCONTINUED="0">
  <variant P_ARTICLE_VARIANT="0" />
 </part>
</partsmanagement>
```

- `<partsmanagement>` holds one `<part>` per part. The part's properties are
  attributes (159 different ones seen). Children: `variant` (with its
  `functiontemplate` elements), `freeproperty`, `userproperty`,
  `accessoryposition` and a few more.
- **An empty property is left out**, not written as `""`. In the example,
  `DEMO.TB-10` has no ERP number, manufacturer or description: the attributes
  are simply missing. Treat "attribute absent" as empty.
- **Multilanguage text** is `xx_XX@text;` repeated, one per language;
  `??_??@` is the language-neutral entry. A value can be present with every
  language blank (`??_??@;`): that is empty too. `;` ends an entry, so it
  can't appear inside the text.
- **Paths** start with a path variable: `$(MD_MACROS)\…` for macros,
  `$(MD_IMG)\…` for pictures. Resolve them with `PathMap.SubstitutePath`
  before touching the disk.

### Import modes

The `MODE` parameter of `partsmanagementapi /TYPE:IMPORT`:

| Mode | Meaning |
|---|---|
| 0 | Append new records only |
| 1 | Update existing records only |
| 2 | Update existing records and append new ones |
| 3 | Replace existing records and append new ones |

To change a few fields of existing parts, import **mode 1** with the **whole
`<part>` elements as exported**, only the attributes you mean to change
rewritten. Measured: only the attributes that differ changed; variants,
function templates and the number of parts stayed as they were. Writing whole
parts means an update can't blank what the file didn't mention. Mode 3
*replaces* existing records with what the file says: treat it as destructive.

Before any import, export the parts it touches (or the whole database) to a
file you keep. After it, export the same parts again and compare them field by
field: that is the only check that an import did what you meant.

## Methods that delete

These calls on `MDPartsDatabase` / `MDPartsManagement`
(`Eplan.EplApi.MasterData`) remove master data:

| Method | Removes |
|---|---|
| `RemovePart` | A part |
| `RemoveDatabase` | A whole parts database |
| `RemoveManufacturer` | A manufacturer |
| `RemoveCustomer` | A customer |
| `RemoveAllTranslations` | The translations |

**There is no undo.** Writes take effect at once (no save step, see
`api-data-access.md`), and a parts database is often shared by a whole
team, so a mistake reaches everyone who uses it. Don't call these from a
generated or automated script; if parts really must go, a person does it in
EPLAN's parts management after a backup. A tool that runs scripts for someone
else should refuse code that calls them.

Deleting the database file itself is the same thing by other means. EPLAN's
own parts database file is `.alk`; an Access one is `.mdb`/`.accdb`; a SQL
Server one is dropped with SQL. Guard all three the same way.

## The Data Portal add-in

EPLAN's Data Portal add-in registers actions that run with EPLAN's own Data
Portal sign-in. Find them with `ActionManager.FindAction(name)`; `null` means
the add-in isn't loaded, isn't licensed or the version doesn't have it.

| Action | 2025 | 2026 | Parameters |
|---|---|---|---|
| `EDPCloudAddPartsToBasketAction` | yes | yes | `PARTNR<i>` and `PARTID<i>`, both required, `<i>` from **1** |
| `EDPCloudGetSearchCountAction` | **no** | yes | `search`; the count comes back in `result` |

`EDPCloudAddPartsToBasketAction` puts parts in the user's download list:

- `PARTID<i>` is the Data Portal's own id of the part (the number after `id-`
  in its web address). `PARTNR<i>` alone, or indexes from 0, give
  `Execute` → `false` (measured on 2026).
- A part already in the list also gives `false`.
- **On 2025 it opens the Data Portal window**, and the call doesn't return
  while that window is open. The part is queued anyway; don't queue it again
  because the call timed out.
- Queueing imports nothing. The download is a modal dialog in EPLAN, so a
  person has to open the Data Portal, go to the download list and download it.

```csharp
using (ActionCallingContext acc = new ActionCallingContext())
{
    acc.AddParameter("PARTNR1", "DEMO.CT-100");
    acc.AddParameter("PARTID1", "123456");      // the part's Data Portal id
    Eplan.EplApi.ApplicationFramework.Action act =
        new ActionManager().FindAction("EDPCloudAddPartsToBasketAction");
    if (act == null) throw new Exception("Data Portal add-in not available");
    bool queued = act.Execute(acc);
}
```

### What a download writes

1. The files are staged in `%TEMP%\EDPADL\<id>\`, in subfolders named after
   the master data folders: `$(MD_MACROS)`, `$(MD_IMG)`, `$(MD_XML)`.
2. Then they are copied into those folders. **The user needs write access to
   all three**: with one of them on a read-only share the download fails
   inside EPLAN ("access denied"), after the staging. Check first by creating
   and deleting an empty file in each resolved folder.
3. **The portal skips the part data of parts the database already has**, but
   still brings their files. Downloading an existing part puts its macro
   file and picture on disk and leaves the part's record unchanged:
   description, manufacturer, macro path and function templates are not
   updated. To use the macro, set 22145 `ARTICLE_GROUPSYMBOLMACRO` to the file
   yourself. The other side of it: don't create an empty placeholder part
   before downloading a part you don't have, or the download skips its data.

A parts database made by hand without EPLAN's own tables (one without
`tblOption`, in one test) made the download fail; a database EPLAN created
works.

### The documented web API

The Data Portal has a documented web API at `https://dataportal.eplan.com/api`
(OpenAPI 3.0, JSON:API responses; description at `/api/doc`). Without a token:

- `POST /partscheck` with `Content-Type: text/plain`, one part number per line
  (`GET` works too): a `partscheckresult` per number found (`partid`,
  `partnr`, `typenr`, `ordernr`, `part_version`, `upload_date`…) or a
  `partscheckmissingresult`. The `partid` is what `PARTID<i>` above takes.
- `GET /parts/{id}`: the part's details, with `designation` and `description`
  per language, `lifecycle`, `identcode`, macro paths and relationships to its
  manufacturer, picture, function templates and macros.
- `GET /parts`: search.

Send `Accept: application/vnd.api+json`. A hit whose manufacturer differs from
the one in your database is a different part with the same number, not a
match. Some catalogs' numbers aren't found exactly as a database stores them
(punctuation differs), so a miss doesn't prove the part isn't there.

The download list over the web API (`/basketitems`) needs a personal access
token the user makes in their ePulse profile, sent as
`Authorization: Bearer PAT:<token>`; without one it answers
`401 Missing JWT token`. EPLAN part data itself only arrives through the
add-in's download, not through the web API.

Use only these two surfaces, the add-in's actions and the documented API.
The Data Portal window's own internal calls carry the user's EPLAN licence and
are not an interface.
