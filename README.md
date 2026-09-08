# GDB Studio
An Esri-free tool for ArcGIS geodatabase schemas held as XML workspace documents: read and write them, export an interactive HTML viewer or an editable draw.io diagram, validate a schema, diff two versions, and generate arcpy scripts to create or migrate a geodatabase. A WinForms editor sits on top of the same library.

Nothing in this project links against ArcObjects, arcpy, or any Esri SDK. The only input and output format is the XML workspace document ArcGIS itself reads and writes (Export/Import XML Workspace Document), so the tool works anywhere a .xml schema export can be produced, with no ArcGIS installation required to run it.
