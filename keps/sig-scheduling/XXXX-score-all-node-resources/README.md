# KEP-XXXX: Score All Node Resources

<!-- toc -->
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Story 1](#story-1)
    - [Story 2](#story-2)
  - [Risks and Mitigations](#risks-and-mitigations)
- [Design Details](#design-details)
  - [API Changes](#api-changes)
  - [Scoring Logic Changes](#scoring-logic-changes)
  - [Test Plan](#test-plan)
    - [Unit Tests](#unit-tests)
    - [Integration Tests](#integration-tests)
    - [E2E Tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Alternatives](#alternatives)
<!-- /toc -->

## Summary

This KEP proposes adding a `ScoreAllResources` boolean field to
`NodeResourcesFitArgs` in the kube-scheduler. When enabled, all resources
defined in `ScoringStrategy.Resources` participate in node scoring, even if
the pod being scheduled does not request them. This allows cluster operators
to make scheduling decisions based on the overall resource utilization of
nodes rather than only the resources requested by individual pods.

## Motivation

Currently, the `NodeResourcesFit` scoring plugin only considers resources
that the pod explicitly requests. For scalar resources (extended resources,
huge pages, etc.), if the pod does not request them, they are skipped during
scoring. This means that even if a cluster operator configures an extended
resource (e.g., `gpu`, `fpga`) in `ScoringStrategy.Resources`, it will have
no effect on scheduling decisions for pods that do not request that resource.

This behavior can lead to unbalanced resource utilization across nodes. For
example, in a cluster with GPU nodes, non-GPU pods may be evenly spread
across all nodes without considering GPU utilization, potentially leaving
some GPU nodes heavily loaded with non-GPU workloads while others sit idle.

### Goals

- Allow cluster operators to configure the scheduler to consider all
  configured resources during scoring, regardless of whether the pod
  requests them.
- Provide a simple boolean toggle (`ScoreAllResources`) at the
  `NodeResourcesFitArgs` level.
- Maintain full backward compatibility: the default value is `false`,
  preserving existing behavior.

### Non-Goals

- Changing the behavior of the `Filter` (pre-filter/filter) phase.
  `ScoreAllResources` only affects scoring.
- Changing the behavior of the `NodeResourcesBalancedAllocation` plugin.
- Introducing per-resource granularity for this toggle (e.g., score some
  unrequested resources but not others). This could be a future enhancement.

## Proposal

Add a `ScoreAllResources` field to `NodeResourcesFitArgs`:

```go
type NodeResourcesFitArgs struct {
    metav1.TypeMeta

    IgnoredResources      []string
    IgnoredResourceGroups []string

    // ScoreAllResources, when set to true, makes all resources defined
    // in ScoringStrategy.Resources participate in scoring even if the
    // pod doesn't request them. The pod's request for unrequested
    // resources is treated as 0.
    ScoreAllResources bool

    ScoringStrategy *ScoringStrategy
}
```

When `ScoreAllResources` is `true`, the scoring loop in
`resourceAllocationScorer.score()` no longer skips scalar resources that the
pod does not request. For such resources, the pod's request value is 0, and
the score is computed based on the node's existing `allocated` and
`allocatable` values.

### User Stories

#### Story 1

As a cluster administrator managing a heterogeneous cluster with both GPU
and non-GPU nodes, I want the scheduler to account for GPU utilization when
scoring nodes for all pods, so that non-GPU workloads are preferentially
placed on non-GPU nodes, preserving GPU node capacity for GPU workloads.

**Configuration:**

```yaml
apiVersion: kubescheduler.config.k8s.io/v1
kind: KubeSchedulerConfiguration
profiles:
- pluginConfig:
  - name: NodeResourcesFit
    args:
      scoreAllResources: true
      scoringStrategy:
        type: LeastAllocated
        resources:
        - name: cpu
          weight: 1
        - name: memory
          weight: 1
        - name: nvidia.com/gpu
          weight: 3
```

With this configuration, nodes with more available GPUs score higher for all
pods (including non-GPU pods), encouraging the scheduler to leave GPU nodes
less loaded overall.

#### Story 2

As a platform engineer, I have a cluster with FPGA resources on some nodes.
I want the `MostAllocated` strategy to pack workloads onto nodes that are
already heavily utilized across all resource dimensions, including FPGA.
With `ScoreAllResources: true`, nodes with idle FPGAs receive lower
`MostAllocated` scores, discouraging the scheduler from placing more
workloads there.

### Risks and Mitigations

**Risk:** Enabling `ScoreAllResources` may cause unexpected scoring changes
for existing clusters.

**Mitigation:** The feature defaults to `false`, requiring explicit opt-in.
Documentation will clearly explain the behavioral difference.

**Risk:** Performance impact from scoring additional resources.

**Mitigation:** The overhead is negligible. The scoring loop already
iterates over all configured resources; the only change is removing a
`continue` statement for unrequested scalar resources. No additional API
calls or computations are introduced.

## Design Details

### API Changes

**Versioned API** (`staging/src/k8s.io/kube-scheduler/config/v1/types_pluginargs.go`):

```go
type NodeResourcesFitArgs struct {
    metav1.TypeMeta `json:",inline"`

    IgnoredResources      []string         `json:"ignoredResources,omitempty"`
    IgnoredResourceGroups []string         `json:"ignoredResourceGroups,omitempty"`
    ScoreAllResources     bool             `json:"scoreAllResources,omitempty"`
    ScoringStrategy       *ScoringStrategy `json:"scoringStrategy,omitempty"`
}
```

**Internal API** (`pkg/scheduler/apis/config/types_pluginargs.go`):

```go
type NodeResourcesFitArgs struct {
    metav1.TypeMeta

    IgnoredResources      []string
    IgnoredResourceGroups []string
    ScoreAllResources     bool
    ScoringStrategy       *ScoringStrategy
}
```

- Type: `bool` (not pointer). A zero value of `false` preserves existing
  behavior. No need to distinguish between unset and `false`.
- No additional validation required: any boolean value is valid.
- No defaulting logic required: Go zero value (`false`) is the desired
  default.

### Scoring Logic Changes

**File:** `pkg/scheduler/framework/plugins/noderesources/resource_allocation.go`

The `resourceAllocationScorer` struct gains a `scoreAllResources bool` field.

The core change is in the `score()` method:

```go
// Before:
if podRequests[i] == 0 && schedutil.IsScalarResourceName(resource) {
    continue
}

// After:
if !r.scoreAllResources && podRequests[i] == 0 && schedutil.IsScalarResourceName(resource) {
    continue
}
```

Key behaviors preserved:
- If the node does not have the resource at all (`allocatable == 0`), the
  resource is still skipped regardless of `ScoreAllResources`. This avoids
  division by zero and is semantically correct.
- CPU and Memory are not affected by this change because
  `IsScalarResourceName` returns `false` for them.
- The `BalancedAllocation` plugin is not affected because it uses a separate
  `NodeResourcesBalancedAllocationArgs` and its own scorer instance.

**File:** `pkg/scheduler/framework/plugins/noderesources/fit.go`

The `nodeResourceStrategyTypeMap` closures pass `args.ScoreAllResources` to
the `resourceAllocationScorer` for all three strategies (`LeastAllocated`,
`MostAllocated`, `RequestedToCapacityRatio`).

### Test Plan

[x] I/we understand the owners of the involved components may require
updates to existing tests to make this code solid enough prior to
committing the changes necessary to implement this enhancement.

#### Unit Tests

Coverage of the core scoring logic in:

- `pkg/scheduler/framework/plugins/noderesources/least_allocated_test.go`:
  New test case "score all resources even if the pod does not request" that
  verifies extended resources participate in scoring when
  `ScoreAllResources=true`.
- `pkg/scheduler/framework/plugins/noderesources/most_allocated_test.go`:
  Analogous test case for MostAllocated strategy.
- `pkg/scheduler/framework/plugins/noderesources/requested_to_capacity_ratio_test.go`:
  Analogous test case for RequestedToCapacityRatio strategy.
- Existing "bypass extended resource" tests remain unchanged, confirming
  backward compatibility when `ScoreAllResources=false`.

#### Integration Tests

Integration tests will be added to verify end-to-end scheduling behavior:
- A pod without extended resource requests should be scheduled considering
  extended resource utilization on nodes when `ScoreAllResources=true`.
- A pod without extended resource requests should NOT consider extended
  resource utilization when `ScoreAllResources=false`.

#### E2E Tests

E2E tests will be added in a later stage (Beta) to validate the feature in
a real cluster environment.

### Graduation Criteria

#### Alpha

- Feature implemented behind the `ScoreAllResources` configuration field
  (no feature gate required as this is purely configuration-driven).
- Unit tests covering all three scoring strategies.
- Documentation in the scheduler configuration reference.

#### Beta

- Integration tests added.
- Gather feedback from early adopters.
- Address any issues discovered during Alpha.

#### GA

- E2E tests added.
- Feature has been stable for at least 2 releases.
- No significant bugs reported.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- Set `scoreAllResources: true` in the `NodeResourcesFitArgs` plugin
  configuration in the `KubeSchedulerConfiguration` file and restart
  kube-scheduler.
- To disable, set `scoreAllResources: false` (or remove the field) and
  restart kube-scheduler.

###### Does enabling the feature change any default behavior?

No. The default value is `false`, which preserves existing behavior. The
feature must be explicitly enabled.

###### Can the feature be disabled once it has been enabled?

Yes. Set `scoreAllResources: false` and restart kube-scheduler. Already
scheduled pods are not affected. Only future scheduling decisions will
revert to the previous scoring behavior.

###### What happens if we reenable the feature if it was previously rolled back?

Future scheduling decisions will again consider all configured resources
during scoring. No state is persisted; behavior changes take effect
immediately upon scheduler restart.

### Rollout, Upgrade and Rollback Planning

###### How can a rollout or rollback of the feature be done?

Modify the scheduler configuration file and restart kube-scheduler. No
other components are affected.

###### What specific metrics should inform a rollback?

- `scheduler_plugin_execution_duration_seconds{plugin="NodeResourcesFit",extension_point="Score"}`:
  Monitor for unexpected increases in scoring latency.
- `scheduler_scheduling_attempt_duration_seconds`: Monitor for overall
  scheduling latency increases.
- Pod distribution across nodes: Monitor whether pods are being placed as
  expected.

###### Were upgrade and rollback tested?

Will be tested during Alpha/Beta phases.

### Monitoring Requirements

###### How can an operator determine if the feature is in use by workloads?

Check the kube-scheduler configuration for `scoreAllResources: true` in the
`NodeResourcesFit` plugin args. There is no runtime signal beyond the
configuration.

###### How can someone using this feature know that it is working for their instance?

Observe that node scores reported by the scheduler (via verbose logging at
`-v=10`) include contributions from extended resources even for pods that
do not request them.

###### What are the reasonable SLOs for the enhancement?

The existing scheduler SLOs apply. This feature should not measurably
impact scheduling latency because the scoring loop already iterates over
all configured resources.

###### What are the SLIs?

- `scheduler_plugin_execution_duration_seconds{plugin="NodeResourcesFit",extension_point="Score"}`

### Dependencies

###### Does this feature depend on any specific services running in the cluster?

No.

### Scalability

###### Will enabling / using this feature result in any new API calls?

No.

###### Will enabling / using this feature result in introducing new API types?

No. Only a new field is added to an existing configuration type.

###### Will enabling / using this feature result in any new calls to the cloud provider?

No.

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

No.

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

No measurable impact. The change removes a `continue` statement in an
existing loop iteration, adding at most one `calculateResourceAllocatableRequest`
call per additional resource per node. This is a lightweight in-memory
operation.

###### Will enabling / using this feature result in non-negligible increase of resource usage?

No.

### Troubleshooting

###### How does this feature react if the API server and/or etcd is unavailable?

Not applicable. This feature is entirely within the kube-scheduler's
scoring logic and does not interact with the API server during scoring.

###### What are the other known failure modes?

None specific to this feature. If misconfigured (e.g., high weight on an
extended resource that most nodes don't have), scheduling may become
suboptimal but will not fail.

###### What steps should be taken if SLOs are not being met to determine the problem?

1. Check scheduler logs at `-v=10` for resource scores on each node.
2. Compare scores with `ScoreAllResources=true` vs `false`.
3. If latency increases are observed, reduce the number of resources in
   `ScoringStrategy.Resources`.

## Implementation History

- 2026-03-06: Initial KEP draft and implementation.

## Alternatives

### Per-Resource Toggle

Instead of a single boolean, we could add a `ScoreWhenUnrequested bool`
field to each `ResourceSpec`. This provides finer granularity but adds
complexity to the API and configuration. The current proposal keeps things
simple; per-resource control can be added later if there is demand.

### Separate Plugin

A separate scoring plugin (e.g., `NodeResourcesGlobalScore`) could be
created that always scores all resources. However, this would duplicate much
of the existing `NodeResourcesFit` scoring logic and increase maintenance
burden. Adding a field to the existing plugin is more pragmatic.

### Default to True

We considered making `ScoreAllResources` default to `true` to provide
better out-of-box behavior. However, this would change existing scheduling
behavior for all clusters, which violates Kubernetes' backward
compatibility guarantees. Defaulting to `false` is the safer choice.
