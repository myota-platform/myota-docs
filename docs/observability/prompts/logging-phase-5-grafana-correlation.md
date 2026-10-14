# Phase 5 prompt — Grafana logs and cross-signal correlation

Use this prompt for the remaining Phase 5 tasks in the
[structured logging plan](../logging-implementation.md). Source implementation
is in the monorepo; former repository paths are migration history.

```text
Complete Phase 5 Grafana correlation in myota-platform/myota. Read AGENTS.md,
myota-docs/docs/observability/logging-implementation.md, the logging runbook,
and the latest browser evidence first. deploy/ owns Compose and Helm sources;
myota-docs owns decisions, prompts and evidence. Keep the deployment copies
synchronized and preserve the existing MyOTA, NATS and K3s folders/datasources.

Work only on the remaining Phase 5 checkboxes and exit criteria. Use Loki UID
myota-loki and Tempo UID myota-tempo. Applications emit service.namespace=myota;
do not substitute a Kubernetes namespace into that application resource label.
Use finite service/environment/severity selectors and actual emitted values.
Keep trace, request and correlation IDs as structured metadata, not indexed
labels or dashboard variables. Never log credentials, account addresses,
request bodies or domain payloads.

Verify log-to-trace and trace-to-log actions in an authorized Test Grafana
browser session. Preserve trace IDs through provisioning by escaping dollar
macros. The reverse query must filter trace_id structured metadata, not search
the event body. Confirm the newly opened pane contains the matching record;
a record remaining in another pane is insufficient evidence. Scope queries
by service and time. Use existing sanitized telemetry where possible; remove
all temporary credentials, fixtures and test resources afterward.

Compare the complete dashboard inventory and definitions, including nested
NATS and K3s assets, across Compose and Helm. Preserve no-data behavior when
Kubernetes metrics are unavailable in standalone Compose. Keep alerts and
performance metrics authoritative; do not replace them with log approximations.
Do not expand this phase into Identity schemas, SMTP, or production changes.

Run focused checks and all required PR gates. Follow feature/fix branches and
PRs into testing; verify the approved files are mounted and provisioned before
claiming rollout. Record exact commands/results, source/merge commits, browser
behavior, cleanup and remaining gaps in myota-docs. Check off only verified
items and close the task issue only when every exit criterion has evidence.
Leave capacity, recovery and full runtime/restart qualification to Phase 6
where they are separately tracked. Commit and push related work coherently.
```
