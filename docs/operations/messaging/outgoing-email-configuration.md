# Outgoing email configuration

This page documents the MyOTA Next Helm chart's SMTP settings and runtime
Secrets. It covers outbound delivery by the `email-worker`; it does not configure
inbound mail, DNS records, or the SMTP provider account itself. Configure the
worker only in an environment with the Identity service credential, Core
database, NATS work stream, and reviewed queue capacity ready.

The worker is disabled by default. Set
`components.email-worker.enabled: true` only after those dependencies and
Secrets are present. In `myota-test`, use the test SMTP account and synthetic
recipients. The SMTP worker's delivery and retry qualification is tracked by
[myota-next #13](https://github.com/myota-platform/myota-next/issues/13).

## Non-secret Helm settings

Set the following in the environment-specific values file. Keep actual SMTP
passwords and service credentials out of Helm values.

```yaml
components:
  email-worker:
    enabled: true

dependencies:
  smtp:
    senderAddress: no-reply@example.org
    senderDisplayName: MyOTA
    host: smtp.example.org
    port: '587'
    username: myota-outbound
    tlsMode: starttls
    timeoutSeconds: '10'
```

Replace the example sender, host, and username with values authorized by the
SMTP provider. `senderAddress`, `host`, and `username` must be non-empty before
the worker starts. `senderDisplayName` defaults to `MyOTA`, port defaults to
`587`, TLS mode defaults to `starttls`, and timeout defaults to 10 seconds.
Supported TLS modes are `starttls` and `implicit`; plaintext SMTP is rejected.
The timeout must be from 1 through 30 seconds. For implicit TLS, use the port
required by the provider, commonly 465.

The chart currently keeps `dependencies.smtp.username` in Helm configuration
and reads only the SMTP password from a Kubernetes Secret. If the provider
classifies the username as a secret, keep the environment-specific values file
private and do not commit it; moving the username to Secret-backed chart
configuration requires a chart change.

## Kubernetes Secrets

Create or reconcile these namespace-local Secrets before enabling the worker.
Use an approved secret manager or permission-restricted files outside the
checkout. Never put secret values in Git, chart values, ConfigMaps, command
arguments, shell history, logs, traces, issue comments, or this document.

- `myota-auth`: `email-service-client-secret`, shared by Identity and the Email
  worker. The application also requires its existing `signing-key` entry.
- `myota-smtp`: `password`, used for the SMTP login.
- `myota-database-urls`: `core-database-url`, used for the Email worker's
  delivery ledger in the Core database.
- `myota-nats-credentials`: `url`, used for the worker's JetStream connection.

`myota-auth` also contains `signing-key` for the application. Preserve all
existing required keys when updating it. The SMTP username is the Helm value
above; it is not a key in `myota-smtp` in the current chart.
The shared `email-service-client-secret` must be generated with at least 32
characters and have the same value in Identity and the Email worker.

For a non-public or private SMTP certificate authority, create an additional
Secret such as `myota-smtp-ca` containing the public CA certificate as
`ca.crt`. Mount it read-only into the Email worker and set
`NODE_EXTRA_CA_CERTS` to the mounted certificate path. The chart supports this
through the component's `extraVolumes`, `extraVolumeMounts`, and `env` values;
an example values block is:

```yaml
components:
  email-worker:
    extraVolumes:
      - name: smtp-ca
        secret:
          secretName: myota-smtp-ca
          items:
            - key: ca.crt
              path: ca.crt
    extraVolumeMounts:
      - name: smtp-ca
        mountPath: /etc/myota/smtp-ca
        readOnly: true
    env:
      NODE_EXTRA_CA_CERTS: /etc/myota/smtp-ca/ca.crt
```

Do not place a private key in the worker. Publicly trusted SMTP certificates
normally need no extra CA Secret. Keep certificate renewal and worker restart
in the operational procedure.

Example commands below read values from files rather than exposing them as
arguments. Adapt the namespace and file paths to the environment. Creating a
Secret containing only the SMTP password must not replace the existing
`myota-auth`, database, or NATS Secrets.

```sh
kubectl -n myota-test create secret generic myota-smtp \
  --from-file=password=/secure/path/smtp-password \
  --dry-run=client -o yaml | kubectl apply -f -

kubectl -n myota-test create secret generic myota-smtp-ca \
  --from-file=ca.crt=/secure/path/smtp-public-ca.crt \
  --dry-run=client -o yaml | kubectl apply -f -
```

Provision or update the shared `myota-auth`, database, and NATS Secrets using
the cluster's approved secret-management process, preserving their other
required keys. Disable shell tracing when handling secret material. Do not use
`--from-literal`, `helm --set`, or checked-in values files for credentials.

If the approved queue topology is provisioned by this chart, review capacity
and set `jetstreamProvisioner.confirmedCapacityReview: true` plus finite values
for all four fields: `jetstreamProvisioner.capacity.emailWorkMaxAgeSeconds`,
`jetstreamProvisioner.capacity.emailWorkMaxBytes`,
`jetstreamProvisioner.capacity.emailWorkMaxMessages`, and
`jetstreamProvisioner.capacity.emailWorkMaxMessageBytes`. The provisioner
requires all four; do not enable email against an unbounded or unreviewed
stream. The Core migrations must also have created the email delivery ledger
before the worker starts.

After rotating an environment variable Secret, restart the affected
Deployments so their processes read the new value. Rotate the provider
credential at the provider as well as in Kubernetes. Use distinct test and
production SMTP accounts and Secret values.

## Network, trust, and sender prerequisites

- Allow egress from the Email worker to the configured SMTP host and port,
  Core PostgreSQL, NATS, and the Identity internal API.
- The SMTP host certificate must match the configured hostname and chain to a
  trusted root. Use `NODE_EXTRA_CA_CERTS` only for an approved additional CA;
  do not disable TLS certificate verification.
- The sender address must be authorized by the provider. Configure SPF, DKIM,
  and DMARC for the sending domain according to the provider's instructions;
  these DNS records are outside Helm and this chart.
- The queue stream and durable must be provisioned with reviewed finite limits
  before enabling the worker. The worker must be able to reach the Identity
  recipient-resolution API using the shared email-service credential.
- The Helm chart's opt-in JetStream provisioner requires the four
  `emailWork*` capacity values documented above; its default is disabled.
- SMTP delivery is at least once. A worker crash after provider acceptance but
  before recording delivery may result in a duplicate message.

## Verification

After deploying through the Helm-backed target environment, verify that the
Email worker is Ready, TLS connection succeeds, an allowlisted test template
arrives at a synthetic recipient, Core records the delivery outcome, and
replaying the same work ID does not create another delivery. Also verify
connection and delivery failure, retry, and notification outcomes. Do not send
test messages to real participants. See the [email delivery operations
record](email-delivery.md) for delivery guarantees and privacy boundaries.
