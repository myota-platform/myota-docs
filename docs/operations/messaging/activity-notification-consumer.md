# Activity domain-event notification consumer

**Status:** Phase 3 consumer deployed and verified on 10 October 2026. The
Activity service owns the notification projection and its `myota_activity`
deduplication state. The
contracts registry owns the supported event set. Helm/Fleet deployment owns
provisioning and retirement of broker durables. PostgreSQL remains authoritative;
JetStream is bounded delivery infrastructure.

## Consumer contract

| Property | Deployed behavior |
|---|---|
| Consumer group / durable | `activity-notifications-v1` on `MYOTA_EVENTS`; stable across releases. |
| Filter | The 21 exact v1 subjects in `myota-contracts/contracts/event-registry.json`: 19 Identity facts and Geodata `entity.reviewed` / `entity.status-changed`. The Contracts test compares the registry, Activity adapter, and relay validator. |
| Delivery | Pull; explicit ACK; `DeliverPolicy.ALL`; instant replay; 60-second initial ack wait; 8 maximum deliveries; 64 max ack pending; 32 waiting pulls; one fetched message per worker at a time. Same-group replicas share this durable. |
| Retry | Handler/DB errors receive delayed NAKs at 5, 15, 45, 135, then 300 seconds, bounded by eight deliveries. Ack-timeout backoff starts at 60 seconds and is capped at 300 seconds. |
| Local side effect | Activity creates a notice only when the approved Identity/Geodata event resolves to a recipient. Stored notice content is limited to event type and event ID; the source payload is not copied into the notice. |
| Idempotency | The notice insert uses the unique `event:<eventId>` key in Activity PostgreSQL. `consumer_processed_event` and `consumer_checkpoint` commit before ACK. If ACK is lost after the notice transaction, the notice key and processed-event row suppress duplicate effects. |
| Shutdown | SIGTERM stops new fetches, unsubscribes the pull subscription, and drains the NATS connection. Helm grants a 60-second termination window. |
| Observability | Operations samples JetStream pending, ack-pending, redelivery, waiting, and oldest-message age read-only. The consumer exports `myota_activity_notification_outcomes_total` with bounded outcome labels and the database-backed `myota_activity_notification_unresolved_dead_letters` gauge. Grafana displays outcomes and unresolved dead letters; Prometheus alerts on retries and any unresolved poison event. |

Identity and Geodata own the source records and event semantics. Activity may
create only its local notification projection; it does not mutate source data.
Programme has no selected Activity subscription. Operations remains a
read-only JetStream metadata observer.

## Durable rollout and retention

The deploy-owned pre-upgrade Helm hook creates and validates
`activity-notifications-v1` before the Activity Deployment rolls. The
post-upgrade hook validates that successor and then removes only the known
catch-all Activity durables `activity-notifications-pull-v1` and
`activity-notifications`. It fails closed if either name has an unexpected
filter or acknowledgement policy. The successor can deliver messages still
retained when it is created and all later matching facts. Interest retention
may already have deleted facts acknowledged by the old durable; this transition
does not recover those messages or provide historical replay.

The shared `MYOTA_EVENTS` stream remains a mixed, file-backed Interest-retained
stream during this phase. This work narrows Activity's durable and does not
claim that the Limits-retained target stream or separate work streams have been
deployed. JetStream's bounded window is not an event archive; use service-owned
database state for longer-term reconstruction.

## Poison events and reviewed redrive

After terminal failure, Activity stores a redacted envelope, event identity,
subject, stream name/sequence, delivery count, and a generic error category in
`dead_letter_event`, then terminates that delivery. Credential, email, address,
phone, and network-address fields are redacted from the stored diagnostic. Raw
exception text and message payloads are not logged. The bounded outcome metric
and Alertmanager rule notify Operations.

After the cause is fixed, an operator with access to the Activity pod can
republish the persisted envelope. The command requires an actor and reason;
the Activity database records the audit before publish and marks it published
after the JetStream acknowledgement. The replay uses a new transport message ID
to avoid the short broker deduplication window while preserving `eventId`, so
the consumer's database idempotency remains authoritative.

```sh
k3s kubectl exec -n myota deploy/myota-activity-notifications -- \
  python3 redrive_notification_dead_letter.py \
  --event-id <event-uuid> \
  --actor <operator-id> \
  --reason "<reviewed remediation and replay reason>"
```

If publish fails after the audit row is committed, rerunning the same command
reuses the pending audit ID and transport message ID. A malformed envelope
without a valid event ID is deliberately not republished by this tool; recover
it from the owning service database and record the incident disposition.

For bulk fact replay, use a separate durable with an explicit start position as
described in the [JetStream recovery runbook](jetstream-recovery.md). Never
rewind this production side-effecting durable.

## Evidence and source ownership

- Activity authority: `myota-activity-service/event_consumer.py`,
  `activity_notification_consumer.py`,
  `activity_notification_topology.py`,
  `redrive_notification_dead_letter.py`, and
  `migrations/006_notification_consumer_replay.sql`.
- Deploy authority: `myota-deploy/deploy/helm/myota/templates/activity-notification-topology.yaml`,
  `deploy/helm/myota/templates/deployment.yaml`, and
  `observability/otel-collector.yaml` / `observability/rules.yml`.
- Contracts authority: `myota-contracts/contracts/event-registry.json` and
  `tests/test_event_registry.py`.
- `myota-platform` and `myota-deploy/services/` copies are synchronized mirrors
  where the integration workspace carries service code.
- Verification results and the live durable inventory are recorded in the
  [Phase 3 evidence record](evidence/phase3-domain-consumers-2026-10-10.md).
