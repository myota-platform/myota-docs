# Admin UI internationalization roadmap

**Status:** proposed implementation plan; no phases below are claimed complete.

**Progress tracking:** leave items unchecked until implementation and evidence
are available; mark `[x]` only when verified. A phase is complete after every
task and exit criterion is checked and its evidence is recorded.

**Scope:** internationalization (i18n) of the MyOTA administration web UI,
including interface text, locale selection, formatting, accessibility, and
translation maintenance. This plan covers `myota-admin-web`; API contracts and
other clients are in scope only where they affect UI language behavior.

**Owner:** `myota-admin-web`; coordinate language policy and API error/content
semantics with `myota-contracts`, `myota-identity-service`,
`myota-programme-service`, and `myota-docs` as needed.

## Description

Internationalize the administration UI so operators can use its navigation,
forms, tables, dialogs, status messages, validation, and help in the supported
languages. Keep a single source locale for interface message keys, make locale
selection predictable, and let browser language preferences choose a sensible
initial locale. Provide a visible in-app language control and persist the
operator's explicit choice. If authenticated user preferences are later
available, define a deliberate precedence rather than silently changing the
current choice.

The admin web is a Vue 3 and TypeScript application. Its existing programme
Content workflow already manages locale-tagged, programme-owned content with
fallback and published-key coverage. That workflow remains distinct: UI
translations are shipped with the admin application and are not programme
content records. Localize the Content workflow's own controls while preserving
the locale values and text that programme administrators enter.

Choose the initial supported locales in Phase 0. English is the source locale;
Spanish is a candidate first translation based on MyOTA's operating context.
Do not claim support for a locale until its required UI messages are complete
and reviewed. Missing translations must resolve through an explicit fallback
chain and never render raw keys or blank controls. Preserve the original
message and diagnostic context for development, while showing a readable
fallback in production.

Use locale-aware browser APIs for dates, numbers, percentages, and lists. Keep
stored/API timestamps in UTC and convert only for presentation under the
operator's selected locale and documented time-zone policy. Locale selection
must not change authorization, programme scope, API payload semantics, or
stored content. Translation must not alter service-owned status codes, stable
identifiers, or audit values.

## Design rules

- Treat translation keys as stable UI identifiers; do not use translated text
  as a key or persist it as domain data.
- Keep source messages and locale catalogs organized by feature or namespace,
  with typed key access where supported by the chosen library.
- Use interpolation/plural rules for counts and dynamic values; do not build
  sentences by concatenating translated fragments.
- Keep API error codes and validation field identities stable. Map known codes
  to localized messages in the UI and retain a safe localized generic fallback
  for unknown errors.
- Keep callsigns, entity IDs, API field names, code values, and user-entered
  programme content unchanged. Translate their labels and explanatory copy.
- Do not translate user-entered names or catalogue values automatically.
- Support long translated labels, keyboard navigation, visible focus, screen
  reader announcements, and text zoom without clipping essential actions.
- Avoid locale-specific assumptions in sorting/searching. Use `Intl.Collator`
  for user-visible localized text; keep canonical IDs and codes stable.
- Avoid including translation strings in logs, telemetry labels, or metrics.
- Preserve current English behavior as the fallback until each migrated screen
  is verified.

## Phased implementation

### Phase 0 — Inventory, language policy, and technical decision

**Work**

- [ ] Inventory every user-visible string in `myota-admin-web`, including
      navigation, forms, tables, empty/loading/error states, validation,
      confirmations, tooltips, ARIA text, document titles, and map controls.
- [ ] Separate admin-interface strings from programme-managed localized
      content, API values, status codes, identifiers, and user-entered data.
- [ ] Agree on the initial supported locales, source locale, fallback order,
      translation review owner, and how incomplete locales are represented.
- [ ] Decide locale precedence: explicit persisted UI choice, browser language,
      then source locale. Document behavior for invalid or retired locale values.
- [ ] Select the Vue-compatible translation approach after checking current
      project dependencies, bundle size, lazy-loading support, pluralization,
      type safety, and testability. Record the decision and avoid custom
      message parsing unless a documented need justifies it.
- [ ] Define key naming, namespace ownership, interpolation/plural conventions,
      missing-key diagnostics, API error mapping, and catalog completeness
      checks.
- [ ] Define presentation policy for date/time, number, percentage, currency
      (if introduced), sorting, and browser time zone. Keep persisted instants
      and API semantics unchanged.

**Exit criteria**

- [ ] String inventory covers all routes/components and identifies shared
      chrome versus feature-owned text.
- [ ] Language, fallback, persistence, and review decisions are documented.
- [ ] A representative catalog/key structure and completeness-check approach
      are approved before bulk string migration.

**ChatGPT prompt — Phase 0**

```text
Work in the MyOTA workspace. Perform a read-only discovery phase for
internationalizing the administration UI in myota-admin-web. Do not change
runtime code or install dependencies.

Inspect the Vue 3/TypeScript application, package manifest and lockfile,
router, shell/navigation, views, components, composables, styles, browser
tests, API client/error handling, and docs. Inventory every visible string,
including labels, hints, buttons, dialogs, loading/empty/error states,
validation, page titles, map controls, and accessibility text. Group findings
by shared chrome and feature, and identify strings that are not UI text:
programme-managed Content values, API codes, identifiers, callsigns, and
user-entered data.

Propose the smallest maintainable Vue-compatible i18n approach. Compare
existing dependencies and current official library guidance from repository
metadata; do not add a dependency in this phase. Recommend source locale,
candidate initial locales, fallback behavior, explicit-choice persistence,
browser-language precedence, catalog structure/key rules, pluralization,
missing-key checks, API error mapping, and locale-aware formatting. Preserve
UTC API/storage semantics and distinguish interface translations from the
existing programme Content locale/fallback workflow.

Write an evidence-backed decision and string inventory to the administration
i18n roadmap in myota-docs. Clearly label proposals and current behavior. Do
not claim any locale is implemented. Report coverage gaps and Phase 0 exit
criteria.
```

### Phase 1 — Localization foundation and locale selection

**Work**

- [ ] Add the approved translation dependency and configure typed or otherwise
      validated catalogs, source locale, supported locale metadata, and an
      explicit fallback chain.
- [ ] Add an application-level translation/composition setup before routes
      render; keep locale changes reactive without a full page reload.
- [ ] Add a language selector in the persistent admin shell with localized
      language names, accessible labeling, keyboard operation, and a clear
      selected state.
- [ ] Resolve locale on startup using the Phase 0 precedence rules and persist
      explicit selection using the approved preference mechanism.
- [ ] Handle unsupported, malformed, or unavailable persisted/browser locale
      values safely; fall back to the source locale and avoid blank screens.
- [ ] Add catalog loading and missing-key diagnostics appropriate to
      development and production; keep catalogs bundled until lazy loading is
      specifically justified.
- [ ] Add a minimal shared proof of concept for shell navigation and a
      parameterized message, then document the pattern for feature authors.

**Exit criteria**

- [ ] Locale can be changed and restored across reloads without changing route,
      programme scope, authorization, or API behavior.
- [ ] Fallback and missing-key behavior is visible, safe, and covered by
      focused automated checks.
- [ ] A second locale can render the shared-shell proof of concept without
      changes to business logic.

**ChatGPT prompt — Phase 1**

```text
Implement Phase 1 of the approved admin UI i18n roadmap in myota-admin-web.
First read the Phase 0 decision in myota-docs and follow its selected library,
locale policy, catalog structure, and persistence rules. Do not broaden scope
to translate every screen yet.

Configure the translation layer and supported/source locales, add reactive
locale switching in the persistent admin shell, and provide accessible
language names and selected-state behavior. Apply the documented startup
precedence and persist only an explicit operator choice. Unsupported or
invalid values must safely fall back; missing keys must not appear as raw
identifiers in production. Keep locale changes from altering current route,
programme scope, auth state, API payloads, or UTC handling.

Translate only the minimum shell proof of concept, including one interpolated
or pluralized message. Add the documented developer usage pattern and focused
automated checks for locale resolution, fallback, persistence, switching, and
missing keys. Update the admin-web README and myota-docs roadmap with exact
implementation paths and evidence. Run only the checks required by the phase
plan; report commands and results, remaining gaps, and do not mark later
phases complete.
```

### Phase 2 — Shared shell, authentication, and common feedback

**Work**

- [ ] Migrate the app shell, page headers, programme scope selector, dashboard,
      global navigation, access-denied page, and shared layout controls.
- [ ] Localize login, logout/session-expiry, authentication failures, and
      common authorization messages without exposing internal error details.
- [ ] Localize shared feedback patterns: loading, empty, stale, success,
      retryable error, destructive confirmation, and generic unexpected error.
- [ ] Define a stable mapping from known API error codes/validation fields to
      localized messages; preserve safe fallback behavior for unknown codes.
- [ ] Localize document titles, button accessible names, landmarks, and live
      announcements used by shared components.
- [ ] Remove migrated hard-coded strings from the shared shell and add a
      completeness check for the catalogs introduced so far.

**Exit criteria**

- [ ] All shared routes and auth/feedback states render in each supported
      locale with no raw keys, empty labels, or untranslated shared controls.
- [ ] Existing auth and access behavior is unchanged when switching locales.
- [ ] API error mapping has coverage for representative success, validation,
      authorization, conflict, and unknown-error cases.

**ChatGPT prompt — Phase 2**

```text
Implement Phase 2 of the admin UI i18n roadmap in myota-admin-web. Read the
approved Phase 0 policy and Phase 1 translation setup before editing. Migrate
only shared application chrome, dashboard, authentication/access-denied
surfaces, programme scope selector, page headers, and common feedback states.

Move all user-visible text in those surfaces into the established catalogs.
Use stable API error codes and validation identities to select localized
messages; keep unknown errors safe and useful without displaying raw server
internals. Translate document titles, ARIA labels, live announcements, empty
and loading text, confirmation buttons, and retry actions as well as visible
headings. Keep callsigns, programme identifiers, route/API values, and
authorization decisions unchanged.

Update or add focused checks proving each shared surface works in every
currently supported locale and locale changes do not affect auth or programme
scope. Run the phase-required checks, update catalog completeness rules,
record evidence and remaining string inventory in myota-docs, and leave
feature-workspace translations for later phases.
```

### Phase 3 — Domain workspaces and operational workflows

**Work**

- [ ] Migrate Programme setup and configuration views, preserving programme
      names, descriptions, policy text, and authored content as user data.
- [ ] Migrate geodata review, imports, entity catalogue/editor, map workspace,
      GIS forms, lifecycle states, and destructive-operation dialogs.
- [ ] Migrate Operations & access, Identity, Activity, Awards, JetStream, and
      object-storage views, including status descriptions and safe error states.
- [ ] Localize table headers, filters, pagination, selection, bulk actions,
      date-range labels, forms, validation, tooltips, and no-results states.
- [ ] Keep service status codes and catalogue/category codes stable; translate
      their display labels through explicit mappings with an unknown-code
      fallback.
- [ ] Localize the admin Content-management interface only. Continue to store
      and display authored content by its selected locale and existing
      fallback/publishing rules.
- [ ] Track migration completion by route/component so no screen is treated as
      covered just because its navigation label is translated.

**Exit criteria**

- [ ] Every routed admin workspace has a reviewed string inventory and no
      hard-coded user-facing English strings outside approved proper nouns,
      code values, or user data.
- [ ] Critical mutation, review, import, award, identity, and operations flows
      remain understandable in all supported locales.
- [ ] Content-management locale behavior and programme-authored values remain
      unchanged by UI locale switching.

**ChatGPT prompt — Phase 3**

```text
Implement Phase 3 of the MyOTA admin UI i18n roadmap in myota-admin-web.
Use the established catalogs and migration conventions; work feature by
feature and keep business logic out of translated strings.

Migrate all visible text and accessibility text in Programme setup, geodata
review/import/catalogue/editor/map workflows, Operations & access, Identity,
Activity, Awards, JetStream, and object-storage screens. Include form
validation, filters, table and pagination labels, empty/loading/error states,
bulk actions, confirmations, status descriptions, and responsive map/editor
controls. Use explicit display mappings for known status/category codes and a
safe fallback for unknown codes. Do not translate or mutate stable IDs,
service values, callsigns, programme-authored names/descriptions/policies, or
user input.

The Content view's controls are UI strings; its locale-tagged programme
content remains user-managed data and its existing fallback/publication
semantics must not change. Maintain a route-by-route completion inventory.
Add/update checks for representative high-impact workflows and catalog
completeness. Update the roadmap with evidence, unresolved translations, and
any required follow-on work; do not claim phase completion without review.
```

### Phase 4 — Locale-aware formatting, layout, and accessibility

**Work**

- [ ] Replace locale-sensitive display formatting with `Intl.DateTimeFormat`,
      `Intl.NumberFormat`, and `Intl.Collator` or the library-approved wrapper.
- [ ] Preserve UTC as the API/storage basis; explicitly distinguish UTC
      timestamps, date-only values, and operator-local display where appropriate.
- [ ] Review translated text expansion, narrow layouts, tables, dialogs,
      navigation, maps, and award designer controls for clipping or hidden
      actions.
- [ ] Verify keyboard navigation, focus order/visibility, screen-reader names,
      live updates, language selector behavior, and document `lang` attribute.
- [ ] Verify that numbers, percentages, dates, plural forms, punctuation,
      sorting, and fallback messages are correct in each supported locale.
- [ ] Confirm third-party map/library controls are localized where supported;
      document controls that remain in a fixed language and any alternatives.
- [ ] Do not implement right-to-left layout until a supported RTL locale is
      explicitly selected; record layout prerequisites if needed.

**Exit criteria**

- [ ] Formatting and locale changes do not shift stored/API timestamps or alter
      UTC date inputs on unchanged records.
- [ ] All supported locales pass responsive and keyboard/screen-reader review
      for the agreed critical routes.
- [ ] Known third-party localization limitations are documented with owner and
      resolution path.

**ChatGPT prompt — Phase 4**

```text
Implement Phase 4 of the admin UI i18n roadmap in myota-admin-web. Audit the
translated screens and replace locale-dependent presentation with the
approved Intl wrappers for dates, numbers, percentages, sorting, and plural
messages. Preserve UTC API/storage values, date-only semantics, and existing
unchanged-instant behavior; changing the interface locale must never rewrite
server data.

Review all supported locales for long-text expansion, responsive layouts,
table overflow, dialogs, navigation, map/editor controls, keyboard access,
visible focus, screen-reader names/live messages, and the root document lang
attribute. Check third-party map controls and document any that cannot follow
the selected language. Do not add RTL scope unless Phase 0 explicitly selected
an RTL locale.

Add targeted automated coverage for formatting and locale switching, then
perform the accessibility/responsive review on critical routes. Update
myota-docs with evidence, UTC/formatting limits, and third-party gaps. Report
exact checks and results; do not mark release readiness until Phase 5 gates
are met.
```

### Phase 5 — Translation quality, release gates, and maintenance

**Work**

- [ ] Complete human review of every supported locale by a fluent reviewer;
      mark locale support incomplete until that review is recorded.
- [ ] Add CI checks for catalog validity, key parity/approved fallback,
      duplicate or unused keys, interpolation consistency, and required
      translation completeness.
- [ ] Add browser-level coverage for locale selection persistence, reload,
      route navigation, auth, representative critical workflows, and
      unsupported-locale fallback.
- [ ] Confirm production builds include only supported locale catalogs and
      record bundle impact; define lazy loading only if measurements justify it.
- [ ] Document translation contribution, review, release, and deprecation
      workflow, including how removed keys and obsolete locale choices are
      handled.
- [ ] Verify source-locale changes cannot silently leave translated catalogs
      appearing complete; expose coverage to maintainers.
- [ ] Record a release checklist and rollback behavior for catalog regressions.

**Exit criteria**

- [ ] Supported locale catalogs pass CI completeness and formatting checks and
      have recorded human review.
- [ ] Production build and browser checks pass for critical admin workflows.
- [ ] Translation ownership, contribution instructions, fallback behavior,
      and release/rollback procedures are documented.

**ChatGPT prompt — Phase 5**

```text
Complete Phase 5 of the MyOTA admin UI i18n roadmap. Inspect the current
implementation, locale catalogs, CI workflows, browser tests, build output,
and documentation. Do not mark a locale supported based only on key parity:
require a recorded human linguistic review for all user-visible messages.

Add or finalize CI validation for catalog syntax, key parity or approved
fallback, duplicate/unused keys, interpolation/plural consistency, and
required locale completeness. Add browser coverage for locale selection,
persistence, reload, route changes, authentication, critical admin tasks,
and unsupported-locale fallback. Measure catalog bundle impact and only
introduce lazy loading if evidence shows it is warranted.

Document translation authoring/review ownership, source-string changes,
obsolete keys, locale retirement, support status, and release/rollback
procedures in the admin web and myota-docs. Run the documented release checks,
record exact results and reviewer evidence, and report any launch blockers.
Do not claim an incomplete or unreviewed locale is supported.
```

## Release checklist

- [ ] Source locale and supported locales are explicitly listed and reviewed.
- [ ] Locale choice persists predictably and has a safe fallback.
- [ ] Critical routes have no missing, raw, or blank messages.
- [ ] Locale formatting preserves UTC/API and domain semantics.
- [ ] Accessibility and responsive checks cover the language selector and
      representative critical workflows.
- [ ] Content entered through the programme Content workflow remains
      programme-owned and unchanged by interface locale selection.
- [ ] CI, browser verification, human review, bundle impact, and rollback
      evidence are recorded.
