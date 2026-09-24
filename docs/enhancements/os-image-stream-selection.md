---
title: os-image-stream-selection
authors:
  - "midu@redhat.com"
reviewers:
  - "sakhoury@redhat.com"
  - "imiller@redhat.com"
  - "sahasan@redhat.com"
approvers:
  - "sakhoury@redhat.com"
  - "imiller@redhat.com"
  - "sahasan@redhat.com"
api-approvers:
  - "TBD"
creation-date: 2026-09-24
last-updated: 2026-09-24
status: provisional
tracking-link:
  - "https://issues.redhat.com/browse/ACM-46094"
see-also:
  - "/docs/enhancements/README.md"
replaces:
  - "None"
superseded-by:
  - "None"
---

# OS Image Stream Selection

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria are defined
- [ ] User-facing documentation is updated

## Summary

This enhancement adds an optional `osImageStream` field to the `ClusterInstance`
spec. When set, the value is rendered as the `osStream` field of the
`AgentClusterInstall` consumed by the Assisted Installer, which selects the OS
image matching the cluster's OpenShift version for that stream (e.g. `rhel-9`,
`rhel-10`). When omitted, the Assisted Installer falls back to the default OS
stream for the OpenShift version referenced by `ClusterImageSetNameRef`,
preserving today's behavior.

## Motivation

Today the OS image used to install a cluster is always the default stream for
the OpenShift version selected by the referenced `ClusterImageSet`. Operators
who need to pin a cluster to a specific OS stream (for example, to standardize
on `rhel-10` across a fleet, or to test a newer OS stream before it becomes the
default) have no way to express that intent through the `ClusterInstance` API.
They are forced to either patch the rendered `AgentClusterInstall` directly or
maintain a separate `ClusterImageSet` per OS stream, both of which are fragile
and hard to reason about.

### User Stories

As a cluster operator, I want to declare the OS stream on my `ClusterInstance`
so that the installed cluster uses the OS image I intend, without patching
rendered resources.

As a fleet administrator, I want to standardize a group of clusters on a
specific OS stream (e.g. `rhel-10`) so that OS-level behavior is consistent
across the fleet.

As a day-2 operator, I want to change the OS stream of an existing
`ClusterInstance` so that a reinstallation or re-provisioning picks up the new
stream.

Day-2 operations considerations:

- **Lifecycle**: The field is a plain spec field with no finalizer impact. It
  does not interact with deletion or sync-wave ordering. On reinstallation, the
  rendered `AgentClusterInstall` reflects the current value of the field.
- **Monitoring**: No new status conditions are introduced. The rendered
  `AgentClusterInstall` (visible via the existing `RenderedTemplatesApplied`
  condition and the rendered manifests) is the source of truth for which stream
  was applied. Structured logs from the template engine are sufficient for
  debugging.
- **Remediation**: If an invalid stream is set, the admission webhook rejects
  the `ClusterInstance` with a clear validation error, so a bad value never
  reaches the rendering pipeline. To recover, the operator corrects the field
  and re-applies.
- **Scale**: The change adds a single optional string field and one conditional
  template branch. It has no measurable impact on `maxConcurrentReconciles` or
  on behavior with many `ClusterInstance`s.

### Goals

- Allow operators to select the OS stream for a cluster via a first-class
  `ClusterInstance` field.
- Keep the change fully backward compatible: omitting the field preserves the
  existing default-stream behavior.
- Validate the field at admission time so invalid values are rejected early
  with actionable error messages.
- Allow the field to be updated after provisioning so day-2 re-provisioning can
  change the stream.

### Non-Goals

- Selecting a specific OS image version or checksum; only the stream name is
  selected, and the Assisted Installer resolves the concrete image.
- Changing the default OS stream for a `ClusterImageSet` or OpenShift version.
- Supporting OS streams for installation flows other than the Assisted
  Installer in this change.
- Validating that the stream actually exists for the referenced OpenShift
  version (that is the Assisted Installer's responsibility at install time).

## Proposal

### Workflow Description

1. The operator sets `spec.osImageStream` on a `ClusterInstance` (e.g.
   `rhel-10`). The value is optional.
2. The admission webhook validates the field: it must be absent or a
   well-formed stream name (starts with a lowercase alphanumeric character,
   followed by lowercase letters, digits, dots, underscores, or dashes, and at
   most 63 characters). Invalid values are rejected with a descriptive error.
3. The `ClusterInstanceReconciler` renders the Assisted Installer templates.
   When `osImageStream` is set, the `AgentClusterInstall` template emits an
   `osStream` field with the configured value; when it is unset, the field is
   omitted entirely.
4. The Assisted Installer consumes the `AgentClusterInstall` and selects the OS
   image matching the cluster's OpenShift version for the requested stream, or
   the default stream when `osStream` is absent.

Error handling: an invalid `osImageStream` is rejected at admission, so the
reconciler never observes a malformed value. If the referenced stream does not
exist for the target OpenShift version, the Assisted Installer surfaces the
failure during installation, which is observable through the existing
`AgentClusterInstall` status and events.

### API Extensions

A new optional field is added to `ClusterInstanceSpec`:

- `osImageStream` (`string`, optional): the OS stream to use when installing
  the cluster (e.g. `rhel-9`, `rhel-10`). Rendered as the `osStream` field of
  the `AgentClusterInstall`. If omitted, the default OS stream for the OpenShift
  version referenced by `ClusterImageSetNameRef` is used.

The field is constrained by a CRD pattern
(`^[a-z0-9][a-z0-9._-]*$`) and by webhook validation that additionally enforces
a maximum length of 63 characters. The corresponding CRD manifests
(`config/crd/bases/...` and `bundle/manifests/...`) are regenerated to include
the new field.

### Siteconfig Impact

- **Controllers**: No controller logic changes. The `ClusterInstanceReconciler`
  is unaffected; the change flows through the existing template rendering
  pipeline.
- **Templates**: The Assisted Installer `AgentClusterInstall` template is
  modified to conditionally render the `osStream` field when
  `spec.osImageStream` is set.
- **API fields**: A new optional `osImageStream` field is added to
  `ClusterInstanceSpec`, plus a `GetOSImageStream()` accessor on the spec.
- **Validation**: A new `validateOSImageStream` check is added to
  `ValidateClusterInstance`, and `/osImageStream` is added to the list of
  fields allowed to change after provisioning.

### Implementation Details/Notes/Constraints

- The template uses a `{{ if .Spec.OSImageStream }}` guard so that the
  `osStream` key is omitted (not rendered as an empty string) when the field is
  unset, preserving the Assisted Installer's default-stream fallback.
- The webhook length check (63 characters) complements the CRD pattern, which
  cannot express a maximum length.
- The field is intentionally allowed in post-provisioning updates so that a
  re-provisioning can switch streams without recreating the `ClusterInstance`.

### Risks and Mitigations

- **Risk**: An operator sets a stream that does not exist for the target
  OpenShift version, causing installation to fail. **Mitigation**: This is
  surfaced by the Assisted Installer at install time through the
  `AgentClusterInstall` status and events; the field is validated for format at
  admission, and the default (unset) behavior is unchanged.
- **Risk**: A malformed stream name reaches the rendered manifest.
  **Mitigation**: Both the CRD pattern and the webhook validation reject
  malformed values before they are rendered.
- **Risk**: Behavior change for existing clusters. **Mitigation**: The field is
  optional and defaults to the current behavior, so existing `ClusterInstance`s
  are unaffected.

### Drawbacks

- The field name and semantics must stay in sync with the Assisted Installer's
  `osStream` concept; a divergence would require coordinated changes.
- Operators may set a stream that is valid in format but unsupported for their
  OpenShift version, which only fails at install time rather than at admission.
  This is an accepted trade-off to avoid duplicating the Assisted Installer's
  version-to-stream knowledge in siteconfig.

## Design Details

### Open Questions

- Whether to later add admission-time validation that the stream exists for the
  referenced OpenShift version. Deferred; the Assisted Installer is the source
  of truth for that mapping.

### Test Plan

- **Unit tests (API validation)**: `ValidateClusterInstance` returns nil when
  `osImageStream` is unset or a valid stream name; it returns an error for an
  invalid format, a value starting with an invalid character, and a value
  exceeding the maximum length. `validatePostProvisioningChanges` returns nil
  when only `osImageStream` changes.
- **Unit tests (template rendering)**: The `AgentClusterInstall` template
  renders the `osStream` field with the configured value when `osImageStream`
  is set, and omits the field entirely when it is unset.
- **CRD manifests**: The regenerated CRD manifests are verified to include the
  new field with the correct pattern and description.

### Graduation Criteria

- [ ] Design reviewed and approved by maintainers
- [ ] Implementation merged with adequate test coverage
- [ ] Documented in user-facing docs
- [ ] Released in version X.Y.Z

### Upgrade / Downgrade Strategy

Upgrading the siteconfig operator regenerates the `ClusterInstance` CRD to
include the new optional field. Existing `ClusterInstance`s that do not set the
field continue to work unchanged. No manual steps are required on upgrade. On
downgrade to a version without the field, the CRD no longer advertises
`osImageStream`; any `ClusterInstance` that set the field should have the field
removed before downgrade to avoid the field being pruned or rejected.

### Version Skew Strategy

The field is additive and optional. A newer siteconfig rendering a
`ClusterInstance` that sets `osImageStream` produces an `AgentClusterInstall`
with an `osStream` field; an Assisted Installer that does not understand
`osStream` ignores it and uses its default stream, so there is no hard
dependency. A `ClusterInstance` that does not set the field behaves identically
across versions.

### Operational Aspects of API Extensions

The change adds an optional field to an existing CRD and does not introduce a
new webhook, aggregated API server, or finalizer. It has no measurable impact
on API throughput, scalability, or availability.

#### Failure Modes

The only new failure mode is an invalid `osImageStream` value, which is rejected
at admission and therefore does not affect cluster health. A stream that is
valid in format but unsupported for the target OpenShift version fails during
installation and is surfaced by the Assisted Installer; escalation goes to the
team operating the Assisted Installer.

#### Support Procedures

To detect an admission rejection, inspect the `ClusterInstance` apply error,
which includes the validation message. To detect an install-time stream failure,
inspect the `AgentClusterInstall` status and events. To disable the feature,
remove the `osImageStream` field from the `ClusterInstance`; the Assisted
Installer then falls back to the default stream.

## Implementation History

- 2026-09-24: Initial proposal drafted alongside the implementation on branch
  `ACM-46094`.

## Alternatives

- **Patch the rendered `AgentClusterInstall` directly**: Rejected because it is
  fragile, not declarative, and is overwritten on re-render.
- **Maintain a separate `ClusterImageSet` per OS stream**: Rejected because it
  conflates the OpenShift version selection with the OS stream selection and
  multiplies the number of `ClusterImageSet` objects operators must manage.
- **Add the field to a `ClusterImageSet`**: Rejected because the OS stream is a
  per-cluster installation choice, not an attribute of the image set.

## Infrastructure Needed

None.
