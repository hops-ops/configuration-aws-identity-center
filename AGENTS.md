# AWS Identity Center Config Agent Guide

This repository publishes the `IdentityCenter` configuration package. Use this guide any time you modify schemas, templates, docs, CI, or release automation.

## Repository Layout

- `apis/`: XRD (`definition.yaml`), composition, and package metadata. Changes here define the public contract.
- `examples/`: Renderable `IdentityCenter` specs. Keep them minimal and refresh whenever the schema changes.
- `functions/render/`: Go-template pipeline executed by `up composition render`. Files execute lexically—reserve `00-` for common variables, `10-` for Identity Store principals, `20-` for permission sets, and keep `90-`/`99-` reserved for observed values + status.
- `tests/`: KCL-based regression tests (`up test`). Add focused assertions when introducing new behaviour.
- `.github/` & `.gitops/`: CI + GitOps workflows. Maintain structural parity between them; only adjust repo-specific defaults such as image names or registry references.
- `_output/` & `.up/`: generated artefacts. `make clean` removes them before a fresh build.

## Contract Overview

`apis/identitycenters/definition.yaml` defines the namespaced `IdentityCenter` XRD:

- `spec.organizationName` defaults to `metadata.name` and feeds naming/tagging.
- `spec.managementMode` toggles self vs managed operation. `spec.managementPolicies` still defaults to `["*"]` and fans out to every rendered resource.
- `spec.forProvider.rootAccountRef` points at the `Account` composite that created the customer's AWS organization. Provider configs default to this name unless the caller overrides `spec.providerConfigName` or the per-block providerConfig values.
- `spec.forProvider.identityCenter` carries the Identity Center instance ARN plus optional relay state, session duration, and tag overrides.
- `spec.forProvider.identityStore` sets the provider config + identity store ID when rendering groups/users.
- `spec.forProvider.groups[]` / `localUsers[]` declare Identity Store principals. Users can list the group names they should join; the templates emit `GroupMembership` resources automatically.
- `spec.forProvider.permissionSets[]` describes the permission set along with inline policies, managed policy ARNs, customer-managed policy references, and assignment intent (`assignToAccounts`, `assignToGroups`, `assignToUsers`). A legacy `assignments[]` block exists for bespoke tuples.
- `spec.externalIdP` (reserved for Authentik/Okta) is defined but currently a no-op.
- Status surfaces the resolved management mode, Identity Center instance ARN, identity store object IDs, and assignment metadata so platform teams can trace readiness from `kubectl get`.

When introducing new schema knobs, update the XRD, README, examples, tests, and templates in the same change.

## Rendering Guidelines

- Gather all shared values in `functions/render/00-desired-values.yaml.gotmpl`. Default aggressively using `default`, `merge`, etc., so later templates never dereference nil values.
- Mirror the pipeline outlined in `docs/plan/03-identity-center.md`: `00-desired-values`, `10-observed-values`, `20-groups`, `30-users`, `40-permission-sets`, `50-account-assignments`, `60-external-idp`, `98-usages`, `99-status`. Leave plenty of numbering gaps for future growth.
- Only render Identity Store resources when an `identityStoreId` is present. Users without an identity store block should still get permission sets and assignments (use external identifiers).
- Generate `identitystore.aws.m.upbound.io/v1beta1, Kind=GroupMembership` resources whenever a local user references one or more groups. Use `groupIdRef` and `memberIdRef` so Crossplane resolves the IDs.
- Use `setResourceNameAnnotation` to assign stable logical names (`identity-center-user-<name>`, `permission-set-<name>`, etc.). Observed-state and Usage gating rely on these annotations.
- Always include `managementPolicies` and `providerConfigRef.kind: ProviderConfig` on managed resources to stay compliant with Crossplane 2.0 expectations.
- Merge caller-supplied tag maps with the default `{"hops": "true", "organization": <name>}` map before applying them to permission sets and propagated resources.

## Testing

`tests/test-render/main.k` currently covers two scenarios:

1. **Minimal** – renders starter groups + users, confirms `GroupMembership` resources are emitted, and checks the admin permission set plus group-based account assignment.
2. **Inline policy & custom policies** – verifies inline policy wiring, customer-managed policy attachments, and explicit user assignments that bypass group references.

Use additional examples under `examples/identitycenters/` plus new assertions when adding behaviour.

Run `make test` (or `up test run tests/*`) after touching templates or schema. Tests should focus on behaviour—assert only the fields that should never change.

## Tooling & Automation

- `make render`, `make render-all`, `make validate`, `make test`, `make publish tag=<version>` mirror other configuration repos.
- `.github/workflows` and `.gitops/` both use `unbounded-tech/workflows-crossplane` v0.8.0. Update both locations together if you bump versions.
- Renovate (`renovate.json`) follows the standard template; extend it here if you need custom behaviour.

## Provider Guidance

Use the `crossplane-contrib` providers defined in `upbound.yaml`. Avoid the Upbound-hosted configuration packages—they now require paid accounts and conflict with our OSS-first workflow. Repeat this reminder in every repo-level `AGENTS.md` you touch.
