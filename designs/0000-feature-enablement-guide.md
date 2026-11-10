# Contributor Guide: Enabling an Azure or AKS Feature in Karpenter

**Last updated:** November 10, 2026

**Status:** Guidance

## Overview

Use this guide when adding support for an Azure or AKS capability that the provider
does not yet expose. It takes you from a feature proposal to a reviewable implementation,
including scheduling, provisioning, upgrades, and real-Azure evidence.

The same code serves AKS-managed Node Auto Provisioning (NAP) and self-hosted Karpenter.
A change can compile and pass isolated tests while still breaking one deployment model,
one provisioning mode, or existing clusters. This guide helps you find those seams.

Start with [CONTRIBUTING.md](../CONTRIBUTING.md) for development setup, the Contributor
License Agreement (CLA), and the code of conduct. Read [AGENTS.md](../AGENTS.md) for the
architecture and repository conventions. This guide complements those documents; it
does not replace a feature-specific design or maintainer review.

**How to use it:** follow the workflow below, then use the implementation sections that
apply to your feature. Record "not applicable" with a reason rather than making unrelated
changes to satisfy every checklist item. Completing the checklist strengthens a
contribution; it does not guarantee acceptance or authorize a new public API.

### Contents

* [Contributor workflow](#contributor-workflow)
* [Terms and provisioning modes](#terms-and-provisioning-modes)
* [1. Establish the feature contract](#1-establish-the-feature-contract)
* [2. Choose the user-facing surface](#2-choose-the-user-facing-surface)
* [3. Define defaults and validation](#3-define-defaults-and-validation)
* [4. Implement scheduling and cache correctness](#4-implement-scheduling-and-cache-correctness)
* [5. Resolve external state and readiness](#5-resolve-external-state-and-readiness)
* [6. Wire provisioning and resource lifecycle](#6-wire-provisioning-and-resource-lifecycle)
* [7. Prove upgrade, drift, and update behavior](#7-prove-upgrade-drift-and-update-behavior)
* [8. Build the evidence and run checks](#8-build-the-evidence-and-run-checks)
* [9. Plan rollout and recovery](#9-plan-rollout-and-recovery)
* [10. Prepare the pull request](#10-prepare-the-pull-request)
* [Ready-for-review checklist](#ready-for-review-checklist)
* [Appendix: implementation examples and lessons](#appendix-implementation-examples-and-lessons)

## Contributor workflow

| Stage | What to do | Evidence to bring to review |
|---|---|---|
| Discuss | Open or find an upstream issue describing the workload problem, AKS support, and proposed scope. Ask for feedback before committing to a public API. | Problem statement, public platform references, and unresolved questions. |
| Design | Use [the design template](./0000-TEMPLATE.md). Include FAQ, testing, and production readiness even though the template marks some sections optional. | Feature contract, surface choice, compatibility matrix, and lifecycle decisions. |
| Implement | Extend existing helpers and tests. Trace the value from user input to scheduling, Azure payload, and the actual node. | Small, coherent diff covering each claimed path. |
| Verify | Run local checks and arrange relevant Azure-backed tests. Explain unavailable infrastructure early. | Commands, results, tested revision, and remaining gaps. |
| Submit | Follow [the PR template](../.github/pull-request-template.md), link the issue/design, and respond to review with evidence. | User documentation, release note, and the checklist at the end of this guide. |

Use a draft PR for early feedback. Where changes can be reviewed independently, separate
an SDK prerequisite, API decision, and implementation into coordinated PRs; identify
dependencies and keep incomplete behavior from being exposed as supported.

**External contributors:** you do not need access to private AKS systems to propose or
explain a feature. If platform behavior cannot be confirmed from public documentation,
SDK definitions, or a reproducible experiment, flag the uncertainty and ask maintainers
for clarification. Do not invent defaults or present unrun tests as passing.

## Terms and provisioning modes

| Term | Meaning in this guide |
|---|---|
| AKSNodeClass | Provider-specific configuration describing how to create nodes. |
| NodePool | Karpenter policy and scheduling requirements for a group of nodes. |
| NodeClaim | Karpenter's request for, and lifecycle record of, an individual node. |
| SKU / offering | An Azure VM size / a capacity option for a size, placement, and capacity type. Capability can vary by zone. |
| NAP / self-hosted | AKS-managed Node Auto Provisioning / an installation operated by the user. |
| MC / AgentPool | AKS ManagedCluster / node-pool API resource. Neither is the Karpenter NodePool resource. |
| RP / SDK | Azure resource provider accepting API requests / the client library used to send them. An SDK field alone does not establish service support. |
| Bootstrap / NPS | Configuration that prepares the VM as a Kubernetes node / the AKS node-provisioning service used by relevant bootstrap paths. |
| CEL / RBAC | Common Expression Language for CRD validation / role-based access control. Kubernetes RBAC and Azure role assignments are separate permissions. |
| Drift / in-place update | A difference that can require replacement / a supported change to an existing resource without replacement. |

The deployment model and provisioning mode are different dimensions. Do not assume every
combination is available to every contributor or supported by a particular feature.

| Mode | Implementation family | What must be checked |
|---|---|---|
| `aksscriptless` | VM-based | Direct VM properties, image/bootstrap settings, NICs, and cleanup. |
| `bootstrappingclient` | VM-based | VM properties plus the bootstrap service request and response. |
| `aksmachineapi` | AKS Machine API | Machine template/profile, service defaults, and lifecycle. |
| `aksmachineapiheaderbatch` | AKS Machine API | Same behavior as `aksmachineapi`, plus shared-template grouping, per-machine header fields, and partial failure. |

Mode definitions live in [consts.go](../pkg/consts/consts.go); configuration lives in
[operator/options](../pkg/operator/options/). Consult the source at your working revision,
not a mode or version list copied from an older PR.

## 1. Establish the feature contract

Before writing implementation code, answer these in the issue or design:

1. **What user problem is solved?** Include a representative workload, desired node
   behavior, and explicit non-goals. Distinguish enabling a capability from advertising
   that a SKU supports it.
2. **Does the downstream support exist?** Check the dependency version in
   [go.mod](../go.mod), its SDK request types, the API version, and public AKS/Azure
   documentation. Confirm applicable region/cloud availability, preview registration,
   Kubernetes versions, and image prerequisites. Identify any SDK or service prerequisite.
3. **Who owns the setting?** Identify cluster, AgentPool, NodeClass, NodePool, and operator
   inputs. State precedence and behavior when inputs conflict, including NAP versus
   self-hosted configuration.
4. **What do omission and explicit values mean?** Establish AKS AgentPool defaults and
   the behavior of each request layer. Match AKS where possible; justify any intentional
   difference without changing existing-user behavior accidentally.
5. **How does the setting change over time?** State Azure mutability, desired-configuration
   mutability, replacement/in-place behavior, and whether external state can change the
   effective result without a spec edit.
6. **What does failure look like?** Define unsupported combinations, readiness failures,
   exhausted candidate sets, downstream rejection, retries, and cleanup.
7. **What are the costs and recovery limits?** Include pricing, provisioning latency,
   API calls, permissions, disruption, and how enablement can be stopped or reversed.

### Copyable compatibility and defaults matrix

Replace the placeholders with feature-specific values. Split rows wherever behavior
differs; use additional columns for network mode, capacity type, or other relevant inputs.
For enum/configuration features, replace `false` and `true` with their actual values.

| Input | Deployment / mode | Architecture / image / Kubernetes version | Region / placement / SKU prerequisites | Effective behavior and request value | Unsupported behavior | Evidence |
|---|---|---|---|---|---|---|
| Omitted, including existing manifests | Fill in | Fill in | Fill in | State default and omitted vs explicit SDK values | State result | Link source/test |
| Explicit `false` | Fill in | Fill in | Fill in | State result | State result | Link source/test |
| Explicit `true` | Fill in | Fill in | Fill in | State result | State result | Link source/test |
| Invalid or conflicting configuration | Fill in | Fill in | Fill in | State what is rejected and where | Condition/error and remediation | Link source/test |

Record each mode and deployment model as **supported**, **unsupported**, or **not yet
verified**. These are not interchangeable. An untested path is not a supported path
merely because it shares a helper. For temporary platform gaps, link a tracking issue.

> **Never advertise a capability that the provider cannot honor.** A limited initial
> implementation can be reasonable when its scope is agreed and documented, but explicit
> unsupported requests must not silently succeed without enabling the requested feature.

**Exit:** reviewers can understand defaults, scope, lifecycle, and known gaps without
reverse-engineering the proposed implementation.

## 2. Choose the user-facing surface

[Labels vs. AKSNodeClass](./0015-nodeclass-vs-labels.md) is the authority on this decision.

| User question | Likely surface | Implementation route |
|---|---|---|
| "Which nodes can run my workload?" | A well-known scheduling label for a simple capability or placement property. | Scheduling-label route in section 4, then applicable provisioning/lifecycle work. |
| "How should Azure configure this node?" | An AKSNodeClass field, particularly for structured settings or complex validation. | NodeClass route in section 3, then applicable scheduling and provisioning work. |
| "Configure a feature, and schedule against the resulting property." | Consider both only when a single surface cannot model the requirement cleanly. | NodeClass remains authoritative; any label expresses the derived property, not an independent configuration switch. |
| "How should the controller operate globally?" | An operator option if there is no meaningful per-NodeClass setting. | Options, deployment configuration, defaults, and mode/model coverage. |

AKS behavior is a useful precedent, not an absolute requirement: explain deviations
needed for Karpenter scheduling semantics or provider constraints. If the choice is
unclear, ask for design feedback before building both APIs.

**Applicability:** every feature needs a contract, validation strategy, lifecycle
decision, tests, and rollout plan. Only NodeClass features need new spec fields; only
schedulable capabilities need well-known-label registration; only externally resolved
features need status resolution; only mutable resources need in-place patchers.

## 3. Define defaults and validation

### NodeClass route

Add new functionality to
[`pkg/apis/v1beta1/aksnodeclass.go`](../pkg/apis/v1beta1/aksnodeclass.go).
`v1alpha2` is deprecated; do not add fields there solely for parity.

* Make the new top-level setting optional so existing manifests remain valid. Use pointers
  and `+optional` where omission must be distinguishable from a zero value. Decide
  deliberately which nested fields are required when the parent is present.
* Group related settings by domain and reuse existing structures where appropriate.
  Avoid an unrelated API redesign.
* For enums, use a named string type, constants documenting values, and kubebuilder enum
  validation. Document units, ranges, defaults, constraints, and update behavior.
* Centralize effective-value logic in accessors that safely handle omitted parent/child
  settings. Avoid duplicated pointer chains and conflicting defaults at call sites.
* Keep **user intent** distinct from **per-node effective value** when they differ.
  Artifact streaming has an architecture-independent explicit-enable accessor for
  filtering and an architecture-aware accessor for provisioning. Test both contracts.

Defaulting must agree across the schema, Go accessors, design, user documentation,
Azure payload, and tests. An omitted SDK pointer may cause service-side defaulting;
it is not necessarily equivalent to an explicit `false`.

### Choose the enforcement layer

| Layer | Use for | Expected user-visible outcome |
|---|---|---|
| CRD schema and CEL | Object-local types, bounds, enums, and cross-field consistency. | Kubernetes rejects the invalid create/update with an actionable message. |
| Status reconciliation | External dependencies, permissions, and resolved cluster state. | Conditions explain readiness; unresolved required dependencies block provisioning. |
| Instance-type/offering requirements and filters | Architecture, SKU, zone, and workload/NodePool compatibility. | Only valid capacity is eligible; infeasibility is diagnosable. |
| Azure request/error handling | Service-side failures that cannot be prevented locally. | Classified provisioning failure, appropriate retry behavior, and cleanup. |

For runtime validation, inspect the existing
[status ValidationReconciler](../pkg/controllers/nodeclass/status/validation.go).
It already checks Disk Encryption Set permissions; it is not an API admission webhook
or an empty skeleton. Wire additional checks and conditions into the appropriate
controller rather than merely adding an unused validation function.

### CRD and CEL checks

* Put item rules on the item, collection rules on the collection, and cross-field rules
  at a common parent. Ensure omitted optional fields do not cause evaluation errors.
* Use `listType=map` and map keys for keyed-list uniqueness rather than duplicating that
  rule in CEL. Set meaningful size bounds such as `maxItems` to constrain input and cost.
* CEL cannot query a NodePool, Azure SKU, or live cluster resource. Keep those checks
  in scheduling or reconciliation as appropriate.
* Test both create and update behavior, including existing stored objects. Tightening
  validation can break updates to configurations that older versions accepted.
* Exercise installation of the generated CRD in the API test environment. Lint/codegen
  alone does not prove that the API server accepts the schema and CEL cost.

**Exit:** valid values are accepted, invalid values fail at the intended layer, and
omitted settings preserve the agreed behavior.

## 4. Implement scheduling and cache correctness

### NodeClass-dependent filtering

If a setting changes instance-type construction, add the relevant input to
`instanceTypeParameters` in
[instancetypes.go](../pkg/providers/instancetype/instancetypes.go) and populate it in
`List()`. This struct is hashed for the instance-type cache key. Test alternating
NodeClasses with different settings, including a cache hit, so results cannot leak
between configurations.

Reuse `isInstanceTypeSupportedByFilters` for SKU-level constraints. When the feature is
requested, exclude SKUs whose capability is false, absent, or otherwise insufficient.
When it is not requested, do not narrow capacity because of that feature.
Handle malformed capability values deliberately; do not turn an unexpected data error
into a success-shaped default.

Apply zone-dependent constraints when constructing **offerings**, not by globally
discarding a SKU that works in another zone. Cover regional placement explicitly.
Check whether the feature also changes allocatable resources, pricing, or image choice.

### Scheduling-label route

A label emitted onto a node is not automatically supported by scheduling simulation.
Trace the full route:

1. Define the scheduling key in
   [v1beta1/labels.go](../pkg/apis/v1beta1/labels.go) and register it in
   `AzureWellKnownLabels`. For a genuinely stable, finite value domain, follow existing
   `WellKnownValuesForRequirements` registration. Do not enumerate a dynamic SKU domain
   as if it were fixed.
2. Populate `computeRequirements` in
   [instancetype.go](../pkg/providers/instancetype/instancetype.go). For zonal
   capabilities, also populate offering requirements in `instancetypes.go` and ensure
   the instance-type requirements accurately represent available offerings.
3. If a selected label value configures Azure, carry that selection from the NodeClaim
   requirements into both applicable provisioning families. Do not confuse "supports
   enabling" with "will be enabled", or enable a paid capability merely because a SKU
   supports it.
4. Check requirement-to-label projection in
   [cloudprovider.go](../pkg/cloudprovider/cloudprovider.go) and
   `GetWellKnownSingleValuedRequirementLabels` in
   [providers/labels.go](../pkg/providers/labels/labels.go). The observed node label must
   match the selected behavior; test reconstruction through Get/List when applicable.
5. Test Pod selectors/affinity and NodePool requirements through the existing scheduling
   acceptance suite. Include matching and conflicting requests, unsupported values,
   absent capability data, and a case with no feasible offering.

Labels in `pkg/providers/labels/labels.go`, such as LocalDNS and artifact-streaming
state labels, are intentionally separate from well-known scheduling labels. Adding a
key there alone does **not** make it schedulable. Match AKS label-presence conventions:
an absent label and a label with value `"false"` have different selector semantics.

### Cache and failure diagnostics

Document all cached inputs, invalidation triggers, and expected convergence time.
A cache key must distinguish inputs that produce different cached output; external
source changes also need invalidation or a bounded refresh.

Follow [the machine-cache design](./0011-machine-cache.md) when adding Azure reads:
cache suitability differs between drift/pre-create checks and authoritative
post-create, in-place-update, and CloudProvider.Get operations.

If filtering can empty the candidate set, record the failure in the table in
[the diagnostic design](./0012-quota-fungibility-reactivity-improvements.md) and explain
what users should inspect. Add appropriate Azure error classification to the existing
create/error-handling path; only mark offerings unavailable for errors that actually
indicate an offering problem, not for bad feature configuration or missing permissions.

**Exit:** scheduling predicts what can be created, labels describe what was created,
and cache reuse does not change either answer.

## 5. Resolve external state and readiness

Use this section only if the effective configuration depends on cluster state or another
external input. [The LocalDNS reconciler](../pkg/controllers/nodeclass/status/localdns.go)
is an example, not a requirement to copy its API or sticky behavior.

* Define the source of truth and a single owner for resolved status. Wire the reconciler
  through the [status controller](../pkg/controllers/nodeclass/status/controller.go)
  and include the relevant condition in aggregate readiness.
* Ensure provisioning cannot consume unresolved or stale-required state as though it
  were a successful resolution. Read the resolved decision consistently in filtering,
  payload construction, and labels.
* Define a transition table: unresolved, enabled, disabled, dependency error, spec
  change, external-input change, and controller restart. Decide explicitly whether
  resolution is reversible or sticky and whether persistent status is needed.
* Observe external changes through reliable watches or a bounded periodic requeue.
  Include upstream cache freshness and retries in the convergence estimate. Do not copy
  LocalDNS's polling interval without considering your inputs.
* Distinguish an optional API being absent from a permission failure or transient error.
  For dynamic-client probes, test both `IsNotFound` and `meta.IsNoMatchError` as applicable.
  Do not silently disable the feature on an unexpected error.
* Update Kubernetes RBAC in
  [clusterrole-core.yaml](../charts/karpenter/templates/clusterrole-core.yaml) and any
  other applicable role. Separately document Azure permissions and which identity
  needs them for NAP and self-hosted installations. Request the least privilege needed.
* Translate a resolved decision at the wire boundary if sending the original "automatic"
  mode would let the service resolve it differently. `ResolvedLocalDNSForWire()` is
  an example of this translation.

Keep feature-local constants near the reconciler and genuinely shared values in
`pkg/consts`. If checks mirror platform validation, link the public contract and explain
how divergence will be detected. Do not depend on private implementation references.

**Exit:** external changes and errors produce tested, observable transitions with an
explained convergence bound.

## 6. Wire provisioning and resource lifecycle

Trace the selected value to the actual SDK request and the running node. Use the
compatibility matrix to decide which paths need changes. Follow `ATTENTION!!!` comments
that identify counterpart implementations.

### VM-based paths

Start in [vminstance.go](../pkg/providers/instance/vminstance.go).
Allocate nested SDK profiles only when needed and modify only your fields. Replacing a
whole `SecurityProfile`, for example, can clobber another feature's settings.
Test the complete relevant payload, including preservation of sibling fields.

For settings delivered through bootstrap, trace:

`AKSNodeClass / selected requirements -> image-family resolver -> bootstrap values -> node`

Inspect [imagefamily/resolver.go](../pkg/providers/imagefamily/resolver.go) and
[customscriptsbootstrap](../pkg/providers/imagefamily/customscriptsbootstrap/).
`CustomScriptsNodeBootstrapping` is implemented per image family: a signature change
affects the interface, each implementation, resolver calls, and tests. Positional
arguments can compile while carrying the wrong meaning.

Thread service-delivered values through `ProvisionClientBootstrap` and
`ConstructProvisionValues` where applicable. Verify differences in units and names
between kubelet, AKS APIs, and provider fields. If an image family cannot implement the
setting, enforce and document that restriction rather than relying on an ignored argument.

### AKS Machine API paths

Start in
[aksmachineinstancehelpers.go](../pkg/providers/instance/aksmachineinstancehelpers.go).
Configure the appropriate `Machine*Profile` through existing helper patterns. Check
template placeholders as well as helper definitions: an uncalled helper changes nothing.

Document pointer semantics per property. `nil` can mean not applicable, omit, or inherit
a service default; explicit `false` can communicate a decision already made by Karpenter.
Assert on the SDK request, not just an intermediate provider struct. Test enum casts,
duration units and bounds, list-to-map conversions, and preservation of sibling fields.

For header batching, inspect
[aksmachinesheaderbatch](../pkg/providers/azclient/aksmachinesheaderbatch/).
A shared template cannot contain a value that varies between machines in that batch.
Use supported per-machine header fields or adjust grouping so incompatible requests
cannot share a template. Test heterogeneous requests and partial failures; do not assume
every new SDK property has a corresponding header representation.

### All applicable paths

Construct new clients through [azclient](../pkg/providers/azclient/) and wire only the
modes that need them. Cover cancellation, long-running operations, throttling, eventual
consistency, and retry classification using existing client patterns.

For every created Azure resource, identify its owner, cleanup path, and behavior after
partial creation or a lost response. Retries must not leak duplicate resources.
Deletion and garbage collection must tolerate resources already being gone.

**Exit:** each supported path sends the intended request, produces the intended node
behavior, and remains safe to retry or delete after partial failure.

## 7. Prove upgrade, drift, and update behavior

Azure resource immutability and NodeClass API immutability are different decisions.
An Azure create-only property may still be editable in desired configuration, with new
nodes receiving the change through replacement. Document what happens to existing nodes,
and consider disruption budgets and workloads before claiming immediate convergence.

### Replacement and hashing

Read `Hash()` and the `AKSNodeClassHashVersion` guidance in
[aksnodeclass.go](../pkg/apis/v1beta1/aksnodeclass.go), plus
[drift.go](../pkg/cloudprovider/drift.go). Review a hash-version change when an existing
hashed field's default changes, an already-populated field enters the hash, or a field
leaves it. A new nil optional field may preserve the hash under the existing hash options;
prove that with tests rather than bumping the version automatically.

Extend both relevant tables in
[aksnodeclass_hash_test.go](../pkg/apis/v1beta1/aksnodeclass_hash_test.go), including
literal-hash expectations and intended hash changes. Test omission and explicit zero
values. Do not update golden hashes merely to make unexplained failures disappear.

The spec hash alone does not cover every effective behavior change. If resolved status,
images, bootstrap data, or operator configuration changes, identify and test the actual
drift mechanism in each supported provisioning family. A hash-version bump is not itself
a promise that nodes will be immediately replaced.

### In-place changes

Only exclude a setting from replacement hashing with an explicit reason and a complete
reconciliation path. For a mutable Azure property, inspect
[inplaceupdate](../pkg/controllers/nodeclaim/inplaceupdate/):

* Extend `vmInPlaceUpdateFields` and/or `aksMachineInPlaceUpdateFields` in `utils.go`,
  including the code populating them. Their contents differ intentionally.
* Extend `vmPatchers` and/or `aksMachinePatchers` in `patch.go`. Preserve unrelated fields
  and confirm whether omitting a property preserves or clears its existing value.
* Extend the watch predicate in `predicates.go`; a tag-only predicate will not observe
  an unrelated setting change.
* Test successful update, no-op reconciliation, retry, and controller restart.
  An E2E update should retain the NodeClaim/resource identity and avoid unintended drift.
* Consider a one-time increase in reconciliation/API traffic if existing in-place hashes
  change. Ensure patch logs cannot expose sensitive values.

### Upgrade contract

Test an existing manifest that omits the feature, an explicitly configured object, and
the new transition behavior. State any old-controller/new-CRD or rollback constraints
that require ordered deployment. Do not assume an older controller understands a new
field, label, or enum value.

**Exit:** reviewers know whether each change affects new nodes, existing nodes in place,
or replacement, and existing users retain the documented behavior.

## 8. Build the evidence and run checks

### Test plan

Extend the established sibling test or package suite and reuse its fakes, setup,
assertions, and table style. Start acceptance tests from pending Pod pressure where
possible. Avoid a disconnected suite that only exercises your new helper.

| Area | Evidence to add where applicable |
|---|---|
| API/accessors: `pkg/apis/v1beta1/*_test.go` | Omitted parents/children, explicit zero/false, enabled values, defaults, and unsupported combinations. |
| API admission: `crd_validation_cel_test.go` | Generated CRD installation; valid/invalid create and update cases; legacy-object compatibility. |
| Scheduling: `pkg/providers/instancetype/` and existing acceptance suites | Capable/incapable/missing-capability SKUs; labels and requirements; zone/regional differences; no feasible offering; cache-hit isolation. |
| Resolution: `pkg/controllers/nodeclass/status/` | External-input transitions, dependency errors, permissions, restart, readiness, and convergence. |
| Payloads: `pkg/providers/instance/` and `pkg/cloudprovider/` | VM and Machine SDK requests, nil/false semantics, sibling preservation, conversions, and heterogeneous batches. |
| Lifecycle: existing drift/in-place-update/GC suites | Enable/disable transitions, no unintended replacement, partial create/update failure, retry, and already-deleted resources. |
| E2E: `test/suites/<existing suite>/` | Real node behavior and Azure lifecycle, not only request acceptance or controller labels. |

For deterministic SKU coverage, include supporting and non-supporting data plus absent
capabilities. Follow existing test fakes. If changing generated SKU fixtures, use the
`codegen-skugen` target and generator flow in [Makefile](../Makefile); do not hand-edit
`pkg/fake/zz_generated.sku.*.go`. Review regenerated data for unrelated changes.

### Local checks

Use the development environment in [CONTRIBUTING.md](../CONTRIBUTING.md) and the
toolchain/dependencies declared by this checkout. From the repository root:

```bash
# Focus unit/acceptance behavior during development; replace the test-name expression.
FOCUS='YourFeature' make test

# Before submitting code: generation, validation, lint, and unit/acceptance tests.
make presubmit
```

`make verify` performs generation and static checks; it does **not** run the test suite.
`make test` runs tests under `pkg/`; the E2E module under `test/` is separate. A focus
expression that matches no tests is not evidence that the feature works: confirm the
intended cases ran. If required tools or the API test environment are missing, use
`make toolchain` as described in the contribution instructions.

Do not hand-edit deepcopy files, generated CRDs, or chart CRD copies. Run `make verify`,
inspect and include generated changes, and rerun after corrections. For documentation-only
changes, review links/examples and follow any applicable documentation checks; Azure E2E
is not required merely because this guide describes it.

### Real-Azure evidence and external contributors

Use a dedicated disposable test cluster/subscription scope that you are authorized to
operate. Provisioning and E2E tests create, modify, and delete resources and incur costs.
Confirm the cluster context, credentials, permissions, regional quota, and feature
registrations before running them. Never use a production or shared workload cluster.
Do not paste credentials or customer identifiers into test evidence.

After following the environment setup, a focused run can use:

```bash
# Example only: select the existing suite, feature expression, and supported mode.
TEST_SUITE=Storage FOCUS='YourFeature' PROVISION_MODE=aksmachineapi make e2etests
```

This command does not create/configure the test environment for you. Inspect prerequisites
and cleanup behavior in [Makefile](../Makefile), [Makefile-az.mk](../Makefile-az.mk), and
the [workflow instructions](../.github/workflows/README.md). Do not invoke broad cleanup
targets without understanding their scope.

For maintainer-operated CI, the workflow instructions describe branch identity
permissions and manual E2E triggering. Fork contributors generally cannot assume access
to upstream credentials or runners. If you cannot run a required case, include local
results, the exact missing coverage, and a request for maintainer-assisted testing.
Maintainers decide what evidence is sufficient before merging.

Record the commit tested, command/workflow link, deployment model, mode, region,
placement, Kubernetes version, architecture/image, assertions, and result. Keep
**passed**, **failed**, **skipped**, and **not run** distinct. Link separate zonal and
regional runs for zone-sensitive behavior. A skip for an unsupported mode is useful,
but is not a passing behavioral test.

Reuse an existing E2E suite. A new directory under `test/suites/` must also enter
[e2e-matrix.yaml](../.github/workflows/e2e-matrix.yaml). Verify on-node effects through
the relevant service, process, runtime configuration, or workload; a label alone does
not prove enablement. Helm snapshots prove rendering, not runtime behavior.

**Exit:** every supported behavior has appropriate evidence, and limitations are visible.
Ask: **would these tests fail if the feature were wired to the wrong mode or did nothing?**

## 9. Plan rollout and recovery

Distinguish these controls; do not call all of them a "kill switch":

| Control | What it can do | What it does not establish |
|---|---|---|
| Default-off field | Require users to opt in. | An operator-wide override for already-enabled configurations. |
| Automatic-enablement threshold | Limit eligibility for automatic resolution. | Disabling explicit-required or already-sticky-enabled configurations. A compiled constant also requires a new deployment. |
| Operator feature gate | Control the behavior defined by that gate. | Automatic remediation of existing nodes, unless implemented and tested. |
| Configuration change or rollback | Stop future use or deploy an older implementation, subject to compatibility. | Undoing Azure-side state or safely replacing all existing nodes. |

For example, LocalDNS `Preferred` retains an enabled state before rechecking eligibility,
and `Required` does not use the Preferred version gate. Raising its threshold is not a
universal disable mechanism. Treat this as a caution about scope, not a recommended
recovery procedure.

Document who can operate each control in NAP and self-hosted installations; whether it
needs a deployment, spec edit, or platform action; how quickly it takes effect; and what
happens to new, in-flight, and existing nodes. If emergency disablement or reversal is
not possible, state that explicitly and discuss rollout risk before implementation.
Test the claimed control rather than assuming a revert is safe.

Also provide:

* Measured provisioning-latency and per-node Azure API-call impact, with baseline and
  conditions, or a defensible explanation when the path adds neither.
* Convergence latency accounting for cache freshness, reconciliation, service behavior,
  and any replacement/disruption constraints.
* Actionable conditions/events/errors and troubleshooting steps for likely failures,
  without credentials, tokens, or customer identifiers.
* Cleanup ownership and a bounded canary plan appropriate to the feature's risk, including
  success criteria and circumstances that stop expansion.

**Exit:** operators can detect trouble, limit further impact, and understand recovery
without assuming the provider can undo a create-only Azure setting.

## 10. Prepare the pull request

Follow [the repository PR template](../.github/pull-request-template.md), including
its title conventions, documentation-impact selection, and release-note block.
Link the issue and feature design, and complete any CLA checks. Explain:

* The user-visible change, public platform contract, alternatives, and intentional
  deviations from AKS defaults.
* The compatibility/defaults matrix, supported modes/models, and unsupported or unverified
  cases with reasons and tracking issues.
* The validation, scheduling, request, label, and lifecycle paths changed. Call out any
  existing assertion you inverted: it is evidence of a behavior change, not routine churn.
* Commands/results and real workflow or reproducible test evidence for the actual revision.
  Identify missing access without claiming a waiver from testing.
* Upgrade, rollout/recovery, pricing, permissions, and documentation implications.
  Add user examples, supported combinations, and troubleshooting in the relevant public
  documentation, or link a coordinated documentation issue as the PR template permits.

Keep commits and explanations focused. Use public references; do not require reviewers
to access private discussions. Check automated review suggestions against source and
tests before accepting them, and explain evidence-based disagreements.

## Ready-for-review checklist

Use this as a summary in the PR; mark conditional items not applicable with a reason.

* [ ] Problem, scope, surface choice, and unresolved design questions are documented.
* [ ] SDK/service availability, defaults, prerequisites, and public references are recorded.
* [ ] Compatibility matrix distinguishes supported, unsupported, and unverified cases.
* [ ] Omitted settings preserve the agreed behavior for existing manifests and nodes.
* [ ] Validation failures occur at the intended layer with useful diagnostics.
* [ ] Scheduling requirements, offering filters, cache keys, and observed labels agree.
* [ ] Every claimed provisioning path is wired through to the SDK payload and node behavior.
* [ ] External-state readiness, observation, transitions, and permissions are covered.
* [ ] Replacement/in-place behavior, hash changes, upgrades, and rollback limits are explained.
* [ ] Retries, partial failures, cancellation, and cleanup have appropriate coverage.
* [ ] Relevant local checks and E2E results identify the tested revision; gaps are explicit.
* [ ] Rollout controls, operator ownership, costs, and convergence are documented.
* [ ] User docs/examples, release note, PR template, and CLA requirements are addressed.

## Appendix: implementation examples and lessons

These are reference points, not prescriptions for every feature. Source links resolve
within your checkout; historical PRs are upstream context. When citing behavior in a
design, link the specific revision and test that establish it rather than relying on
a mutable file or an old review comment. Update related references when implementation
changes; avoid copying version thresholds, hash literals, or TTLs into new designs.

| Concern | Starting point | Lesson |
|---|---|---|
| Defaults and effective values | Artifact-streaming accessors in [aksnodeclass.go](../pkg/apis/v1beta1/aksnodeclass.go); upstream [1397](https://github.com/Azure/karpenter-provider-azure/pull/1397), [1600](https://github.com/Azure/karpenter-provider-azure/pull/1600), [1611](https://github.com/Azure/karpenter-provider-azure/pull/1611). | Confirm platform defaults first; keep description, accessors, payloads, and tests consistent. |
| Missing Machine API wiring | [Machine helpers](../pkg/providers/instance/aksmachineinstancehelpers.go); upstream [1607](https://github.com/Azure/karpenter-provider-azure/pull/1607). | A bootstrap implementation does not prove Machine API support. |
| Image/bootstrap parity | [Image-family resolver](../pkg/providers/imagefamily/resolver.go); upstream [1039](https://github.com/Azure/karpenter-provider-azure/pull/1039). | OS-specific values must reach the bootstrap request, not stop at an intermediate struct. |
| Labels versus configuration | [Surface guidance](./0015-nodeclass-vs-labels.md), [Ultra SSD design](./0013-ultra-ssd-support.md), and upstream [1704](https://github.com/Azure/karpenter-provider-azure/pull/1704). | Capability support, selected configuration, and scheduling intent are different concepts. |
| Cluster-dependent resolution | [LocalDNS reconciler](../pkg/controllers/nodeclass/status/localdns.go) and its sibling tests. | Single ownership, readiness, observation, and wire translation matter; sticky semantics are feature-specific. |
| Mutable resources | [In-place update controller](../pkg/controllers/nodeclaim/inplaceupdate/controller.go) and [utils](../pkg/controllers/nodeclaim/inplaceupdate/utils.go). | Hash exclusion alone does not implement updates; predicates, patchers, retries, and evidence are required. |

### Category-specific checks

| Category | Additional seams to inspect |
|---|---|
| Networking | NodeClass versus operator subnet precedence; plugin/dataplane-dependent labels and `maxPods`; NIC ownership/GC; subnet, NSG, and route behavior in Azure. Extend existing networking tables rather than duplicating them. |
| Security | SKU capability, image lineage, FIPS/Trusted Launch interlocks, least-privilege permissions, and patch-log redaction. Re-derive interacting validation rules when adding another security input. |
| Storage | Capability versus paid enablement; managed/ephemeral OS disk constraints; zonal/regional availability; attach/use behavior and resource cleanup. |
| Bootstrap / GPU | Every supported image family, architecture, Kubernetes version, units/conversions, driver ownership, and actual service/runtime readiness. |
| Capacity / scheduling | Offering-level constraints, pricing/quota/consolidation effects, error classification, and consistent placement values in both requirements and node labels. |
