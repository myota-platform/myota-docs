# Catalogue navigation and persistence

```mermaid
flowchart TD
  Catalogue["Filtered catalogue and independent batch selection"]
  Select["Click entity name or map feature"]
  Editor["Focused editor dialog; catalogue position retained"]
  Details["Name and categories"]
  Location["Location and missing-field enrichment"]
  Geometry["Geometry map; explicit draft editing"]
  Evidence["Immutable source comparison and audit"]
  Save["Explicit section save with If-Match"]
  API["Geodata API; authorization and validation"]
  Refresh["Refresh revision and audit; preserve other drafts"]
  Leave["Close or previous / next entity"]
  Guard["Confirm discard if unsaved"]
  Delete["Global-admin impact and deletion confirmation"]
  Jobs["Durable deletion jobs; continuous status polling"]
  Catalogue --> Select --> Editor
  Editor --> Details & Location & Geometry & Evidence
  Details & Location & Geometry --> Save --> API --> Refresh --> Editor
  Editor --> Leave --> Guard --> Catalogue
  Evidence --> Delete --> Jobs
  Catalogue --> Delete
```

Review decisions remain on the separate review page. No database or broker
connection is introduced in the client. See the [editor guide](../entity-catalogue-editor.md).
