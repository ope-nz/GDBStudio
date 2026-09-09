# GDB Studio
An tool for ArcGIS geodatabase schemas held as XML workspace documents: read and write them, export an interactive HTML viewer or an editable draw.io diagram, validate a schema, diff two versions, and generate arcpy scripts to create or migrate a geodatabase. A WinForms editor sits on top of the same library.

Nothing in this project links against ArcObjects, arcpy, or any Esri SDK. The only input and output format is the XML workspace document ArcGIS itself reads and writes (Export/Import XML Workspace Document), so the tool works anywhere a .xml schema export can be produced, with no ArcGIS installation required to run it.

<img width="1364" height="874" alt="image" src="https://github.com/user-attachments/assets/c4a8c876-bf4d-4cb2-a7b7-8f5362a9d5aa" />

## Features

**Core model**
- Typed, round-trip-lossless model over the live XML - reading and re-saving a document
  reproduces it byte-for-byte, including elements and attributes the model doesn't
  understand (geometric networks, topologies, network datasets, and anything future
  ArcGIS versions add) - nothing is ever silently dropped.
- Understands feature datasets, feature classes, tables, relationship classes (simple and
  attributed), fields, domains (coded value and range), subtypes with per-subtype field
  overrides, indexes, spatial references, attribute rules (Arcade), field groups, and
  contingent values (a reverse-engineered, undocumented Protocol Buffers format Esri itself
  has never published a spec for).

**Validation**
- 35 rules across 7 families (Workspace, Domains, Tables/Classes, Fields, Feature Classes,
  Relationship Classes, Attribute Rules) - see [Validation Rules](docs/validation.md) for
  the full list.
- Severity for name-length and index rules follows the target geodatabase kind (file vs.
  enterprise), auto-detected from the workspace or forced with a flag.
- Tuned against real ArcGIS Enterprise production exports, not just Esri's small published
  samples - several rules are deliberately more permissive because a real schema legitimately
  does something a naive rule would flag.

**Diff**
- Structural comparison of two schema documents: added, removed, and changed objects,
  matched by name (coded values and subtypes by code so a rename isn't reported as a
  delete+add).
- Configurable ignore options for metadata, DSIDs/CLSIDs, spatial reference, editor
  tracking, and housekeeping tables (attachments, archives) - the noise that changes on
  every export regardless of an intentional schema change.

**Exports** - every one a plain file, opened afterward with whatever's associated with it;
nothing renders inside the tool itself
- **Interactive HTML viewer** - a single self-contained file (~3.7 MB + schema size) with a
  filterable sidebar tree, per-class/domain/relationship pages, and live Mermaid
  entity-relationship diagrams. No server, no network, works from a `file://` URL.
- **Schema report (HTML)** - a printable report modeled on ArcGIS Pro's own "Generate Schema
  Report": workspace properties, an anchor-linked dataset index, and a section per class,
  domain, and relationship class. A legacy XSLT-driven report is also available.
- **Schema report (XLSX)** - a `.xlsx` workbook shaped like Pro's own Excel export: a TOC
  sheet plus the same 15 flat, cross-cutting sheets (Field, Index, Domain, and so on), each
  a real, filterable Excel Table - hand-written OOXML, no third-party library.
- **draw.io / diagrams.net diagrams** - editable entity-relationship diagrams: tables as
  native shapes colour-coded by geometry type, relationship classes as labelled edges,
  feature datasets as grouped containers, domains as linked boxes.
- **Mermaid diagram text** - for a class neighbourhood, a feature dataset, or the whole
  schema, printable or copyable straight to the clipboard.
- **arcpy scripts** - see below.

**arcpy script generation**
- **Create scripts**: recreate a class, a feature dataset, or the whole schema - domains,
  fields, subtypes, indexes, relationship classes, attribute rules, editor tracking, and
  Global IDs, in dependency order.
- **Migration scripts**: diff two schema versions and emit the exact arcpy calls
  (`AddField`, `AlterField`, `AssignDomainToField`, `AddSubtype`, `AddAttributeRule`, and so
  on) that bring a geodatabase built from the baseline up to the target. Anything arcpy
  can't express in place becomes a `# TODO` naming the exact schema path.
- Never calls arcpy itself - it writes Python text for you to read and run.

**WinForms editor**
- Catalog tree, tabbed class panel (fields, indexes, subtypes with override grids,
  attribute rules with a full Arcade script viewer, relationships, field groups with
  decoded contingent values), a domain panel with a live "used by" cross-reference, and a
  property grid for everything else.
- Full undo/redo (Ctrl+Z / Ctrl+Y) via whole-document XML snapshots - always an exactly
  consistent state, never a partial edit.
- Renaming a field, class, feature dataset, or domain updates every other place in the
  document that stores the old name as a plain string copy (index field lists, subtype
  overrides, attribute rule targets, relationship keys, contingent value field groups,
  catalog paths) in the same undo step.
- Create new feature datasets, feature classes, tables, relationship classes, and domains
  from the same embedded templates the original Esri sample tool shipped - correct
  `xsi:type`, well-known CLSIDs, and system fields from the start.
- A searchable coordinate system picker backed by a portable, xcopy-deployed `.prj` library
  next to the executable - nothing pre-supplied, import your own or point it at a folder.
- Live validation (F5), a Compare-with-another-document window (added/removed/changed,
  exportable as HTML or an arcpy migration script), and every export format above,
  scoped to a selection where that makes sense.
- Error list panel with severity filters, auto-validate-on-edit, and an **Export...**
  button that saves the current errors/warnings/notes to a text file.

**Everything else**
- No dependency on ArcObjects, arcpy, or any Esri SDK, anywhere in the tool.
- No third-party WinForms controls - standard controls only.
- Distributable binaries are code-signed at build time when a certificate is available.
- The About box and CLI both show the actual build date (`YYYY.MM.DD`), not a
  hand-maintained version number that drifts out of date.

---

## Requirements

| | |
|---|---|
| **Runtime** | .NET Framework 4.5 (Windows) |
| **Build tools** | None beyond the framework itself - `csc.exe` ships with Windows |
| **To run the editor** | Windows, WinForms - no other runtime |
| **To use the CLI or Core library** | Same - it's a plain .NET Framework assembly |
| **To browse an exported HTML viewer** | Any modern browser, no server needed |

---

## CLI reference

```
gdbstudio <command> [arguments]
```

`gdbstudio help`, no arguments, or an unknown command prints the same usage summary. Flags
are `--name value` or `--name=value`; a flag with no value (like `--open`) is a switch.

| Command | Usage | What it does |
|---|---|---|
| `info` | `gdbstudio info <file.xml>` | Prints namespace version, workspace type, element/field/domain counts, and feature dataset summaries. Read-only. |
| `roundtrip` | `gdbstudio roundtrip <in.xml> [out.xml]` | Loads and re-saves the document, reporting whether the result is structurally equivalent. Exit `0` equivalent, `3` different. |
| `html` | `gdbstudio html <in.xml> [out.html] [--title T] [--open]` | Writes the self-contained interactive HTML viewer. |
| `json` | `gdbstudio json <in.xml> [out.json]` | Writes the viewer's JSON model directly. |
| `mermaid` | `gdbstudio mermaid <in.xml> [--class NAME \| --dataset NAME] [--fields] [--system]` | Prints Mermaid diagram text to stdout. |
| `report` | `gdbstudio report <in.xml> [out.html] [--title T] [--open] [--legacy]` | Writes the printable schema report (or `--legacy` for the original XSLT report). |
| `xlsx` | `gdbstudio xlsx <in.xml> [out.xlsx] [--title T] [--open]` | Writes the Excel schema report (TOC + 15 flat sheets). |
| `drawio` | `gdbstudio drawio <in.xml> [out.drawio] [--dataset NAME] [--nosystem] [--maxfields N]` | Writes an editable diagrams.net diagram. |
| `validate` | `gdbstudio validate <in.xml> [--target fgdb\|enterprise] [--no-warnings] [--suppress ID,ID] [--html out] [--json out] [--txt out]` | Runs the validation rules. Exit code = number of errors (capped at 250). |
| `diff` | `gdbstudio diff <a.xml> <b.xml> [--html out] [--json out] [--include-metadata] [--include-ids] [--ignore-sr] [--ignore-tracking] [--ignore-housekeeping]` | Compares two documents. Exit `0` equivalent, `4` different. |
| `rules` | `gdbstudio rules` | Lists every validation rule id and description, live from the rule set. |
| `arcpy create` | `gdbstudio arcpy create <in.xml> [out.py] [--class NAME] [--dataset NAME] [--gdb PATH]` | Writes a Python script that recreates a class, a dataset, or the whole schema. |
| `arcpy migrate` | `gdbstudio arcpy migrate <a.xml> <b.xml> [out.py] [--gdb PATH] [--no-delete] [--ignore-sr] [--ignore-tracking]` | Writes a Python migration script from a diff of two documents. |
| `templates` | `gdbstudio templates` | Lists the embedded "new object" template names. |
| `version` | `gdbstudio version` | Prints the product name and build date. |

### Quick examples

```
bin\gdbstudio info "samples\esri\ArcGIS Telecom.xml"
bin\gdbstudio html "samples\esri\ArcGIS Telecom.xml" out\telecom.html --open
bin\gdbstudio validate schema.xml --target enterprise --txt out\findings.txt
bin\gdbstudio diff baseline.xml target.xml --html out\changes.html
bin\gdbstudio arcpy create schema.xml out\create.py --class Structure
bin\gdbstudio arcpy migrate baseline.xml target.xml out\migrate.py --gdb C:\Data\Live.gdb
bin\gdbstudio drawio schema.xml out\schema.drawio --dataset Water
bin\gdbstudio xlsx schema.xml out\schema.xlsx --open
```

---

## Editor

```
bin\GdbStudioEditor.exe [schema.xml] [--select /FD=Water/FC=Main]
```

Opens a new empty workspace with no argument, or the given file with the object at that
[schema path](docs/xml-format.md#schema-paths) selected. See [`docs/editor.md`](docs/editor.md)
for the full walkthrough of the catalog tree, class panel, domain panel, creating new
objects, the coordinate system library, validation, comparison, and every Export menu item.

---

## Documentation

### Working on the HTML viewer

The viewer's HTML, CSS, and JavaScript live under `web\viewer` (`viewer.html`,
`viewer.css`, `js\*.js`), not inside the compiled assembly. `build\build-viewer.ps1`
inlines everything into `web\viewer\dist\viewer.html`, which `core.rsp` embeds;
`build.cmd` runs the bundler automatically. To iterate without a full rebuild:

```
bin\gdbstudio json "samples\esri\ArcGIS Telecom.xml" web\viewer\dev\model.json
powershell -ExecutionPolicy Bypass -File web\viewer\serve.ps1
```

then open `http://localhost:8080/web/viewer/viewer.html`, which falls back to
`dev\model.json` when no model is embedded - edits to the JavaScript or CSS show up on a
browser refresh, no rebuild needed. `web\viewer\dev` is git-ignored.

---

## Attribution

A handful of data files (`data\xslt\*.xslt`, `data\xsd\*.xsd`, `data\templates\*.xml`) are
copied from Esri's ArcGIS Diagrammer sample code; everything else is
this project's own code.

Developed by [Ope Ltd](https://www.ope.nz).

