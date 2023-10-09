---
title: cvo-log-level-api
authors:
  - "@Davoska"
reviewers:
  - "@wking"
  - "@LalatenduMohanty"
  - "@petr-muller"
approvers:
  - "@wking"
api-approvers:
  - "@deads2k"
  - "@bparees"
creation-date: 2023-10-09
last-updated: 2024-09-10
tracking-link:
  - https://issues.redhat.com/browse/OTA-923
see-also:
  - "None"
replaces:
  - "None"
superseded-by:
  - "None"
---

# CVO Log Level API

## Summary

This enhancement describes the API changes needed to provide a simple
way of dynamically changing the verbosity level of Cluster Version Operator's logs.

## Motivation

The [Cluster Version Operator (CVO)](https://github.com/openshift/cluster-version-operator)
is responsible for a number of resources and thus logs a lot of information.
However, there is currently no way to easily change the verbosity of the CVO logs.

It would be useful to provide functionality for the cluster administrators and
OpenShift engineers to easily modify the log level to a desired level similarly
as can be done for [`ClusterOperators`](https://github.com/openshift/api/blob/5852b58f4b1071fe85d9c49dff2667a9b2a74841/operator/v1/types.go#L58-L65).

### User Stories

* As an OpenShift administrator, I want to increase the log level of the CVO to
  more easily troubleshoot any potential issues regarding the cluster or CVO.
* As an OpenShift administrator, I want to decrease the log level of the CVO to
  save up more storage space.
* As an OpenShift engineer, I want to increase the log level of the CVO for
  the CI runs so that I can more easily troubleshoot any potential issues that occurred.
* As an OpenShift engineer, I want to decrease the log level of the CVO for
  shipped releases so that customers don't receive debug logs.

### Goals

The goal is to add a user-facing API for controlling the verbosity of the CVO logs.

### Non-Goals

Change the default logging verbosity of the Cluster Version Operator in production
OCP clusters.

## Proposal

This enhancement proposes to add a new field to the `ClusterVersion` Custom Resource Definition (CRD). The CVO
will dynamically change the verbosity of its logs based on the value provided in the
new field. The field is described in more detail in the **Implementation Details**
section. A CRD modification is proposed because `ClusterOperator` CRD contains
such a field as well. This will ensure consistency across different OpenShift operators.

### Workflow Description

Given a cluster administrator and a working cluster for which the administrator is responsible.

**Cluster administrator** is a human user responsible for managing the cluster.

1. The cluster administrator notices an issue in the cluster and chooses to
   troubleshoot the issue.
2. The cluster administrator, after some troubleshooting, notices that the logs
   of the Cluster Version Operator (CVO) might help.
3. The cluster administrator notices that the logs are not detailed enough to
   troubleshoot the issue.
4. The cluster administrator raises the log level from the default value to a
   more verbose level by simply modifying the ClusterVersion object via the web
   console or by patching the resource by using the CLI.
5. The cluster administrator fixes the issue in the cluster.
6. The cluster administrator notices that the CVO outputs too many logs for the
   administrator's liking.
7. The cluster administrator lowers the log level of the CVO to the default level.
8. The cluster administrator is now a happy cluster administrator.

### API Extensions

The enhancement proposes to modify the `ClusterVersion` CRD.

### Topology Considerations

#### Hypershift / Hosted Control Planes

The implementation of HyperShift needs to account for this change because updating
the `github.com/openshift/api` module in HyperShit would provide a way for the
administrator of a hosted cluster to set the log level of a CVO in a
hosted control plane. The hosted cluster administrator could set an undesirable level.

However, this design proposes a desired feature that HyperShift can potentially
utilize, and thus HyperShift implementation should address the above-mentioned issue
for the overall benefit.

In HyperShift, this enhancement proposes to reconcile a fixed value of the new property
as [HyperShift currently does for some other properties of the ClusterVersion object](https://github.com/openshift/hypershift/blob/90aa44d064f6fe476ba4a3f25973768cbdf05eb5/control-plane-operator/hostedclusterconfigoperator/controllers/resources/resources.go#L973-L979)
of a hosted cluster. And potentially, in the future, the HyperShift may even
dynamically utilize this property.

#### Standalone Clusters

In standalone clusters, cluster administrators will be able to specify the desired
CVO log level. No additional changes are needed.

#### Single-node Deployments or MicroShift

SNO and MicroShift Cluster administrators will now have the choice to modify the
CVO logs. Same as standalone clusters. The choice to reduce the CVO logs may be
even more desirable for such cluster administrators to reduce the storage
consumed by the CVO if needed. No additional changes are needed.

### Implementation Details/Notes/Constraints

This enhancement proposes to add a new field to configure the log level of the CVO
by modifying the `ClusterVersion` resource.
Because more additional CVO specific options may be exposed by the
`ClusterVersion` in the future, this enhancement proposes to add a new
`OperatorOptions` field to the existing [`ClusterVersionSpec`](https://github.com/openshift/api/blob/3e5de946111c67cb1f2b09ff7da569012d15931d/config/v1/types_cluster_version.go#L50) struct:

```go
type ClusterVersionSpec struct {
...
	// operatorOptions contains options to configure the CVO.
	//
	// +optional 
	OperatorOptions *ClusterVersionOperatorOptions `json:"operatorOptions,omitempty"`
}
```

The new `OperatorOptions` field will group all the CVO specific fields
to make the `ClusterVersionSpec` more organized, future-proof, and clear.
The `ClusterVersionOperatorOptions` type is defined as:

```go
// ClusterVersionOperatorOptions contains options to configure the CVO.
//
// +k8s:deepcopy-gen=true
type ClusterVersionOperatorOptions struct {
	// operatorLogLevel controls the verbosity level of logs produced by the CVO.
	//
	// Valid values are: "Normal", "Debug", "Trace", "TraceAll".
	// Defaults to "Normal".
	// +optional
	// +kubebuilder:default=Normal
	LogLevel CVOLogLevel `json:"logLevel,omitempty"`
}
```

The `CVOLogLevel` type is defined as:

```go
// CVOLogLevel represents the verbosity level of logs produced by the CVO.
//
// +kubebuilder:validation:Enum="";Normal;Debug;Trace;TraceAll
type CVOLogLevel string

var (
	// CVOLogLevelNormal is the default.  Normal, working log information, everything is fine, but helpful notices for auditing or common operations.  In kube, this is probably glog=2.
	CVOLogLevelNormal CVOLogLevel = "Normal"

	// CVOLogLevelDebug is used when something went wrong.  Even common operations may be logged, and less helpful but more quantity of notices.  In kube, this is probably glog=4.
	CVOLogLevelDebug CVOLogLevel = "Debug"

	// CVOLogLevelTrace is used when something went really badly and even more verbose logs are needed.  Logging every function call as part of a common operation, to tracing execution of a query.  In kube, this is probably glog=6.
	CVOLogLevelTrace CVOLogLevel = "Trace"

	// CVOLogLevelTraceAll is used when something is broken at the level of API content/decoding.  It will dump complete body content.  If you turn this on in a production cluster
	// prepare from serious performance issues and massive amounts of logs.  In kube, this is probably glog=8.
	CVOLogLevelTraceAll CVOLogLevel = "TraceAll"
)
```

Ideally, this enhancement would propose to use the existing [`LogLevel`](https://github.com/openshift/api/blob/36ce464529eb357673342c06be5886c5463cfc50/operator/v1/types.go#L91-L107)
type that is used by `ClusterOperators` defined in the [`github.com/openshift/api/operator/v1`](https://github.com/openshift/api/tree/master/operator/v1) package.

However, importing the [`github.com/openshift/api/operator/v1`](https://github.com/openshift/api/tree/master/operator/v1) package from inside
the [`github.com/openshift/api/config/v1`](https://github.com/openshift/api/tree/master/config/v1) package where the `ClusterVersionSpec`
struct resides causes **import cycles**. To ensure consistency across the operators
and to not cause import cycles, this type will be redefined in the [api/config/v1/types_cluster_version.go](https://github.com/openshift/api/blob/master/config/v1/types_cluster_version.go) file.

### Risks and Mitigations

No risks are known.

### Drawbacks

No drawbacks are known.

## Open Questions [optional]

The `ClusterVersion` type is currently defined as a configuration for the CVO.

```go
// ClusterVersion is the configuration for the ClusterVersionOperator. This is where
// parameters related to automatic updates can be set.
...
type ClusterVersion struct {
```

However, `ClusterVersionSpec` is defined as a desired version state of a cluster.

```go
// ClusterVersionSpec is the desired version state of the cluster. It includes
// the version the cluster should be at, how the cluster is identified, and
// where the cluster should look for version updates.
...
type ClusterVersionSpec struct 
```

Thus, including CVO specific options may not be potentially desired. We could
update the `ClusterVersionSpec` definition.

## Test Plan

Unit tests will be written to test if setting the new field sets the respective logging in the CVO.

## Graduation Criteria

### Dev Preview -> Tech Preview

The enhancement will be released directly to GA.

### Tech Preview -> GA

The enhancement will be released directly to GA. The new `LogLevel` field and its data type are equal
to that of the [`OperatorLogLevel`](https://github.com/openshift/api/blob/1f9525271dda5b7a3db735ca1713ad7dc1a4a0ac/operator/v1/types.go#L74)
field in the [`OperatorSpec`](https://github.com/openshift/api/blob/1f9525271dda5b7a3db735ca1713ad7dc1a4a0ac/operator/v1/types.go#L54)
structure in the [`github.com/openshift/api/operator/v1`](https://github.com/openshift/api/tree/1f9525271dda5b7a3db735ca1713ad7dc1a4a0ac/operator/v1)
package. The field in the [`github.com/openshift/api/operator/v1`](https://github.com/openshift/api/tree/1f9525271dda5b7a3db735ca1713ad7dc1a4a0ac/operator/v1)
package has been stable for several years. Thus, the addition may be directly
released to GA. The implementation of the CVO to dynamically change its verbosity
level is considered to be less complex.

User facing documentation will be created in [openshift-docs](https://github.com/openshift/openshift-docs/).

### Removing a deprecated feature

No existing feature will be deprecated or removed.

## Upgrade / Downgrade Strategy

Not applicable.

## Version Skew Strategy

The relevant components for this enhancement are the CVO and the ClusterVersion
CR. These components move closely together on updates and downgrades. During a
version skew, the default log level will be used.

A newer CVO that consumes an older ClusterVersion will receive an empty
`OperatorLogLevel` field, and the CVO will continue using the default log level.

An older CVO that consumes newer ClusterVersion will not notice the
`OperatorLogLevel` field, and the CVO will continue using the default log level.

## Operational Aspects of API Extensions

This enhancement proposes a minor addition to the existing `ClusterVersion` CRD.
This new addition will operationally impact only the CVO. It may increase or
decrease its logs. Impacting the storage.

## Support Procedures

Not applicable.

## Alternatives

### Move the existing `LogLevel` type to a different package

The proposed implementation does not follow the DRY (Don't repeat yourself) 
principle, as it copies over the existing `LogLevel` data type from the
[`github.com/openshift/api/operator/v1`](https://github.com/openshift/api/tree/master/operator/v1) package. This was proposed as the
`ClusterVersionSpec` struct may not import the existing type, as this causes 
import cycles.

An alternative solution is to move the existing `LogLevel` type definition to a
package that will not cause import cycles. It may be the
[`github.com/openshift/api/config/v1`](https://github.com/openshift/api/tree/master/config/v1)
package, different package, or a new package.
This will not duplicate the code and may be a desired alternative. However, this
would move a stable definition of a type corresponding to its package where it's
heavily being used to another less logically corresponding package only due to
an importing issue. Due to this issue, this alternative was not chosen.

### The CVO state will be represented by a new Custom Resource

The CVO will use a new Custom Resource (CR) for its state. Cluster operators 
tend to have two associated CRs. `ClusterOperator` CR to configure a respective
cluster operator state, and a CR that represents the operator's operand.

The CVO is not a `ClusterOperator`; however, the CVO will try to adopt
conceptually the structure of CRs used by `ClusterOperators`.

However, right now there is no need to provide CVO specific conditions, status,
or plethora of options. The health of the CVO affects the health of the cluster. All
this information is propagated to the `ClusterVersion` CR. Using a new CR could
result in a duplication of status information. A new CR would only contain the new
log level specification as of the moment, and using a new CR only due to this 
reason may not be desired.

This alternative was not chosen due to its increased complexity.

### Do not introduce an option to dynamically modify the verbosity level of logs

The option for cluster administrators to choose the desired verbosity level will
not be introduced. This alternative was not chosen as the potential benefit of
the proposed change greatly outweighs the implementation cost.

## Infrastructure Needed [optional]

No new infrastructure is needed.