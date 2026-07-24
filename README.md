# Grafana alerts

This repository contains Grafana-managed alert rules and notification resources
for the Aurora platform. Argo CD renders the Helm chart at the repository root,
and the Grafana Operator reconciles the resulting resources.

The chart follows Grafana's alerting resource model. Folders organize rule
groups, alert labels select notification routes, routes reference contact
points, and contact points read credentials from Kubernetes Secrets. These
collections are independent so adding a team or domain does not require copying
or changing Helm templates.

The root notification policy uses Grafana's built-in `empty` receiver as its
safe fallback. Grafana 12.4 and newer provide this no-op receiver; Aurora runs
Grafana 13.1. Alerts that do not match an enabled route therefore send no
notification.

## Repository layout

```text
alerts/<folder-key>/<group-name>.json       Grafana UI rule-group exports
templates/folders/                          GrafanaFolder resources
templates/rule-groups/                      GrafanaAlertRuleGroup resources
templates/contact-points/                   Provider-specific contact points
templates/notification-policy-routes/       Label-driven notification routes
templates/mute-timings/                     Reusable active/mute intervals
templates/notification-templates/           Shared message templates
templates/secrets/                           Optional AVP-backed Secrets
ci/test-values.yaml                          Test-only Trivy render fixture
```

## How to use

Normal Day 2 alert changes do not require edits under `templates/`.

| Task | Where to make the change |
| --- | --- |
| Add or update an alert in an existing domain | Export JSON to `alerts/<folder-key>/` |
| Change recipients, credentials, or environment-specific schedules | Deployment overrides in the platform `config.yaml` |
| Change shared routing, resource names, or schedule defaults | `values.yaml` in this repository |
| Add a new notification domain | Defaults in `values.yaml` and deployment-specific overrides in the platform repository |
| Change the GC Notify payload or add a provider | Helm templates and supporting values |

Configuration is split by ownership:

- `values.yaml` defines resource names, Secret key contracts, provider settings,
  routing behavior, schedules, and other shared chart defaults.
- `ci/test-values.yaml` contains non-production values used only to exercise the
  complete chart in CI.
- The platform `config.yaml` selects the Grafana instance and datasource, enables
  the resources needed by that deployment, and supplies recipients, template IDs,
  and AVP placeholders for credentials.

Keeping shared behavior here makes alert rules and their notification setup
reviewable together. The platform repository remains responsible for where the
chart runs and for values that differ by environment. A deployment override is
only needed when it intentionally differs from these defaults.

### Add or change an alert group

1. Create or modify the Grafana-managed alert in a non-production Grafana
   instance.
2. Give every rule a stable, unique `uid`.
   Ensure it also has an explicit pending period (`for`); use `0s` for an
   immediate-fire rule. Add a non-empty `description` annotation; it is used as
   the GC Notify message body.
3. Add the routing label configured by `notificationPolicy.routingLabel`. The
   default is `notification_route`; its value must match one enabled route's
   `matcherValue`.
4. Configure the rule to use notification policies rather than a direct contact
   point. Exports containing rule-level `notification_settings` are rejected.
5. From the grouped Alert rules view, export the complete rule group as JSON.
6. Save the export as `alerts/<folder-key>/<group-name>.json`. Each file must
   contain exactly one `groups[]` entry, and its folder title must match the
   corresponding `folders` entry.
7. Ensure all non-expression queries use the stable `datasourceUid` configured
   by the chart. The default is `prometheus`.
8. Do not include credentials, webhook URLs, recipients, Vault paths, tenant
   identifiers, or other environment-specific data in an alert export.
9. Validate the result locally:

   ```bash
   find alerts -name '*.json' -print0 | xargs -0 -n1 jq empty
   helm lint .
   helm lint . -f ci/test-values.yaml
   helm template grafana-alert-resources . \
     --namespace grafana-system \
     -f ci/test-values.yaml
   ```

The chart uses file and directory names only for Kubernetes resource identity.
Grafana displays the exported group name and preserves its rule/query model.
Changes made directly to Operator-managed resources in Grafana may be
overwritten during reconciliation.

### Add a notification domain

Add values for the domain without changing templates:

1. Add a folder under `folders` if the domain needs one.
2. Add or reference a Secret under `managedSecrets`. Leave it disabled when an
   external secret controller owns the referenced Secret.
3. Add a contact point under `contactPoints`.
4. Add a route under `routes` and point `receiverRef` at the contact-point key.
5. Add reusable entries under `timeIntervals` and reference their keys from
   `activeTimeIntervalRefs` or `muteTimeIntervalRefs`.
6. Add a representative enabled configuration to `ci/test-values.yaml` when the
   new domain contains alert exports that CI must render.

The currently supported contact-point provider is `gc-notify-webhook`.
Additional providers should be implemented as explicit templates so their
settings and Secret references remain clear.

## Validation

CI performs three checks that protect the deployed alert configuration:

1. Alert exports contain valid JSON, unique rule UIDs, and no credential fields.
2. Helm lints the chart and renders every alert export using
   `ci/test-values.yaml`.
3. Kubeconform checks the rendered Kubernetes resources against the standard
   Kubernetes schemas and the Grafana Operator schemas from the public CRD
   catalog.

CI does not independently validate alerts. Alerts are expected to be
created and tested in a non-production Grafana instance before the rule group is
exported. This keeps the repository workflow focused on validating the exported
JSON, Helm rendering, and Kubernetes resource structure.

## Private notification configuration

The consuming Argo CD Application renders this chart through the AVP Helm
plugin. Private values supply recipients, GC Notify template IDs, and Secret
data as AVP placeholders. AVP resolves the placeholders after Helm renders the
chart, so credentials are not committed to this repository.

The Trivy webhook targets the GC Notify email API. Its template must contain
`((name))` and `((message))` personalisation fields. Configure authorization
with the `ApiKey-v1` scheme. The API key and webhook URL are read from the
contact-point Secret.

A private Git repository is not a secret store. Do not provide literal secret
values in Git values files; use Vault placeholders or a cluster-side external
secret controller.

Helm validates alert exports and required values used by enabled resources.

CI rejects common credential-bearing keys in rule exports as an accidental
secret-disclosure guard. This is a focused policy check, not a replacement for
repository-level secret scanning.