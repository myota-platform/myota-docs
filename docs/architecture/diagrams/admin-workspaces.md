# Administration workspace navigation

```mermaid
flowchart TD
  Identity["Identity API: resolved roles and scopes"] --> Workspaces["Shared authorized workspace definitions"]
  Workspaces --> Sidebar["Searchable sidebar and breadcrumbs"]
  Workspaces --> Guard["Route guards and shared page headings"]
  Workspaces --> Overview["Overview: available tasks and partial live signals"]
  Sidebar --> Entities["Entities: catalogue, map, review, imports, categories"]
  Sidebar --> Programmes["Programmes: configuration, policies, content, certificates"]
  Sidebar --> People["People: users, roles, security tabs"]
  Sidebar --> Activity["Activity: activations and QSO counts"]
  Sidebar --> Health["Platform health: JetStream, storage, Grafana"]
  Programmes --> Scope["One programme selector; read-only header context"]
  Entities --> Shared["Shared catalogue and imports; unassigned entities included"]
  Guard --> APIs["Existing service APIs remain authorization authority"]
```

Existing URLs stay valid. Category definitions are shared; programme assignments
stay in Programmes. Certificate design is separate from policy schemas. See the
[workspace guide](../../domain/administration/navigation-reorganization.md).
