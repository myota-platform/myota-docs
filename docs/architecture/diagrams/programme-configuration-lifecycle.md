# Programme configuration lifecycle

Programme configuration is programme-owned and must be versioned before it
can affect new activity. Creating or editing a draft must not silently change
the rules used by historical activations, QSOs, or awards.

```mermaid
stateDiagram-v2
  [*] --> DRAFT
  DRAFT --> UNDER_REVIEW: submit configuration
  UNDER_REVIEW --> DRAFT: request changes
  UNDER_REVIEW --> PUBLISHED: approve + effective date
  PUBLISHED --> PUBLISHED: publish later version
  PUBLISHED --> SUSPENDED: suspend programme
  SUSPENDED --> PUBLISHED: resume with explicit decision
  PUBLISHED --> RETIRED: retire with effective date
  SUSPENDED --> RETIRED: retire with effective date
  RETIRED --> [*]
```

```mermaid
flowchart TB
  Draft[Programme draft]
  Validate[Schema validation\nrequired fields, references, accessibility]
  Review[Owner / approver review\nversion diff + audit trail]
  Publish[Published policy snapshot\neffective_from + version]
  Execute[Activity / geodata / identity\nconsume immutable snapshot]
  History[Historical activity and award records\nretain policy version used]
  Retire[Retired or superseded version\nno new use, history remains valid]

  Draft --> Validate
  Validate -->|valid| Review
  Validate -->|errors| Draft
  Review -->|changes requested| Draft
  Review -->|approved| Publish
  Publish --> Execute
  Execute --> History
  Publish -->|superseded| Retire
  Retire --> History
```

## Configuration layers

| Layer | Examples | Owner |
| --- | --- | --- |
| Programme identity | name, slug, owner, contacts, legal links, lifecycle | Programme service |
| Presentation/content | locales, fallback, theme, logos, public copy | Programme service + admin web |
| Eligibility policy | entity status/category, jurisdiction, location, duration, callsign, bands/modes, evidence | Programme service; evaluated by activity/geodata |
| Award linkage | published award references, participant categories, issuance defaults | Activity service owns definitions; programme stores policy linkage |
| Operational policy | notifications, public history, leaderboard/privacy, export defaults | Programme/activity services |

Imports remain programme-independent: a source adapter creates or refreshes a
candidate in geodata, and a programme later decides whether the category,
status, jurisdiction, provenance, and geometry satisfy its published policy.
