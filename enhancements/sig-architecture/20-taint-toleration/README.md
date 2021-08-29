# Taint-Toleration in Placement APIs

## Release Signoff Checklist

- [ ] Enhancement is `implementable`
- [ ] Design details are appropriately documented from clear requirements
- [ ] Test plan is defined
- [ ] Graduation criteria for dev preview, tech preview, GA
- [ ] User-facing documentation is created in [website](https://github.com/open-cluster-management/website/)

## Summary

The proposed work would define Taints and Tolerations in placement APIs to allow users to control the selection of managed clusters more flexibly, and evict selected unhealthy/not-reporting clusters after a certain period.

## Motivation

For the new placement enhancement need of supporting filtering unhealthy/not-reporting clusters and keep workloads from being placed in unhealthy or unreachable clusters, we may consider the similar logic of taint/toleration in Kubernetes. Basically, there are two things we could do:

1) Implement taint/toleration in placement apis.
2) Add a new controller to map states of clusters to taints automatically.

Taints and tolerations would work together to allow users to control the selection of managed clusters more flexibly, and evict selected clusters after a certain period. Specifically, Taints are properties of managed clusters, they allow a placement to repel a set of managed clusters. Tolerations are applied to placements, and allow the managed clusters with matching taints to be scheduled onto placements.

### User Stories

Story 1: Users can prevent some workloads from being scheduled to those managed clusters.

Story 2: Users schedule some workloads to certain managed clusters.

Story 3: When a managed cluster gets offline, the system can make applications deployed on this cluster to be transferred to another available managed cluster immediately. 

Story 4: Users are able to evict workloads deployed on a cluster without modifying or being aware of existing placements’ details.

Story 5: When a managed cluster gets offline, users can make applications deployed on this cluster to be transferred to another available managed cluster after a tolerated time.

## Proposal

### API repo

#### ManagedClusterSpec
In `ManagedClusterSpec`, we can add `taints` which has the similar structure with taints in Kubernetes. But to keep things simple at the beginning, we can remove `effect` which is in Kubernetes' taint while only keep `key` and `value` .

```go
type ManagedClusterSpec struct {
   ManagedClusterClientConfigs []ClientConfig `json:"managedClusterClientConfigs,omitempty"`
   HubAcceptsClient bool `json:"hubAcceptsClient"`
   LeaseDurationSeconds int32 `json:"leaseDurationSeconds,omitempty"`
   // Taints is a property of managed cluster that allow the cluster to be repelled when scheduling.
   // +optional
   Taints []Taint `json:"taints,omitempty"`
}
 
type Taint struct {
   // Required. The taint key to be applied to a cluster.
   Key string `json:"key"`
   // The taint value corresponding to the taint key.
   // +optional
   Value string `json:"value,omitempty"`
   // TimeAdded represents the time at which the taint was added.
   // +optional
   TimeAdded *metav1.Time `json:"timeadded,omitempty"`
}
```

#### PlacementSpec
Similarly, we add "Kubernetes like" `tolerations` to the `PlacementSpec`, which has `key`, `value`, `operator`, and `tolerationSeconds`, while skip `effect` now.

```go
type PlacementSpec struct {
   ClusterSets []string `json:"clusterSets,omitempty"`
   NumberOfClusters *int32 `json:"numberOfClusters,omitempty"`
   Predicates []ClusterPredicate `json:"predicates,omitempty"`
   // Tolerations are applied to placements, and allow (but do not require) the managed clusters with
   // certain taints to schedule onto placements with matching tolerations.
   // +optional
   Tolerations []Toleration `json:"tolerations,omitempty"`
}
 
type Toleration struct {
   // Key is the taint key that the toleration applies to. Empty means match all taint keys.
   // If the key is empty, operator must be Exists; this combination means to match all values and all keys.
   // +optional
   Key string `json:"key,omitempty"`
   // Operator represents a key's relationship to the value.
   // Valid operators are Exists and Equal. Defaults to Equal.
   // Exists is equivalent to wildcard for value, so that a placement can
   // tolerate all taints of a particular category.
   // +optional
   Operator TolerationOperator `json:"operator,omitempty"`
   // Value is the taint value the toleration matches to.
   // If the operator is Exists, the value should be empty, otherwise just a regular string.
   // +optional
   Value string `json:"value,omitempty"`
   // TolerationSeconds represents the period of time the toleration (which must only be effective under the Evict Senario, otherwise this field should be ignored) tolerates the taint. By default,
   // it is not set, which means tolerate the taint forever (do not evict). 
   // The start time of counting the TolerationSeconds should be the TimeAdded in Taint, not the cluster scheduled time or TolerationSeconds added time.
   // +optional
   TolerationSeconds *int64 `json:"tolerationSeconds,omitempty"`
}
```

### Placement repo

#### Four steps of current schedule:

* Filter: Get clusters satisfying all the requirements from all the clusters. 

* Score: Sort by certain criteria.

* Select: Only select the number of clusters we need, not all feasible clusters.

* Bind: merge the cluster decisions into placementdecisions

#### Changes:

* Current placement process is implemented by plugins. To add taint-toleration, we need to add a new plugin which check taint-toleration matching after the filter pipline in the schedule function. Specifically, in this new plugin, for every taint of all the taints on a managed cluster, we go through every toleration on the placement to see if there is a match. Once the match is found, the checking passes.
* By the end of the schedule process, we check all tolerated taints to get the first expiration time of the cluster. `scheduleResult` should also be updated by adding `requeue` and `delay`, which represent whether requeue need to be triggered in the controller and after how many seconds it would be triggerd. Then we add a requeue process in the controller to evict the cluster after this `delay` time. `delay` time is the first expiration time of tolerated taints, and it would be computed as: toleration.TolerationSeconds - (current time - taint.TimeAdded), and the first expiration time is the minimum among all expiration times.

### Register repo

#### Add new controller:

Based on what we have in current Placement APIs, we add a new controller which can map the state of unhealthy and unreachable managed clusters to corresponding taints, so we can leverage the Taint-Toleration to support[ filtering unhealthy/not-reporting clusters (issue #48)](https://github.com/open-cluster-management-io/community/issues/48).

### Examples

#### 1. No Schedule on Certain Clusters

Users want to prevent some workloads from being scheduled to certain managed clusters. 

cluster.yaml:

```yaml
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: cluster1
  labels:
    cluster.open-cluster-management.io/clusterset: clusterset1
spec:
  hubAcceptsClient: true
  taints:
    - key: gpu
      value: "true"
```

placement.yaml:

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: placement1
  namespace: default
spec:~
```

#### 2. Schedule on Certain Clusters

Users want to schedule some workloads to certain managed clusters.

cluster.yaml:

```yaml
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: cluster1
  labels:
    cluster.open-cluster-management.io/clusterset: clusterset1
spec:
  hubAcceptsClient: true
  taints:
    - key: gpu
      value: "true"
```
placement.yaml:

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: placement1
  namespace: default
spec:
  tolerations:
    - key: gpu
      value: "true"
      operator: Equal
```

#### 3. Support Filtering Unhealthy/Unreachable Clusters

When managed clusters' availability becomes `false`, the taint `unhealthy` would be automatically added to clusters. If managed clusters' availability becomes `unknown`, the taint `unreachable` would be added. Those unhealthy or unreachable clusters could be evicted immediatly or after `TolerationSeconds`.

##### 3.1 Immediate Eviction

Users want applications deployed on this cluster to be transferred to another available managed cluster immediately.

cluster.yaml:

```yaml
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: cluster1
  labels:
    cluster.open-cluster-management.io/clusterset: clusterset1
spec:
  hubAcceptsClient: true
  taints:
    - key: gpu
      value: "true"
    - key: unhealthy
```

placement.yaml:

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: placement1
  namespace: default
spec:
  tolerations:
    - key: gpu
      value: "true"
      operator: Equal
```

##### 3.2 Eviction After TolerationSeconds

When a managed cluster gets offline, users want applications deployed on this cluster to be transferred to another available managed cluster after a tolerated time.

cluster.yaml:

```yaml
apiVersion: cluster.open-cluster-management.io/v1
kind: ManagedCluster
metadata:
  name: cluster1
  labels:
    cluster.open-cluster-management.io/clusterset: clusterset1
spec:
  hubAcceptsClient: true
  taints:
    - key: gpu
      value: "true"
    - key: unreachable
      timeadded: 2021-07-06T15:00:00+08:00
```

placement.yaml:

```yaml
apiVersion: cluster.open-cluster-management.io/v1alpha1
kind: Placement
metadata:
  name: placement1
  namespace: default
spec:
  tolerations:
    - key: gpu
      value: "true"
      operator: Equal
    - key: unreachable
      operator: Exists
      tolerationSeconds: 90
```

### Test Plan

- Unit tests will cover the functionality of the controller.
- Unit tests will cover the new plugin.
- Integration tests will cover all user stories.
- e2e tests will cover the following cases:
  - Choose clusters by creating a taint with matched toleration;
  - Prevent choosing clusters by creating a taint without matched toleration;
  - `unhealthy`/`unreachable` taints are added automaticaly when availabiliies of clusters become `false`/`unknown`;
  - Clusters with `unhealthy`/`unreachable` taints are evicted immediately when tolerationSeconds <= 0 or was not set;
  - Clusters with `unhealthy`/`unreachable` taints are evicted after tolerationSeconds when tolerationSeconds > 0;

### Graduation Criteria

#### Alpha
1. The new plugin is reviewed and accepted;
2. Implementation is completed to support the functionalities of taint-toleration;
3. Develop test cases to demonstrate that the above user stories work correctly;

#### Beta
1. Support mapping unhealthy/unreachable clusters to taints automatically by adding controllers in Register repo;

#### GA
1. Pass the performance/scalability testing;

### Upgrade Strategy

### Version Skew Strategy

## Alternatives

### Use affinity by adding timeAdded to claims, instead of taint-toleration to ensure placement can repel inappropriate clusters.

1. Pros

- Less code, less implementation.

2. Cons

- As for user story 1, if users want to make certain clusters unselectable by using affinity, they also need to know affinity settings on placements. While by using taint-toleration, users can be awareless to tolerations. Similarly, user story 4 is also not supported.
- Only claims can be modified to add TolerationSeconds, while labels can’t.