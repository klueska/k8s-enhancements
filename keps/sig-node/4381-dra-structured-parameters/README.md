<!--
**Note:** When your KEP is complete, all of these comment blocks should be removed.

To get started with this template:

- [ ] **Pick a hosting SIG.**
  Make sure that the problem space is something the SIG is interested in taking
  up. KEPs should not be checked in without a sponsoring SIG.
- [ ] **Create an issue in kubernetes/enhancements**
  When filing an enhancement tracking issue, please make sure to complete all
  fields in that template. One of the fields asks for a link to the KEP. You
  can leave that blank until this KEP is filed, and then go back to the
  enhancement and add the link.
- [ ] **Make a copy of this template directory.**
  Copy this template into the owning SIG's directory and name it
  `NNNN-short-descriptive-title`, where `NNNN` is the issue number (with no
  leading-zero padding) assigned to your enhancement above.
- [ ] **Fill out as much of the kep.yaml file as you can.**
  At minimum, you should fill in the "Title", "Authors", "Owning-sig",
  "Status", and date-related fields.
- [ ] **Fill out this file as best you can.**
  At minimum, you should fill in the "Summary" and "Motivation" sections.
  These should be easy if you've preflighted the idea of the KEP with the
  appropriate SIG(s).
- [ ] **Create a PR for this KEP.**
  Assign it to people in the SIG who are sponsoring this process.
- [ ] **Merge early and iterate.**
  Avoid getting hung up on specific details and instead aim to get the goals of
  the KEP clarified and merged quickly. The best way to do this is to just
  start with the high-level sections and fill out details incrementally in
  subsequent PRs.

Just because a KEP is merged does not mean it is complete or approved. Any KEP
marked as `provisional` is a working document and subject to change. You can
denote sections that are under active debate as follows:

```
<<[UNRESOLVED optional short context or usernames ]>>
Stuff that is being argued.
<<[/UNRESOLVED]>>
```

When editing KEPS, aim for tightly-scoped, single-topic PRs to keep discussions
focused. If you disagree with what is already in a document, open a new PR
with suggested changes.

One KEP corresponds to one "feature" or "enhancement" for its whole lifecycle.
You do not need a new KEP to move from beta to GA, for example. If
new details emerge that belong in the KEP, edit the KEP. Once a feature has become
"implemented", major changes should get new KEPs.

The canonical place for the latest set of instructions (and the likely source
of this file) is [here](/keps/NNNN-kep-template/README.md).

**Note:** Any PRs to move a KEP to `implementable`, or significant changes once
it is marked `implementable`, must be approved by each of the KEP approvers.
If none of those approvers are still appropriate, then changes to that list
should be approved by the remaining approvers and/or the owning SIG (or
SIG Architecture for cross-cutting KEPs).
-->

# [KEP-4381](https://github.com/kubernetes/enhancements/issues/4381): Dynamic Resource Allocation with Structured Parameters

<!-- toc -->
- [Release Signoff Checklist](#release-signoff-checklist)
- [Summary](#summary)
- [Motivation](#motivation)
  - [Goals](#goals)
  - [Non-Goals](#non-goals)
- [Proposal](#proposal)
  - [User Stories](#user-stories)
    - [Cluster add-on development](#cluster-add-on-development)
    - [Cluster configuration](#cluster-configuration)
    - [Partial GPU allocation](#partial-gpu-allocation)
  - [Publishing node resources](#publishing-node-resources)
  - [Using structured parameters](#using-structured-parameters)
  - [Communicating allocation to the DRA driver](#communicating-allocation-to-the-dra-driver)
  - [Risks and Mitigations](#risks-and-mitigations)
    - [Feature not used](#feature-not-used)
    - [Compromised node](#compromised-node)
    - [Compromised kubelet plugin](#compromised-kubelet-plugin)
    - [User permissions and quotas](#user-permissions-and-quotas)
    - [Usability](#usability)
- [Design Details](#design-details)
  - [Components](#components)
  - [State and communication](#state-and-communication)
  - [Sharing a single ResourceClaim](#sharing-a-single-resourceclaim)
  - [Ephemeral vs. persistent ResourceClaims lifecycle](#ephemeral-vs-persistent-resourceclaims-lifecycle)
  - [Scheduled pods with unallocated or unreserved claims](#scheduled-pods-with-unallocated-or-unreserved-claims)
  - [Handling non graceful node shutdowns](#handling-non-graceful-node-shutdowns)
  - [API](#api)
    - [resource.k8s.io](#resourcek8sio)
      - [ResourceSlice](#resourceslice)
      - [DeviceClass](#deviceclass)
      - [Allocation result](#allocation-result)
      - [ResourceClaimTemplate](#resourceclaimtemplate)
      - [ResourceQuota](#resourcequota)
      - [Object references](#object-references)
    - [core](#core)
  - [kube-controller-manager](#kube-controller-manager)
  - [kube-scheduler](#kube-scheduler)
    - [EventsToRegister](#eventstoregister)
    - [PreEnqueue](#preenqueue)
    - [Pre-filter](#pre-filter)
    - [Filter](#filter)
    - [Post-filter](#post-filter)
    - [Reserve](#reserve)
    - [PreBind](#prebind)
    - [Unreserve](#unreserve)
  - [kubelet](#kubelet)
    - [Communication between kubelet and kubelet plugin](#communication-between-kubelet-and-kubelet-plugin)
    - [Version skew](#version-skew)
    - [Security](#security)
    - [Managing resources](#managing-resources)
      - [NodePrepareResource](#nodeprepareresource)
      - [NodeUnprepareResources](#nodeunprepareresources)
  - [Simulation with CA](#simulation-with-ca)
  - [Test Plan](#test-plan)
      - [Prerequisite testing updates](#prerequisite-testing-updates)
      - [Unit tests](#unit-tests)
      - [Integration tests](#integration-tests)
      - [e2e tests](#e2e-tests)
  - [Graduation Criteria](#graduation-criteria)
    - [Alpha](#alpha)
    - [Beta](#beta)
    - [GA](#ga)
  - [Upgrade / Downgrade Strategy](#upgrade--downgrade-strategy)
  - [Version Skew Strategy](#version-skew-strategy)
- [Production Readiness Review Questionnaire](#production-readiness-review-questionnaire)
  - [Feature Enablement and Rollback](#feature-enablement-and-rollback)
  - [Rollout, Upgrade and Rollback Planning](#rollout-upgrade-and-rollback-planning)
  - [Monitoring Requirements](#monitoring-requirements)
  - [Dependencies](#dependencies)
  - [Scalability](#scalability)
  - [Troubleshooting](#troubleshooting)
- [Implementation History](#implementation-history)
- [Drawbacks](#drawbacks)
- [Alternatives](#alternatives)
  - [Publishing resource information in node status](#publishing-resource-information-in-node-status)
  - [Injecting vendor logic into CA](#injecting-vendor-logic-into-ca)
  - [ResourceClaimTemplate](#resourceclaimtemplate-1)
  - [Reusing volume support as-is](#reusing-volume-support-as-is)
  - [Extend volume support](#extend-volume-support)
  - [Extend Device Plugins](#extend-device-plugins)
  - [ResourceDriver](#resourcedriver)
- [Infrastructure Needed](#infrastructure-needed)
<!-- /toc -->

## Release Signoff Checklist

<!--
**ACTION REQUIRED:** In order to merge code into a release, there must be an
issue in [kubernetes/enhancements] referencing this KEP and targeting a release
milestone **before the [Enhancement Freeze](https://git.k8s.io/sig-release/releases)
of the targeted release**.

For enhancements that make changes to code or processes/procedures in core
Kubernetes—i.e., [kubernetes/kubernetes], we require the following Release
Signoff checklist to be completed.

Check these off as they are completed for the Release Team to track. These
checklist items _must_ be updated for the enhancement to be released.
-->

Items marked with (R) are required *prior to targeting to a milestone / release*.

- [ ] (R) Enhancement issue in release milestone, which links to KEP dir in [kubernetes/enhancements] (not the initial KEP PR)
- [ ] (R) KEP approvers have approved the KEP status as `implementable`
- [ ] (R) Design details are appropriately documented
- [ ] (R) Test plan is in place, giving consideration to SIG Architecture and SIG Testing input (including test refactors)
  - [ ] e2e Tests for all Beta API Operations (endpoints)
  - [ ] (R) Ensure GA e2e tests meet requirements for [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
  - [ ] (R) Minimum Two Week Window for GA e2e tests to prove flake free
- [ ] (R) Graduation criteria is in place
  - [ ] (R) [all GA Endpoints](https://github.com/kubernetes/community/pull/1806) must be hit by [Conformance Tests](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/conformance-tests.md) 
- [ ] (R) Production readiness review completed
- [ ] (R) Production readiness review approved
- [ ] "Implementation History" section is up-to-date for milestone
- [ ] User-facing documentation has been created in [kubernetes/website], for publication to [kubernetes.io]
- [ ] Supporting documentation—e.g., additional design documents, links to mailing list discussions/SIG meetings, relevant PRs/issues, release notes

<!--
**Note:** This checklist is iterative and should be reviewed and updated every time this enhancement is being considered for a milestone.
-->

[kubernetes.io]: https://kubernetes.io/
[kubernetes/enhancements]: https://git.k8s.io/enhancements
[kubernetes/kubernetes]: https://git.k8s.io/kubernetes
[kubernetes/website]: https://git.k8s.io/website

## Summary

This KEP originally defined an extension of the ["classic" DRA #3063
KEP](../3063-dynamic-resource-allocation/README.md). Now the roles are
reversed: this KEP defines the base functionality and #3063 adds an optional
extension.

Users are increasingly deploying Kubernetes as management solution for new
workloads (batch processing) and in new environments (edge computing). Such
workloads no longer need just RAM and CPU, but also access to specialized
hardware. With upcoming enhancements of data center interconnects, accelerators
can be installed outside of specific nodes and be connected to nodes
dynamically as needed.

This KEP introduces a new API for describing which of these new resources
a pod needs. Typically, such resources are devices like a GPU or other kinds
of accelerators. The API supports:

- Network-attached devices. The existing [device plugin API](https://github.com/kubernetes/design-proposals-archive/blob/main/resource-management/device-plugin.md)
  is limited to hardware on a node. However, further work is still
  needed to actually use the new API with those.
- Sharing of allocated devices between multiple containers or pods.
  The device plugin API currently cannot share devices at all. It
  could be extended to share devices between containers in a single pod,
  but supporting sharing between pods would need a completely new
  API similar to the one in this KEP.
- Using a device that is expensive to initialize multiple times
  in different pods. This is not possible at the moment.
- Custom parameters that describe device configuration.
  With the current Pod API, annotations have to be used to capture such
  parameters and then hacks are needed to access them from a CSI driver or
  device plugin.

Support for new hardware will be provided by hardware vendor add-ons. Those add-ons
are called "DRA drivers". They are responsible for reporting available devices in a format defined and
understood by Kubernetes and for configuring hardware before it is used. Kubernetes
handles the allocation of those devices as part of pod scheduling.
This KEP does not replace other means of requesting traditional resources
(RAM/CPU, volumes, extended resources).

In the typical case of node-local devices, the high-level form of DRA with
structured parameters is as follows:

* DRA drivers publish their available devices in the form of a
  `ResourceSlice` object on a node-by-node basis. This object is stored in the
  API server and available to the scheduler (or Cluster Autoscaler) to query
  when a request for devices comes in later on.

* When a user wants to consume a resource, they create a `ResourceClaim`.
  This object defines how
  many devices are needed and which capabilities they must have.

* With such a claim in place, the scheduler (or Cluster Autoscaler) can evaluate against the
  `ResourceSlice` of any candidate nodes by comparing attributes, without knowing exactly what is
  being requested. They then use this information to help decide which node to
  schedule a pod on (as well as allocate resources from its `ResourceSlice`
  in the process).

* Once a node is chosen and the allocation decisions made, the scheduler will
  store the result in the API server as well as update its in-memory model of
  available resources. DRA drivers are responsible for using this allocation
  result to inject any allocated resource into the Pod, according to
  the device choices made by the scheduler. This includes applying any
  configuration information attached to the original request.

This KEP defines a way to describe devices with a name and some associated
attributes that are used to select devices.

## Motivation

<!--
This section is for explicitly listing the motivation, goals, and non-goals of
this KEP.  Describe why the change is important and the benefits to users. The
motivation section can optionally provide links to [experience reports] to
demonstrate the interest in a KEP within the wider Kubernetes community.

[experience reports]: https://github.com/golang/go/wiki/ExperienceReports
-->

Originally, Kubernetes and its scheduler only tracked CPU and RAM as
resources for containers. Later, support for storage and discrete,
countable per-node extended resources was added. The kubelet device plugin
interface then made such local resources available to containers. But
for many newer devices, this approach and the Kubernetes API for
requesting these custom resources is too limited. This KEP may eventually
address limitations of the current approach for the following use cases:

- *Device initialization*: When starting a workload that uses
  an accelerator like an FPGA, I’d like to have the accelerator
  reconfigured or reprogrammed without having to deploy my application
  with full hardware access and/or root privileges. Running applications
  with less privileges is better for overall security of the cluster.

  *Limitation*: Currently, it’s impossible to specify the desired
  device properties that are required for reconfiguring devices.
  For the FPGA example, a file containing the desired configuration
  of the FPGA has to be referenced.

- *Device cleanup*: When my workload is finished, I would like to have
  a mechanism for cleanup of the device, that will ensure that device
  does not contain traces/parameters/data from previous workloads and
  appropriate power state/shutdown. For example, an FPGA might have
  to be reset because its configuration for the workload was
  confidential.

  *Limitation*: Post-stop actions are not supported.

- *Partial allocation*: When workloads use only a portion of the device
  capabilities, devices can be partitioned (e.g. using Nvidia MIG or SR-IOV) to
  better match workload needs. Sharing the devices in this way can greatly
  increase HW utilization / reduce costs.

- *Limitation*: currently there's no API to request partial device
  allocation. With the current device plugin API, devices need to be
  pre-partitioned and advertised in the same way a full / static devices
  are. User must then select a pre-partitioned device instead of having one
  created for them on the fly based on their particular resource
  constraints. Without the ability to create devices dynamically (i.e. at the
  time they are requested) the set of pre-defined devices must be carefully
  tuned to ensure that device resources do not go unused because some of the
  pre-partioned devices are in low-demand. It also puts the burden on the user
  to pick a particular device type, rather than declaring the resource
  constraints more abstractly.

- *Optional allocation*: When deploying a workload I’d like to specify
  soft(optional) device requirements. If a device exists and it’s
  allocatable it will be allocated. If not - the workload will be run on
  a node without a device. GPU and crypto-offload engines are
  examples of this kind of device. If they’re not available, workloads
  can still run by falling back to using only the CPU for the same
  task.

  *Limitation*: Optional allocation is supported neither by the device
  plugins nor by current Pod resource declaration.

- *Support Over the Fabric devices*: When deploying a container, I’d
  like to utilize devices available over the Fabric (network, special
  links, etc).

  *Limitation*: The device plugin API is designed for node-local resources that
  get discovered by a plugin running on the node. Projects like
  [Akri](https://www.cncf.io/projects/akri/) have to work around that by
  reporting the same network-attached resource on all nodes that it could
  get attached to and then updating resource availability on all of those
  nodes when resources get used.

Several other limitations are addressed by
[CDI](https://github.com/container-orchestrated-devices/container-device-interface/),
a container runtime extension that this KEP is using to expose resources
inside a container.

### Goals

- Enable cluster autoscaling when pods use resource claims, with correct
  decisions and changing the cluster size by more than one node at a time.

- Support node-local resources

- Support configuration parameters that are specified in a format defined by a
  vendor.

### Non-Goals

* Replace the device plugin API. For devices that fit into its model
  of a single, linear quantity it is a good solution. Other devices
  should use dynamic resource allocation. Both are expected to co-exist, with vendors
  choosing the API that better suits their needs on a case-by-case
  basis. Because the new API is going to be implemented independently of the
  existing device plugin support, there's little risk of breaking stable APIs.

* Provide an abstraction layer for device requests, i.e., something like a
  “I want some kind of GPU”. Users will need to know about specific
  DRA drivers and which configuration parameters and attributes they support.
  Administrators and/or vendors can simplify this by creating device class
  objects in the cluster, but defining those is out-of-scope for Kubernetes.

  Portability of workloads could be added on top of this proposal by
  standardizing attributes and what they mean for certain classes of devices.
  The
  [Resource Class
  Proposal](https://docs.google.com/document/d/1qKiIVs9AMh2Ua5thhtvWqOqW0MSle_RV3lfriO1Aj6U/edit#heading=h.jzfmfdca34kj)
  included such an approach.

* Support network-attached resources

## Proposal

### User Stories

#### Cluster add-on development

As a hardware vendor, I want to make my hardware available also to applications
that run in a container under Kubernetes. I want to make it easy for a cluster
administrator to configure a cluster where some nodes have this hardware.

I develop a DRA driver, package it in a container image and provide YAML files
for running it as a kubelet plugin via a daemon set.

Documentation for administrators explains how the nodes need to be set
up. Documentation for users explains which parameters control the behavior of
my hardware and how to use it inside a container.

#### Cluster configuration

As a cluster administrator, I want to make GPUs from vendor ACME available to users
of that cluster. I prepare the nodes and deploy the vendor's components with
`kubectl create`.

I create a DeviceClass for the hardware with parameters that only I as the
administrator am allowed to choose, like for example running a command with
root privileges that does some cluster-specific initialization of a device
each time it is prepared on a node:

```yaml
apiVersion: resource.k8s.io/v1alpha3
kind: DeviceClass
metadata:
  name: acme-gpu

selectors:
- expression: device.driverName == "gpu.acme.example.com"

config:
- opaque:
    driverName: gpu.acme.example.com
    parameters:
      apiVersion: gpu.acme.example.com/v1
      kind: GPUInit
      # DANGER! This option must not be accepted for
      # user-supplied parameters.
      initCommand:
        - /usr/local/bin/acme-gpu-init
        - --cluster
        - my-cluster
```

#### Partial GPU allocation

As a user, I want to use a GPU as accelerator, but don't need exclusive access
to that GPU. Running my workload with just 2Gb of memory is sufficient. This is
supported by the ACME GPU hardware. I know that the administrator has created
an "acme-gpu" DeviceClass and that such devices have a
`gpu.acme.example.com/memory` attribute.

For a simple trial, I create a Pod directly where two containers share the same subset
of the GPU:

```yaml
apiVersion: resource.k8s.io/v1alpha2
kind: ResourceClaimTemplate
metadata:
  name: device-consumer-gpu-template
spec:
  metadata:
    # Additional annotations or labels for the
    # ResourceClaim could be specified here.
  spec:
    requests:
    - name: gpu-request # could be used to select this device in a container when requesting more than one
      deviceClassName: acme-gpu
      selectors:
      - expression: device.attributes["gpu.acme.example.com/memory"].isGreaterThan(quantity("2Gi")) # Always set for ACME GPUs.
---
apiVersion: v1
kind: Pod
metadata:
  name: device-consumer
spec:
  resourceClaims:
  - name: "gpu" # this name gets referenced below under "claims"
    resourceClaimTemplateName: device-consumer-gpu-template
  containers:
  - name: workload
    image: my-app
    command: ["/bin/program"]
    resources:
      requests:
        memory: "64Mi"
        cpu: "250m"
      limits:
        memory: "128Mi"
        cpu: "500m"
      claims:
        - name: "gpu"
  - name: monitor
    image: my-app
    command: ["/bin/other-program"]
    resources:
      requests:
        memory: "32Mi"
        cpu: "25m"
      limits:
        memory: "64Mi"
        cpu: "50m"
      claims:
      - name: "gpu"
```

This request triggers allocation on a node that has a GPU device or fraction of a GPU device
with at least 2Gi of memory and then the Pod runs on that node.
The lifecycle of the allocation is tied to the lifecycle of the Pod.

In production, a similar PodTemplateSpec in a Deployment will be used.

### Publishing node resources

The devices available on a node need to be published to the API server. In
the typical case, this is expected to be published by the on-node driver
as described in the next paragraph. However, the source of this data may vary; for
example, a cloud provider controller could populate this based upon information
from the cloud provider API.

In the node-local case, each driver running on a node publishes a set of
`ResourceSlice` objects to the API server for its own resources, using its
connection to the apiserver. The collection of these objects form a pool from
which resources can be allocated. Some additional fields (defined in the API
section) enable a consumer to determine whether it has a complete and
consistent view of that pool.

Access control through a validating admission
policy can ensure that the drivers running on one node are not allowed to
create or modify `ResourceSlices` belonging to another node. The `nodeName`
and `driverName` fields in each `ResourceSlice` object are used to determine which objects are
managed by which driver instance. The owner reference ensures that objects
beloging to a node get cleaned up when the node gets removed.

In addition, whenever the kubelet starts, it first deletes all `ResourceSlices`
belonging to the node with a `DeleteCollection` call that uses the node name in
a field filter. This ensures that no pods depending in DRA get scheduled to the
node until the required DRA drivers have started up again (node reboot) and
reconnected to kubelet (kubelet restart). It also ensures that drivers which
don't get started up again at all don't leave stale `ResourceSlices`
behind. Garbage collection does not help in this case because the node object
still exists. For the same reasons, the ResourceSlices belonging to a driver
get removed when the driver unregisters, this time with a field filter for node
name and driver name.

Deleting `ResourceSlices` is possible because all information in them can be
reconstructed by the driver. This has no effect on already allocated claims
because the allocation result is tracked in those claims, not the
`ResourceSlice` objects (see [below](#state-and-communication)).

#### Devices as a named list of attributes

Embedded inside each `ResourceSlice` is a list of one or more devices, represented as a named list of attributes.

```yaml
kind: ResourceSlice
apiVersion: resource.k8s.io/v1alpha3
...
spec:
  # The node name indicates the node.
  # Each driver on a node provides pools of devices for allocation,
  # with unique device names inside each pool.
  # Usually, but not necessarily, that pool name is the same as the
  # node name.
  nodeName: worker-1
  poolName: worker-1
  driverName: cards.dra.example.com
  devices:
  - name: card-1 # unique inside the worker-1 pool
    attributes:
    - name: manufacturer # a vendor-specific attribute, automatically qualified as cards.dra.example.com/manufacturer
      string: Cards-are-us Inc.
    - name: productName:
      string: ACME T1000 16GB
    - name: driverVersion
      version: 1.2.3
    - name: runtimeVersion
      string: 11.1.42
    - name: memory
      quantity: 16Gi
    - name: powerSavingSupported
      bool: true
    - name: dra.k8s.io/pciRoot # a fictional standardized attribute, not actually part of this KEP
      string: pci-root-0
```

Compared to labels, attributes have values of exactly one type. As
described later on, these attributes can be used in CEL expressions to select a
specific resource for allocation on a node.

To avoid any future conflicts, we reserve any attributes with the ".k8s.io/" domain prefix
for future use and standardization by Kubernetes. This could be used to describe
topology across resources from different vendors, for example, but this is out-
of-scope for now.

**Note:** If a driver needs to remove a device or change its attributes,
then there is a risk that a claim gets allocated based on the old
`ResourceSlice`. The scheduler must handle
scenarios where more devices are allocated than available. The kubelet plugin
of a DRA driver ***must*** double-check that the allocated devices are still
available when NodePrepareResource is called. If not, the pod cannot start until
the device comes back. Checking it at admission time and treating this as a fatal error
would allow us to delete the pod and trying again with a new one, but is not done
at the moment because admission checks cannot be retried if a check finds
a transient problem.

#### Partitionable devices

In addition to devices, a `ResourceSlice` can also embed a list of
`SharedCapacity` objects. Each `SharedCapacity` object represents some amount
of shared "capacity" that can be consumed by one or more devices listed in the
slice. When listing such devices, the sum of the capacity consumed across all
devices *may* exceed the total amount available in the `SharedCapacity` object.
This allows one, for example, to provide a way of logically partition a device
into a set of overlapping sub-devices, each of which "could" be allocated by a
scheduler (just not at the same time).

As such, scheduler support will need to be added to track the set of
`SharedCapacity` objects provided by each `ResourceSlice` as well as perform
the following steps to decide if a given device is a candidate for allocation
or not:

1. Look at the set of `SharedCapacityConsumed` objects referenced by the device
1. For each `SharedCapacityConsumed` object, look and see how much capacity is still available in the `SharedCapacity` object it is tracking with the same name
1. If enough capacity is still available in the `SharedCapacity` object to satisfy the device's consumption, continue to consider it for allocation
1. If not enough capacity is available, move on to the next device
1. Upon deciding to allocate a device, subtract all of the capacity in its `SharedCapacityConsumed` objects from the corresponding `SharedCapacity` objects being tracked
1. Upon freeing a device, add all of the capacity in its `SharedCapacityConsumed` objects back into the corresponding `SharedCapacity` objects being tracked

**Note:** Only devices embedded in the _same_ `ResourceSlice` where a given
`SharedCapacity` is declared have access to that `SharedCapacity`. This
restriction simplifies the logic required to track these capacities in the
scheduler, and shouldn't be too limiting in practice. Also, note that while it
is described as "subtracting" and "adding" capacity in the `SharedCapacity`
object, these activities are not written back to the `ResourceSlice` itself.
Just like all summaries of current allocations, they are maintained in-memory
by the scheduler based on the totality of device allocations recorded in all
claim statuses.

As an example, consider the following YAML which declares shared capacity for a
set of "memory" blocks within a GPU. Multiple devices (including the full GPU)
are defined which consume some number of these memory blocks at varying
physical locations within GPU memory. With this in place, the scheduler is free
to choose any device that matches a provided device selector, so long as no
other devices have already been allocated that consume overlapping memory
blocks.

```yaml
kind: ResourceSlice
apiVersion: resource.k8s.io/v1alpha3
...
spec:
  # The node name indicates the node.
  # Each driver on a node provides pools of devices for allocation,
  # with unique device names inside each pool.
  # Usually, but not necessarily, that pool name is the same as the
  # node name.
  nodeName: worker-1
  poolName: worker-1
  driverName: gpu.dra.example.com
  sharedCapacity:
  - name: gpu-0-memory-block-0
    capacity: 1
  - name: gpu-0-memory-block-1
    capacity: 1
  - name: gpu-0-memory-block-2
    capacity: 1
  - name: gpu-0-memory-block-3
    capacity: 1
  devices:
  - name: gpu-0
    attributes:
    - name: memory
      quantity: 40Gi
    sharedCapacityConsumed:
    - name: gpu-0-memory-block-0
      capacity: 1
    - name: gpu-0-memory-block-1
      capacity: 1
    - name: gpu-0-memory-block-2
      capacity: 1
    - name: gpu-0-memory-block-3
      capacity: 1
  - name: gpu-0-first-half
    attributes:
    - name: memory
      quantity: 20Gi
    sharedCapacityConsumed:
    - name: gpu-0-memory-block-0
      capacity: 1
    - name: gpu-0-memory-block-1
      capacity: 1
  - name: gpu-0-middle-half
    attributes:
    - name: memory
      quantity: 20Gi
    sharedCapacityConsumed:
    - name: gpu-0-memory-block-1
      capacity: 1
    - name: gpu-0-memory-block-2
      capacity: 1
  - name: gpu-0-second-half
    attributes:
    - name: memory
      quantity: 20Gi
    sharedCapacityConsumed:
    - name: gpu-0-memory-block-2
      capacity: 1
    - name: gpu-0-memory-block-3
      capacity: 1
  - name: gpu-0-first-quarter
    attributes:
    - name: memory
      quantity: 10Gi
    sharedCapacityConsumed:
    - name: gpu-0-memory-block-0
      capacity: 1
  - name: gpu-0-second-quarter
    attributes:
    - name: memory
      quantity: 10Gi
    sharedCapacityConsumed:
    - name: gpu-0-memory-block-1
      capacity: 1
  - name: gpu-0-third-quarter
    attributes:
    - name: memory
      quantity: 10Gi
    sharedCapacityConsumed:
    - name: gpu-0-memory-block-2
      capacity: 1
  - name: gpu-0-fourth-quarter
    attributes:
    - name: memory
      quantity: 10Gi
    sharedCapacityConsumed:
    - name: gpu-0-memory-block-3
      capacity: 1
```

In this example, `gpu-0-first-half` and `gpu-0-second-half` could be allocated
simultaneouly (because the set of `gpu-0-memory-block`s they consume are
mutually exclusive). However, `gpu-0-first-half` and `gpu-0-first-quarter`
could not (because `gpu-0-memory-block-0` is consumed completely by both of
them).

### Using structured parameters

A ResourceClaim is a request to allocate one or more devices. Each request in a
claim may reference a pre-defined DeviceClass to narrow down which devices are
desired or describe that directly in the request itself through one or more device
selectors. A device selector is a CEL expression that must evaluate to true if a
device satisfies the request. A special `device` variable provides access to
the attributes of a device.

To correlate different devices, a claim may have "match attributes". Those are
the names of attributes whose values must be the same for all devices that get
allocated for the claim.

Configuration can be embedded in the claim for all devices at the claim level
and separately for each request. These configuration parameters are ignored by
the scheduler when selecting devices. They get passed down to the DRA drivers
which provide the devices when preparing the devices on a node.

A DeviceClass can contain the same device selector and device configuration
parameters. Those get added to what is specified in the claim when a class gets
referenced.

### Communicating allocation to the DRA driver

The scheduler decides which devices to use for a claim. It also needs to pass
through the opaque vendor parameters, if there are any. This accurately
captures the configuration parameters as they were set at the time of
allocation. All of this information gets stored in the allocation result inside
the ResourceClaim status.

### Risks and Mitigations

<!--
What are the risks of this proposal, and how do we mitigate? Think broadly.
For example, consider both security and how this will impact the larger
Kubernetes ecosystem.

How will security be reviewed, and by whom?

How will UX be reviewed, and by whom?

Consider including folks who also work outside the SIG or subproject.
-->

#### Feature not used

In a cluster where the feature is not used (no DRA driver installed, no
pods using dynamic resource allocation) the impact is minimal, both for
performance and security. The scheduler plugin will
return quickly without doing any work for pods.

#### Compromised node

Kubelet is intentionally limited to read-only access for ResourceClaim
to prevent that a
compromised kubelet interferes with scheduling of pending pods, for example
by updating status information normally set by the scheduler.
Faking such information could be used for a denial-of-service
attack against pods using those ResourceClaims, for example by overwriting
their allocation result with a node selector that matches no node. A
denial-of-service attack against the cluster and other pods is harder, but
still possible. For example, frequently updating ResourceSlice objects could
cause new scheduling attempts for pending pods.

Another potential attack goal is to get pods with sensitive workloads to run on
a compromised node. For pods that don't use special resources nothing changes
in that regard. Such an attack is possible for pods with extended resources
because kubelet is in control of which capacity it reports for those: it could
publish much higher values than the device plugin reported and thus attract
pods to the node that normally would run elsewhere. With dynamic resource
allocation, such an attack is still possible, but the attack code would have to
be different for each DRA driver because all of them will use structured
parameters differently for reporting resource availability.

#### Compromised kubelet plugin

This is the result of an attack against the DRA driver, either from a
container which uses a device exposed by the driver, a compromised kubelet
which interacts with the plugin, or due to DRA driver running on a node
with a compromised root account.

The DRA driver needs write access for ResourceSlices. It can be deployed so
that it can only write objects associated with the node, so the impact of a
compromise would be limited to the node. Other drivers on the node could also
be impacted because there is no separation by driver.

However, a DRA driver may need root access on the node to manage
hardware. Attacking the driver therefore may lead to root privilege
escalation. Ideally, driver authors should try to avoid depending on root
permissions and instead use capabilities or special permissions for the kernel
APIs that they depend on. Long term, limiting apiserver access by driver
name would be useful.

A DRA driver may also need privileged access to remote services to manage
network-attached devices. DRA driver vendors and cluster administrators
have to consider what the effect of a compromise could be for that and how such
privileges could get revoked.

#### User permissions and quotas

Similar to generic ephemeral inline volumes, the [ephemeral resource use
case](#ephemeral-vs-persistent-resourceclaims-lifecycle) gets covered by
creating ResourceClaims on behalf of the user automatically through
kube-controller-manager. The implication is that RBAC rules that are meant to
prevent creating ResourceClaims for certain users can be circumvented, at least
for ephemeral claims. Administrators need to be aware of this caveat when
designing user restrictions.

A quota system that is based on the attributes of devices
could be implemented in Kubernetes. When a user has exhausted their
quota, the scheduler then refuses to allocate further ResourceClaims.

#### Usability

Aside from security implications, usability and usefulness of dynamic resource
allocation also may turn out to be insufficient. Some risks are:

- Slower pod scheduling due to more complex decision making.

- Additional complexity when describing pod requirements because
  separate objects must be created.

All of these risks will have to be evaluated by gathering feedback from users
and DRA driver developers.

## Design Details

### Components

![components](./components.png)

Several components must be implemented or modified in Kubernetes:
- The new API must be added to kube-apiserver.
- A new controller in kube-controller-manager which creates
  ResourceClaims from ResourceClaimTemplates, similar to
  https://github.com/kubernetes/kubernetes/tree/master/pkg/controller/volume/ephemeral.
  It also removes the reservation entry for a consumer in `claim.status.reservedFor`,
  the field that tracks who is allowed to use a claim, when that user no longer exists.
  It clears the allocation and thus makes the underlying resources available again
  when a ResourceClaim is no longer reserved.
- A kube-scheduler plugin must detect Pods which reference a
  ResourceClaim (directly or through a template) and ensure that the
  resource is allocated before the Pod gets scheduled, similar to
  https://github.com/kubernetes/kubernetes/blob/master/pkg/controller/volume/scheduling/scheduler_binder.go
- Kubelet must be extended to manage ResourceClaims
  and to call a kubelet plugin. That plugin returns CDI device ID(s)
  which then must be passed to the container runtime.

A DRA driver can have the following components:
- *admission webhook* (optional): a central component which checks the opaque
  configuration parameters in ResourceClaims, ResourceClaimTemplates and DeviceClasses at
  the time that they are created. Without this, invalid parameters can only
  be detected when a Pod is about to run on a node.
- *kubelet plugin* (required): a component which publishes device information
  and cooperates with kubelet to prepare the usage of the devices on a node.

A [utility library](https://github.com/kubernetes/kubernetes/tree/master/staging/src/k8s.io/dynamic-resource-allocation) for DRA drivers was developed.
It does not have to be used by drivers, therefore it is not described further
in this KEP.

### State and communication

A ResourceClaim object defines what devices are needed and what parameters should be used to configure them once allocated. It is owned by a user and namespaced. Additional
parameters are provided by a cluster admin in DeviceClass objects.

The ResourceClaim spec is immutable. The ResourceClaim
status is reserved for system usage and holds the current state of the
resource. The status must not get lost, which in the past was not ruled
out. For example, status could have been stored in a separate etcd instance
with lower reliability. To recover after a loss, status was meant to be recoverable.
A [recent KEP](https://github.com/kubernetes/enhancements/tree/master/keps/sig-architecture/2527-clarify-status-observations-vs-rbac)
clarified that status will always be stored reliably and can be used as
proposed in this KEP.

Handling state and communication through objects has two advantages:
- Changes for a resource are (almost) atomic, which avoids race conditions.
  One small exception is that changing finalizers and the status have to
  be done in separate operations.
- The only requirement for deployments is that the components can connect to
  the API server.

The entire state of a claim can be determined by looking at its
status (see [API below](#api) for details), for example:

- It is **allocated** if and only if `claim.status.allocation` is non-nil and
  points to the `AllocationResult`, i.e. the struct where information about
  a successful allocation is stored.

- It is in use if and only if `claim.status.reservedFor` contains one or
  more consumers. It does not matter whether those users, usually pods, are
  currently running because that could change at any time.

- A resource is no longer needed when `claim.deletionTimestamp` is set. It must not
  be deallocated yet when it is still in use.

Some of the race conditions that need to be handled are:

- A ResourceClaim gets created and deleted again while the scheduler
  is allocating it. Before it actually starts doing anything, the
  scheduler adds a finalizer. Either adding the finalizer or removing the
  ResourceClaim win. If the scheduler wins, it continues with allocation
  and can either complete or abort the operation when it notices the non-nil
  DeletionTimestamp. Otherwise, allocation gets aborted immediately.

  What this avoids is the situation where an allocation succeed without having
  an object where the result can be stored. The driver can also be killed at
  any time: when it restarts, the finalizer indicates that allocation may be in
  progress and has to be completed or aborted.

  However, users may still force-delete a ResourceClaim, or the entire
  cluster might get deleted. Driver implementations must store enough
  information elsewhere to detect when some allocated resource is no
  longer needed to recover from such scenarios.

- A ResourceClaim gets deleted and recreated while the scheduler is
  adding the finalizer. The scheduler can update the object to add the finalizer
  and then will get a conflict error, which informs the scheduler that it must
  work on a new instance of the claim. In general, patching a ResourceClaim
  is only acceptable when it does not lead to race conditions. To detect
  delete+recreate, the UID must be added as precondition for a patch.
  To detect also potentially conflicting other changes, ResourceVersion
  needs to be checked, too.

- In a cluster with multiple scheduler instances, two pods might get
  scheduled concurrently by different schedulers. When they reference
  the same ResourceClaim which may only get used by one pod at a time,
  only one pod can be scheduled.

  Both schedulers try to add their pod to the `claim.status.reservedFor` field, but only the
  update that reaches the API server first gets stored. The other one fails
  with a conflict error and the scheduler which issued it knows that it must
  put the pod back into the queue, waiting for the ResourceClaim to become
  usable again.

- Two pods get created which both reference the same unallocated claim with
  delayed allocation. A single scheduler can detect this special situation
  and then do allocation only for one of the two pods. When the pods
  are handled by different schedulers, only one will succeed with writing
  back the `claim.status.allocation`.

- Scheduling a pod and allocating resources for it has been attempted, but one
  claim needs to be reallocated to fit the overall resource requirements. A second
  pod gets created which references the same claim that is in the process of
  being deallocated. Because that is visible in the claim status, scheduling
  of the second pod cannot proceed.

### Sharing a single ResourceClaim

Pods reference resource claims in a new `pod.spec.resourceClaims` list. Each
resource in that list can then be made available to one or more containers in
that Pod. Containers can also get access to a specific requested devices in cases where a
claim has asked for more than one.

Consumers of a ResourceClaim are listed in `claim.status.reservedFor`. They
don't need to be Pods. At the moment, Kubernetes itself only handles Pods and
allocation for Pods.

The only limit on the number of concurrent consumers is the maximum size of
that field. Support for additional constraints (maximum number of consumers,
maximum number of nodes) could be added once there are use cases for those.

### Ephemeral vs. persistent ResourceClaims lifecycle

A persistent ResourceClaim has a lifecyle that is independent of any particular
pod. It gets created and deleted by the user. This is useful for resources
which are expensive to configure and that can be used multiple times by pods,
either at the same time or one after the other. Such persistent ResourceClaims
get referenced in the pod spec by name. When a PodTemplateSpec in an app
controller spec references a ResourceClaim by name, all pods created by that
controller also use that name and thus share the resources allocated for that
ResourceClaim.

But often, each Pod is meant to have exclusive access to its own ResourceClaim
instance instead. To support such ephemeral resources without having to modify
all controllers that create Pods, an entry in the new PodSpec.ResourceClaims
list can also be a reference to a ResourceClaimTemplate. When a Pod gets created, such a
template will be used to create a normal ResourceClaim with the Pod as owner
with an
[OwnerReference](https://pkg.go.dev/k8s.io/apimachinery/pkg/apis/meta/v1#OwnerReference)),
and then the normal allocation of the resource takes place. Once the pod got
deleted, the Kubernetes garbage collector will also delete the
ResourceClaim.

This mechanism documents ownership and serves as a fallback for scenarios where
dynamic resource allocation gets disabled in a cluster (for example, during a
downgrade). But it alone is not sufficient: for example, the job controller
does not delete pods immediately when they have completed, which would keep
their resources allocated. Therefore the resource controller watches for pods
that have completed and releases their resource allocations.

The difference between persistent and ephemeral resources for kube-scheduler
and kubelet is that the name of the ResourceClaim needs to be determined
differently: the name of an ephemeral ResourceClaim is recorded in the Pod status.
Ownership must be checked to detect accidental conflicts with
persistent ResourceClaims or previous incarnations of the same ephemeral
resource.

### Scheduled pods with unallocated or unreserved claims

There are several scenarios where a Pod might be scheduled (= `pod.spec.nodeName`
set) while the claims that it depends on are not allocated or not reserved for
it:

* A user might manually create a pod with `pod.spec.nodeName` already set.
* Some special cluster might use its own scheduler and schedule pods without
  using kube-scheduler.
* The feature might have been disabled in kube-scheduler while scheduling
  a pod with claims.

The kubelet is refusing to run such pods and reports the situation through
an event (see below). It's an error scenario that should better be avoided.

Users should avoid this situation by not scheduling pods manually. If they need
it for some reason, they can use a node selector which matches only the desired
node and then let kube-scheduler do the normal scheduling.

Custom schedulers should emulate the behavior of kube-scheduler and ensure that
claims are allocated and reserved before setting `pod.spec.nodeName`.

The last scenario might occur during a downgrade or because of an
administrator's mistake. Administrators can fix this by deleting such pods.

### Handling non graceful node shutdowns

When a node is shut down unexpectedly and is tainted with an `out-of-service`
taint with NoExecute effect as explained in the [Non graceful node shutdown KEP](https://github.com/kubernetes/enhancements/tree/master/keps/sig-storage/2268-non-graceful-shutdown),
all running pods on the node will be deleted by the GC controller and the
resources used by the pods will be deallocated. However, they will not be
un-prepared as the node is down and Kubelet is not running on it.

DRA drivers should be able to handle this situation correctly and
should not expect `UnprepareNodeResources` to be always called.
If resources are unprepared when `Deallocate` is called, `Deallocate`
might need to perform additional actions to correctly deallocate
resources.

### API

The PodSpec gets extended. To minimize the changes in core/v1, all new types
get defined in a new resource group. This makes it possible to revise those
more experimental parts of the API in the future. The new fields in the
PodSpec are gated by the DynamicResourceAllocation feature gate and can only be
set when it is enabled. Initially, they are declared as alpha. Even though they
are alpha, changes to their schema are discouraged and would have to be done by
using new field names.

ResourceClaim, DeviceClass and ResourceClaimTemplate are new built-in types
in `resource.k8s.io/v1alpha3`. This alpha group must be explicitly enabled in
the apiserver's runtime configuration. Using builtin types was chosen instead
of using CRDs because core Kubernetes components must interact with the new
objects and installation of CRDs as part of cluster creation is an unsolved
problem.

Secrets are not part of this API: if a DRA driver needs secrets, for
example to access its own backplane, then it can define custom parameters for
those secrets and retrieve them directly from the apiserver. This works because
drivers are expected to be written for Kubernetes.

#### resource.k8s.io

##### ResourceSlice

For each node, one or more ResourceSlice objects get created. The drivers
on a node publish them with the node as the owner, so they get deleted when a node goes
down and then gets removed.

All list types are atomic because that makes tracking the owner for
server-side-apply (SSA) simpler. Patching individual list elements is not
needed and there is a single owner.

```go
// One or more slices represent a pool of devices managed by a given driver.
// How many slices the driver uses to publish that pool is driver-specific.
// Each device in a given pool must have a unique name.
//
// The slice in which a device gets published may change over time. The unique identifier
// for a device is the tuple `<driver name>/<pool name>/<device name>`. Driver name
// and device name don't contain slashes, so it is okay to concatenate them
// like this in a string with a slash as separator. The pool name itself may contain
// additional slashes.
//
// Whenever a driver needs to update a pool, it bumps the pool generation number
// and updates all slices with that new number and any new device definitions. A consumer
// must only use device definitions from slices with the highest generation number
// and ignore all others.
//
// If necessary, a consumer can check the number of total devices in a pool (included
// in each slice) to determine whether its view of a pool is complete.
//
// For devices that are not local to a node, the node name is not set. Instead,
// a pool name is chosen by the driver as necessary. It can be left blank if the
// device names are unique in the entire cluster. The unique identifier then
// has the same format as before with `<driver name>[/<pool name>]/<device name>`.
type ResourceSlice struct {
    metav1.TypeMeta
    // Standard object metadata
    // +optional
    metav1.ObjectMeta

    Spec ResourceSliceSpec

    // A status might get added later.
}

type ResourceSliceSpec struct {
    // DriverName identifies the DRA driver providing the capacity information.
    // A field selector can be used to list only ResourceSlice
    // objects with a certain driver name.
    //
    // Must be a DNS subdomain and should end with a DNS domain owned by the
    // vendor of the driver.
    DriverName string

    // PoolName is used to identify devices. For node-local devices, this
    // is often the node name, but this is not required.
    //
    // It must not be longer than 253 and must consist of one or more DNS sub-domains
    // separated by slashes.
    PoolName string

    // NodeName identifies the node which provides the devices.
    // A field selector can be used to list only ResourceSlice
    // objects belonging to a certain node.
    //
    // This field can be used to limit access from nodes to slices with
    // the same node name. It also indicates to autoscalers that adding
    // new nodes of the same type as some old node might also make new
    // devices available.
    //
    // NodeName and NodeSelector are mutually exclusive.
    // If both are unset, then the devices are available in the
    // entire cluster.
    //
    // +optional
    NodeName string

    // Defines which nodes have access to the devices in the pool.
    // Must not be set when the node name is set.
    //
    // NodeName and NodeSelector are mutually exclusive.
    // If both are unset, then the devices are available in the
    // entire cluster.
    //
    // +optional
    NodeSelector *v1.NodeSelector

    // The generation gets bumped in all slices of a pool whenever device
    // definitions change. A consumer must only use device definitions from slices
    // with the highest generation number and ignore all others.
    PoolGeneration int64

    // The total number of devices in the pool.
    // Consumers can use this to check whether they have
    // seen all slices.
    PoolDeviceCount int64

    // SharedCapacity defines the set of shared capacity consumable by
    // devices in this ResourceSlice.
    //
    // Must not have more than 128 entries.
    //
    // +listType=atomic
    // +optional
    SharedCapacity []SharedCapacity

    // Devices lists all available devices in this pool.
    //
    // Must not have more than 128 entries.
    Devices []Device

    // FUTURE EXTENSION: some other kind of list, should we ever need it.
    // Old clients seeing an empty Devices field can safely ignore the (to
    // them) empty pool.
}

const ResourceSliceMaxSharedCapacity = 128
const ResourceSliceMaxDevices = 128
const PoolNameMaxLength = validation.DNS1123SubdomainMaxLength // Same as for a single node name.
```

The ResourceSlice object holds the information about available devices.
Together, the slices form a pool of devices that can be allocated.

A status is not strictly needed because
the information in the allocated claim statuses is sufficient to determine
which devices are allocated to which claims.

However, despite the finalizer on the claims it could happen that a well
intentioned but poorly informed user deletes a claim while it is in use.
Therefore adding a status is a useful future extension. That status will
include information about reserved devices (set by schedulers before
allocating a claim) and in-use resources (set by the kubelet). This then
enables conflict resolution when multiple schedulers schedule pods to the same
node because they would be required to set a reservation before proceeding with
the allocation. It also enables detecting inconsistencies and taking actions to
fix those, like deleting pods which use a deleted claim.

```go
// Device represents one individual hardware instance that can be selected based
// on its attributes.
type Device struct {
    // Name is unique identifier among all devices managed by
    // the driver in the pool. It must be a DNS label.
    Name string

    // Attributes defines the set of attributes for this device.
    // The name of each attribute must be unique.
    //
    // Must not have more than 32 entries.
    //
    // +listType=atomic
    // +optional
    Attributes []DeviceAttribute

    // SharedCapacityConsumed defines the set of shared capacity consumed by
    // this device.
    //
    // Must not have more than 32 entries.
    //
    // +listType=atomic
    // +optional
    SharedCapacityConsumed []SharedCapacity
}

const ResourceSliceMaxAttributesPerDevice = 32
const ResourceSliceMaxSharedCapacityConsumedPerDevice = 32

// ResourceSliceMaxDevices and ResourceSliceMaxAttributesPerDevice were chosen
// so that with a maximum `DeviceAttribute` length of 96 characters and a
// maximum `SharedCapacity` length of ~40 characters (ignoring overhead), the
// total size of the ResourceSlice object is around 590KB.

// DeviceAttribute is a combination of an attribute name and its value.
// Exactly one value must be set.
type DeviceAttribute struct {
    // Name is a unique identifier for this attribute, which will be
    // referenced when selecting devices.
    //
    // Attributes are defined either by the owner of the specific driver
    // (usually the vendor) or by some 3rd party (e.g. the Kubernetes
    // project). Because attributes are sometimes compared across devices,
    // a given name is expected to mean the same thing and have the same
    // type on all devices.
    //
    // Attribute names must be either a C-style identifier
    // (e.g. "the_name") or a DNS subdomain followed by a slash ("/")
    // followed by a C-style identifier
    // (e.g. "example.com/the_name"). Attributes whose name does not
    // include the domain prefix are assumed to be part of the driver's
    // domain. Attributes defined by 3rd parties must include the domain
    // prefix.
    //
    // The maximum length for the DNS subdomain is 63 characters (same as
    // for driver names) and the maximum length of the C-style identifier
    // is 32.
    Name string

    // The Go field names below have a Value suffix to avoid a conflict between the
    // field "String" and the corresponding method. That method is required.
    // The Kubernetes API is defined without that suffix to keep it more natural.

    // QuantityValue is a quantity.
    QuantityValue *resource.Quantity
    // BoolValue is a true/false value.
    BoolValue *bool
    // StringValue is a string. Must not be longer than 64 characters.
    StringValue *string
    // VersionValue is a semantic version according to semver.org spec 2.0.0.
    // Must not be longer than 64 characters.
    VersionValue *string
}

type SharedCapacity struct {
    // Name is a unique identifier among all shared capacities managed by the
    // driver in the pool.
    //
    // It is referenced both when defining the total amount of shared capacity
    // that is available, as well as by individual devices when declaring
    // how much of this shared capacity they consume.
    //
    // SharedCapacity names must be a C-style identifier (e.g. "the_name") with
    // a maximum length of 32.
    //
    // By limiting these names to a C-style identifier, the same validation can
    // be used for both these names and the identifier portion of a
    // DeviceAttribute name.
    //
    // +required
    Name string `json:"name"`

    // Capacity is the total capacity of the named resource.
    // This can either represent the total *available* capacity, or the total
    // capacity *consumed*, depending on the context where it is referenced.
    //
    // +required
    Capacity resource.Quantity `json:"capacity"`
}

// CStyleIdentifierMaxLength is the maximum length of a c-style identifier used for naming.
const CStyleIdentifierMaxLength = 32

// DeviceAttributeMaxIDLength is the maximum length of the identifier in a device attribute name (`<domain>/<ID>`).
const DeviceAttributeMaxIDLength = CStyleIdentifierMaxLength

// DeviceAttributeMaxValueLength is the maximum length of a string or version attribute value.
const DeviceAttributeMaxValueLength = 64

// SharedCapacityMaxNameLength is the maximum length of a shared capacity name.
const SharedCapacityMaxNameLength = CStyleIdentifierMaxLength
```

###### ResourceClaim


```go
// ResourceClaim describes which resources (typically one or more devices)
// are needed by a claim consumer.
// Its status tracks whether the claim has been allocated and what the
// resulting attributes are.
//
// This is an alpha type and requires enabling the DynamicResourceAllocation
// feature gate.
type ResourceClaim struct {
    metav1.TypeMeta
    // Standard object metadata
    // +optional
    metav1.ObjectMeta

    // Spec defines what to allocated and how to configure it.
    Spec ResourceClaimSpec

    // Status describes whether the claim is ready for use.
    // +optional
    Status ResourceClaimStatus
}

// Finalizer is the finalizer that gets set for claims
// which were allocated through a builtin controller.
const Finalizer = "dra.k8s.io/delete-protection"
```

The scheduler must set a finalizer in a ResourceClaim before it adds
an allocation. This ensures that an allocated, reserved claim cannot
be removed accidentally by a user.

If storing the status fails, the scheduler will retry on the next
scheduling attempt. If the ResourceClaim gets deleted in the meantime,
the scheduler will not try to schedule again. This situation is handled
by the kube-controller-manager by removing the finalizer.

Force-deleting a ResourceClaim by clearing its finalizers (something that users
should never do without being aware of the consequences) cannot be
prevented. Deleting the entire cluster also leaves resources allocated outside
of the cluster in an allocated state.

```go
type ResourceClaimSpec struct {
    // Requests are individual requests for separate resources for the claim.
    // An empty list is valid and means that the claim can always be allocated
    // without needing anything. A class can be referenced to use the default
    // requests from that class.
    //
    // +listType=atomic
    Requests []Request

    // These constraints must be satisfied by the set of devices that get
    // allocated for the claim.
    Constraints []Constraint

    // This field holds configuration for multiple potential drivers which
    // could satisfy requests in this claim. It is ignored while allocating
    // the claim.
    //
    // +optional
    // +listType=atomic
    Config []ConfigurationParameters

    // Future extension, ignored by older schedulers. This is fine because scoring
    // allows users to define a preference, without making it a hard requirement.
    //
    //
    // Score *SomeScoringStruct
}

// Request is a request for one of many resources required for a claim.
// This is typically a request for a single resource like a device, but can
// also ask for several identical devices. It might get extended to support
// asking for one of several different alternatives.
type Request struct {
    // The name can be used to reference this request in a pod.spec.containers[].resources.claims
    // entry and in a constraint of the claim.
    //
    // Must be a DNS label.
    Name string

    *RequestDetail

    // FUTURE EXTENSION:
    //
    // OneOf contains a list of requests, only one of which must be satisfied.
    // Requests are listed in order of priority.
    //
    // +optional
    // +listType=atomic
    // OneOf []RequestDetail
}

type RequestDetail struct {
    // When referencing a DeviceClass, a request inherits additional
    // configuration and selectors.
    //
    // +optional
    DeviceClassName *string

    // Each selector must be satisfied by a device which is requested.
    //
    // +optional
    // +listType=atomic
    Selectors []Selector

    *Amount // inline, so no extra level in YAML and no separate documentation

    // AdminAccess indicates that this is a claim for administrative access
    // to the device(s). Claims with AdminAccess are expected to be used for
    // monitoring or other management services for a device.  They ignore
    // all ordinary claims to the device with respect to access modes and
    // any resource allocations. Ability to request this kind of access is
    // controlled via ResourceQuota in the resource.k8s.io API.
    //
    // Default is false.
    //
    // +optional
    AdminAccess *bool
}

// Exactly one field must be set.
type Selector struct {
    // This CEL expression must evaluate to true if a
    // device is suitable. This covers qualitative aspects of
    // device selection.
    //
    // The language is as defined in
    // https://kubernetes.io/docs/reference/using-api/cel/
    // with several additions that are specific to device selectors.
    //
    // Attributes of a device are made available through a `device.attributes` map.
    // The value of each entry varies, depending on the attribute
    // that is being looked up.
    //
    // Unknown keys trigger a runtime error.
    //
    // The `device.driverName` string variable can be used to check for a specific
    // driver explicitly in a filter that is meant to work for devices from
    // different vendors. It is provided by Kubernetes and matches the
    // `driverName` from the ResourceSlice which provides the device.
    //
    // The CEL expression is applied to *all* available devices from any driver.
    // The expression has to check for existence of an attribute when it is not
    // certain that it is provided because runtime errors are not automatically
    // treated as "don't select device". Instead, device selection fails completely
    // and reports the error.
    //
    // Some examples:
    //    "memory.dra.example.com" in device.attributes && # Is the attribute available?
    //       device.attributes["memory.dra.example.com"].isGreaterThan(quantity("1Gi")) # >= 1Gi
    //    device.driverName == "dra.example.com" # any device from that driver
    //
    // +optional
    Expression *string
}
```

```
<<[UNRESOLVED @pohly ]>>

The `device.` prefix is included in the CEL environment for the reasons given in
https://github.com/kubernetes-sigs/wg-device-management/issues/23#issuecomment-2160044359.

Whether we really do it like that will be discussed further in that issue and/or during
the implementation phase. Dropping it is not a conceptual change.

<<[/UNRESOLVED]>>
```

```go

// Exactly one field must be set.
type Amount struct {
    // All, if set, asks for all devices matching the selectors.
    // Allocation fails if not all of them are available, unless admin
    // access is requested. Admin access is granted also for
    // devices which are in use.
    //
    // The default if `all` and `count` are unset is to allocate one device.
    //
    // +optional
    All bool

    // A fixed number of devices which match the same selectors.
    //
    // The default if `all` and `count` are unset is to allocate one device.
    //
    // +optional
    Count *int64
}

// Besides the request name slice, constraint must have one and only one field set.
type Constraint struct {
    // The constraint applies to devices in these requests. A single entry is okay
    // and used when that request is for multiple devices.
    //
    // If empty, the constrain applies to all devices in the claim.
    //
    // +optional
    RequestNames []string

    // The devices must have this attribute and its value must be the same.
    //
    // For example, if you specified "numa.dra.example.com" (a hypothetical example!),
    // then only devices in the same NUMA node will be chosen.
    //
    // +optional
    // +listType=atomic
    MatchAttribute *string

    // Future extension, not part of the current design:
    // A CEL expression which compares different devices and returns
    // true if they match.
    //
    // Because it would be part of a one-of, old schedulers will not
    // accidentally ignore this additional, for them unknown match
    // criteria.
    //
    // matcher string
}

// Besides the request name slice, ConfigurationParameters must have one and only one field set.
type ConfigurationParameters struct {
    // The constraint applies to devices in these requests.
    //
    // If empty, the configuration applies to all devices in the claim.
    //
    // +optional
    RequestNames []string

    Opaque *OpaqueConfigurationParameters
}

// OpaqueConfigurationParameters contains configuration parameters for a driver.
type OpaqueConfigurationParameters struct {
    // DriverName is used to determine which kubelet plugin needs
    // to be passed these configuration parameters.
    //
    // An admission webhook provided by the driver developer could use this
    // to decide whether it needs to validate them.
    //
    // Must be a DNS subdomain and should end with a DNS domain owned by the
    // vendor of the driver.
    DriverName string

    // Parameters can contain arbitrary data. It is the responsibility of
    // the driver developer to handle validation and versioning. Typically this
    // includes self-identification and a version ("kind" + "apiVersion" for
    // Kubernetes types), with conversion between different versions.
    Parameters runtime.RawExtension
}
```

The `DeviceClassName` field may be left empty. The request itself can specify which
driver needs to provide devices. There are some corner cases:
- Empty `claim.requests`: this is a "null request" which can be satisfied without
  allocating any device.
- Empty `claim.requests[].deviceClassName` and empty `selectors`: this is a request
  which can be satisfied by any device. This does not really make sense and
  is treated as an error because it is probably a mistake.
- Non-empty `deviceClassName`, empty class: this is handled the same way.

There is no default DeviceClass. If that is desirable, then it can be
implemented with a mutating and/or admission webhook.

```go
// ResourceClaimStatus tracks whether the resource has been allocated and what
// the result of that was.
type ResourceClaimStatus struct {
    // Allocation is set once the claim has been allocated successfully.
    // +optional
    Allocation *AllocationResult

    // ReservedFor indicates which entities are currently allowed to use
    // the claim. A Pod which references a ResourceClaim which is not
    // reserved for that Pod will not be started. A claim that is in
    // use or might be in use because it has been reserved must not get
    // deallocated.
    //
    // In a cluster with multiple scheduler instances, two pods might get
    // scheduled concurrently by different schedulers. When they reference
    // the same ResourceClaim which already has reached its maximum number
    // of consumers, only one pod can be scheduled.
    //
    // Both schedulers try to add their pod to the claim.status.reservedFor
    // field, but only the update that reaches the API server first gets
    // stored. The other one fails with an error and the scheduler
    // which issued it knows that it must put the pod back into the queue,
    // waiting for the ResourceClaim to become usable again.
    //
    // There can be at most 32 such reservations. This may get increased in
    // the future, but not reduced.
    //
    // +listType=map
    // +listMapKey=uid
    // +patchStrategy=merge
    // +patchMergeKey=uid
    // +optional
    ReservedFor []ResourceClaimConsumerReference
}
```

##### DeviceClass

```go
// DeviceClass is a vendor or admin-provided resource that contains
// device configuration and selectors. It can be referenced in
// the device requests of a claim to apply these presets.
// Cluster scoped.
type DeviceClass struct {
    metav1.TypeMeta
    // Standard object metadata
    // +optional
    metav1.ObjectMeta

    // Each selector must be satisfied by a device which is claimed via this class.
    //
    // +optional
    // +listType=atomic
    Selectors []Selector

    // Config defines configuration parameters that apply to each device that is claimed via this class.
    // Some classses may potentially be satisfied by multiple drivers, so each instance of a vendor
    // configuration applies to exactly one driver.
    //
    // They are passed to the driver, but are not consider while allocating the claim.
    //
    // +optional
    // +listType=atomic
    Config []ConfigurationParameters
}
```

##### Allocation result

```go
// AllocationResult contains attributes of an allocated resource.
type AllocationResult struct {
    // Results lists all allocated devices.
    //
    // +listType=atomic
    Results []RequestAllocationResult

    // This field is a combination of all the claim and class configuration parameters.
    // Drivers can distinguish between those based on a flag.
    //
    // This includes configuration parameters for drivers which have no allocated
    // devices in the result because it is up to the drivers which configuration
    // parameters they support. They can silently ignore unknown configuration
    // parameters.
    //
    // +optional
    // +listType=atomic
    Config []AllocationConfigurationParameters

    // Setting this field is optional. If unset, the allocated devices are available everywhere.
    //
    // +optional
    AvailableOnNodes *v1.NodeSelector
}

// AllocationResultsMaxSize represents the maximum number of
// entries in allocation.results.
const AllocationResultsMaxSize = 32

// RequestAllocationResult contains the allocation result for one request.
type RequestAllocationResult struct {
    // DriverName specifies the name of the DRA driver whose kubelet
    // plugin should be invoked to process the allocation once the claim is
    // needed on a node.
    //
    // Must be a DNS subdomain and should end with a DNS domain owned by the
    // vendor of the driver.
    DriverName string

    // RequestName identifies the request in the claim which caused this
    // device to be allocated. If a request was satisfied by multiple
    // devices, they are indexed starting with zero in the order
    // in which they are listed in the results.
    RequestName string

    // This name together with the driver name and the device name field
    // identify which device was allocated (`<driver name>/<node name>/<device name>`).
    //
    // Must not be longer than 253 characters and may contain one or more
    // DNS sub-domains separated by slashes.
    //
    // +optional
    PoolName string

    // DeviceName references one device instance via its name in the driver's
    // resource pool. It must be a DNS label.
    DeviceName string
}

// AllocationConfigurationParameters is a superset of ConfigurationParameters which
// is used in AllocationResult to track where the parameters came from.
type AllocationConfigurationParameters struct {
    // Admins is true if the source of the configuration was a class and thus
    // not something that a normal user would have been able to set.
    Admin bool

    ConfigurationParameters
}
```

##### ResourceClaimTemplate

```go
// ResourceClaimTemplate is used to produce ResourceClaim objects.
type ResourceClaimTemplate struct {
    metav1.TypeMeta
    // Standard object metadata
    // +optional
    metav1.ObjectMeta

    // Describes the ResourceClaim that is to be generated.
    //
    // This field is immutable. A ResourceClaim will get created by the
    // control plane for a Pod when needed and then not get updated
    // anymore.
    Spec ResourceClaimTemplateSpec
}

// ResourceClaimTemplateSpec contains the metadata and fields for a ResourceClaim.
type ResourceClaimTemplateSpec struct {
    // ObjectMeta may contain labels and annotations that will be copied into the PVC
    // when creating it. No other fields are allowed and will be rejected during
    // validation.
    // +optional
    metav1.ObjectMeta

    // Spec for the ResourceClaim. The entire content is copied unchanged
    // into the ResourceClaim that gets created from this template. The
    // same fields as in a ResourceClaim are also valid here.
    Spec ResourceClaimSpec
}
```

##### ResourceQuota

At the moment, this type is only used to control whether admin access is
allowed in a namespace. Other aspects may get added later.

Admin access can be checked in an admission plugin. Other limitations may have
to be checked at allocation time, either because they depend on device
attributes which cannot be assumed to be known at the time of admission of a
claim or because the quota itself influences device selection. For example, a
future "give me device A if available, otherwise device B" request might not be
able to provide A because usage of A over quota while device B is still under
quota.

```go
// Quota controls whether a ResourceClaim may get allocated.
// Quota is namespaced and applies to claims within the same namespace.
type Quota struct {
    metav1.TypeMeta
    // Standard object metadata.
    metav1.ObjectMeta

    Spec QuotaSpec
}

type QuotaSpec struct {
    // Controls whether devices may get allocated with admin access
    // (concurrent with normal use, potentially privileged access permissions
    // depending on the driver). If multiple quota objects exist and at least one
    // has a true value, access will be allowed. The default to deny such access.
    //
    // +optional
    AllowAdminAccess bool `json:"allowManagementAccess,omitempty"`

    // Stretch goals for 1.31:
    //
    // - maximum number of devices matching a selector
    // - maximum sum of a certain quantity attribute
    //
    // These are additional restrictions when checking whether a device
    // instance can satisfy a request. Creating a claim is always allowed,
    // but allocating it fails when the quota is currently exceeded.

    // Other useful future extensions (>= 1.32):

    // DeviceLimits is a CEL expression which take the currently allocated
    // devices and their attributes and some new allocations as input and
    // returns false if those allocations together are not permitted in the
    // namespace.
    //
    // DeviceLimits string

    // A class listed in DeviceClassDenyList must not be used in this
    // namespace. This can be useful for classes which contain
    // configuration parameters that a user in this namespace should not have
    // access to.
    //
    // DeviceClassDenyList []string

    // A class listed in ResourceClassAllowList may be used in this namespace
    // even when that class is marked as "privileged". Normally classes
    // are not privileged and using them does not require explicit listing
    // here, but some classes may contain more sensitive configuration parameters
    // that not every user should have access to.
    //
    // DeviceClassAllowList []string
}
```

##### Object references

```go
// ResourceClaimConsumerReference contains enough information to let you
// locate the consumer of a ResourceClaim. The user must be a resource in the same
// namespace as the ResourceClaim.
type ResourceClaimConsumerReference struct {
    // APIGroup is the group for the resource being referenced. It is
    // empty for the core API. This matches the group in the APIVersion
    // that is used when creating the resources.
    // +optional
    APIGroup string
    // Resource is the type of resource being referenced, for example "pods".
    Resource string
    // Name is the name of resource being referenced.
    Name string
    // UID identifies exactly one incarnation of the resource.
    UID types.UID
}
```

`ResourceClaimConsumerReference` is typically set by the control plane and
therefore uses the more technically correct "resource" name instead of the
more user-friendly "kind".

#### core

```go
type PodSpec {
   ...
    // ResourceClaims defines which ResourceClaims must be allocated
    // and reserved before the Pod is allowed to start. The resources
    // will be made available to those containers which consume them
    // by name.
    //
    // This is an alpha field and requires enabling the
    // DynamicResourceAllocation feature gate.
    //
    // This field is immutable.
    //
    // +featureGate=DynamicResourceAllocation
    // +optional
    ResourceClaims []PodResourceClaim
   ...
}

type  ResourceRequirements {
   Limits ResourceList
   Requests ResourceList
   ...
    // Claims lists the names of resources, defined in spec.resourceClaims,
    // that are used by this container.
    //
    // This is an alpha field and requires enabling the
    // DynamicResourceAllocation feature gate.
    //
    // This field is immutable.
    //
    // +featureGate=DynamicResourceAllocation
    // +optional
    Claims []ResourceClaim
}

// ResourceClaim references one entry in PodSpec.ResourceClaims.
type ResourceClaim struct {
    // Name must match the name of one entry in pod.spec.resourceClaims of
    // the Pod where this field is used. It makes that resource available
    // inside a container.
    Name string

    // A name set in claim.spec.requests[].name.
    // +optional
    RequestName string
}
```

`Claims` is a list of structs with a single `Name` element because that struct
can be extended later, for example to add parameters that influence how the
resource is made available to a container. This wouldn't be possible if
it was a list of strings.

```go
// PodResourceClaim references exactly one ResourceClaim through a ClaimSource.
// It adds a name to it that uniquely identifies the ResourceClaim inside the Pod.
// Containers that need access to the ResourceClaim reference it with this name.
type PodResourceClaim struct {
    // Name must match the name of one entry in pod.spec.resourceClaims of
    // the Pod where this field is used. It makes that resource available
    // inside a container. Must be a DNS label.
    Name string

    ClaimSource // inlined to avoid one level of indirectin in the YAML.
}

// ClaimSource describes a reference to a ResourceClaim.
//
// Exactly one of these fields should be set.  Consumers of this type must
// treat an empty object as if it has an unknown value.
type ClaimSource struct {
    // The ResourceClaim named here is shared by all pods which
    // reference it. The lifecycle of that ResourceClaim is not
    // managed by Kubernetes.
    //
    // +optional
    ResourceClaimName *string

    // ResourceClaimTemplateName is the name of a ResourceClaimTemplate
    // object in the same namespace as this pod.
    //
    // The template will be used to create a new ResourceClaim, which will
    // be bound to this pod. When this pod is deleted, the ResourceClaim
    // will also be deleted. The pod name and resource name, along with a
    // generated component, will be used to form a unique name for the
    // ResourceClaim, which will be recorded in pod.status.resourceClaimStatuses.
    //
    // This field is immutable and no changes will be made to the
    // corresponding ResourceClaim by the control plane after creating the
    // ResourceClaim.
    ResourceClaimTemplateName *string
}

struct PodStatus {
    ...
    // Status of resource claims.
    // +featureGate=DynamicResourceAllocation
    // +optional
    ResourceClaimStatuses []PodResourceClaimStatus
}

// PodResourceClaimStatus is stored in the PodStatus for each PodResourceClaim
// which references a ResourceClaimTemplate. It stores the generated name for
// the corresponding ResourceClaim.
type PodResourceClaimStatus struct {
    // Name uniquely identifies this resource claim inside the pod.
    // This must match the name of an entry in pod.spec.resourceClaims,
    // which implies that the string must be a DNS_LABEL.
    Name string

    // ResourceClaimName is the name of the ResourceClaim that was
    // generated for the Pod in the namespace of the Pod. If this is
    // unset, then generating a ResourceClaim was not necessary. The
    // pod.spec.resourceClaims entry can be ignored in this case.
    ResourceClaimName *string
}
```

### kube-controller-manager

The code that creates a ResourceClaim from a ResourceClaimTemplate started
as an almost verbatim copy of the [generic ephemeral volume
code](https://github.com/kubernetes/kubernetes/tree/master/pkg/controller/volume/ephemeral),
just with different types. Later, generating the name of the ephemeral ResourceClaim
was added.

kube-controller-manager needs [RBAC
permissions](https://github.com/kubernetes/kubernetes/commit/ff3e5e06a79bc69ad3d7ccedd277542b6712514b#diff-2ad93af2302076e0bdb5c7a4ebe68dd3188eee8959c72832181a7597417cd196) that allow creating and updating ResourceClaims.

kube-controller-manager also removes `claim.status.reservedFor` entries that reference
removed pods or pods that have completed ("Phase" is "done" or will never start).
This is required for pods because kubelet does not have write
permission for ResourceClaimStatus. Pods as user is the common case, so special
code based on a shared pod informer will handle it. Other consumers
need to be handled by whatever controller added them.

In addition to updating `claim.status.reservedFor`, kube-controller-manager also
removes the allocation from ResourceClaims that are no longer in use.
Updating the claim during deallocation will be observed by kube-scheduler and
tells it that it can use the capacity set aside for the claim
again. kube-controller-manager itself doesn't need to support specific structured
models.

### kube-scheduler

The scheduler plugin for ResourceClaims ("claim plugin" in this section)
needs to implement several extension points. It is responsible for
ensuring that a ResourceClaim is allocated and reserved for a Pod before
the final binding of a Pod to a node.

The following extension points are implemented in the new claim plugin. Except
for some unlikely edge cases (see below) there are no API calls during the main
scheduling cycle. Instead, the plugin collects information and updates the
cluster in the separate goroutine which invokes PreBind.


#### EventsToRegister

This registers all cluster events that might make an unschedulable pod
schedulable, like creating a claim that the pod needs or finishing the
allocation of a claim.

[Queuing hints](https://github.com/kubernetes/enhancements/issues/4247) are
supported. These are callbacks that can limit the effect of a cluster event to
specific pods. For example, allocating a claim only makes those pods
scheduleable which reference the claim. There is no need to try scheduling a pod
which waits for some other claim. Hints are also used to trigger the next
scheduling cycle for a pod immediately when some expected and require event
like "drivers have provided information" occurs, instead of forcing the pod to
go through the backoff queue and the usually 5 second long delay associated
with that.

Queuing hints are an optional feature of the scheduler, with (as of Kubernetes
1.29) their own `SchedulerQueueingHints` feature gate that defaults to
off. When turned off, performance of scheduling pods with resource claims is
slower compared to a cluster configuration where they are turned on.

#### PreEnqueue

This checks whether all claims referenced by a pod exist. If they don't,
scheduling the pod has to wait until the kube-controller-manager or user create
the claims. PreEnqueue tries to finish quickly because it is called from
event handlers, so not everything is checked.

#### Pre-filter

This is a more thorough version of the checks done by PreEnqueue. It ensures
that all information that is needed (ResourceClaim, ResourceClass, parameters)
is available.

Another reason why a Pod might not be schedulable is when it depends on claims
which are in the process of being allocated. That process starts in Reserve and
ends in PreBind or Unreserve (see below).

It then prepares for filtering by converting information stored in various
places (node filter in ResourceClass, available resources in ResourceSlices,
allocated resources in ResourceClaim statuses, in-flight allocations) into a
format that can be used efficiently by Filter.

#### Filter

This checks whether the given node has access to those ResourceClaims which
were already allocated. For ResourceClaims that were not, it checks that the
allocation can succeed for a node.

#### Post-filter

This is called when no suitable node could be found. If the Pod depends on ResourceClaims with delayed
allocation, then deallocating one or more of these ResourceClaims may make the
Pod schedulable after allocating the resource elsewhere. Therefore each
ResourceClaim with delayed allocation is checked whether all of the following
conditions apply:
- allocated
- not currently in use
- it was the reason why some node could not fit the Pod, as recorded earlier in
  Filter

One of the ResourceClaims satisfying these criteria is picked randomly and gets
deallocated by clearing the allocation in its status. This may make it possible to run the Pod
elsewhere. If it still doesn't help, deallocation may continue with another
ResourceClaim, if there is one.

This is currently using blocking API calls. It's quite rare because this
situation can only arise when there are multiple claims per pod and writing
the status of one of them fails, thus leaving the other claims in the
allocated state.

#### Reserve

A node has been chosen for the Pod.

For each unallocated claim, the actual allocation result is determined now. To
avoid blocking API calls, that result is not written to the status yet. Instead,
it gets stored in a map of in-flight claims.

#### PreBind

This is called in a separate goroutine. The plugin now checks all the
information gathered earlier and updates the cluster accordingly. If some
some API request fails now, PreBind fails and the pod must be
retried.

Claims whose status got written back get removed from the in-flight claim map.

#### Unreserve

The claim plugin removes the Pod from the `claim.status.reservedFor` field if
set there because it cannot be scheduled after all.

This is necessary to prevent a deadlock: suppose there are two stand-alone
claims that only can be used by one pod at a time and two pods which both
reference them. Both pods will get scheduled independently, perhaps even by
different schedulers. When each pod manages to allocate and reserve one claim,
then neither of them can get scheduled because they cannot reserve the other
claim.

Giving up the reservations in Unreserve means that the next pod scheduling
attempts have a chance to succeed. It's non-deterministic which pod will win,
but eventually one of them will. Not giving up the reservations would lead to a
permanent deadlock that somehow would have to be detected and resolved to make
progress.

All claims get removed from the in-flight claim map.

Unreserve is called in two scenarios:
- In the main goroutine when scheduling a pod has failed: in that case the plugin's
  Reserve call hasn't actually changed the claim status yet, so there is nothing
  that needs to be rolled back.
- After binding has failed: this runs in a goroutine, so reverting the
  `claim.status.reservedFor` with a blocking call is acceptable.

### kubelet

#### Communication between kubelet and kubelet plugin

kubelet plugins are discovered through the [kubelet plugin registration
mechanism](https://kubernetes.io/docs/concepts/extend-kubernetes/compute-storage-net/device-plugins/#device-plugin-registration). A
new "ResourcePlugin" type will be used in the Type field of the
[PluginInfo](https://pkg.go.dev/k8s.io/kubelet/pkg/apis/pluginregistration/v1#PluginInfo)
response to distinguish the plugin from device and CSI plugins.

Under the advertised Unix Domain socket the kubelet plugin provides the
k8s.io/kubelet/pkg/apis/dra gRPC interface. It was inspired by
[CSI](https://github.com/container-storage-interface/spec/blob/master/spec.md),
with “volume” replaced by “resource” and volume specific parts removed.

#### Version skew

Previously, kubelet retrieved ResourceClaims and published ResourceSlices on
behalf of DRA drivers on the node. The information included in those got passed
between API server, kubelet, and kubelet plugin using the version of the
resource.k8s.io used by the kubelet. Combining a kubelet using some older API
version with a plugin using a new version was not possible because conversion
of the resource.k8s.io types is only supported in the API server and an old
kubelet wouldn't know about a new version anyway.

Keeping kubelet at some old release while upgrading the control and DRA drivers
is desirable and officially supported by Kubernetes. To support the same when
using DRA, the kubelet now leaves [ResourceSlice
handling](#publishing-node-resources) almost entirely to the plugins. The
remaining calls are done with whatever resource.k8s.io API version is the
latest known to the kubelet. To support version skew, support for older API
versions must be preserved as far back as support for older kubelet releases is
desired.

#### Security

The daemonset of a DRA driver must be configured to have a service account
which grants the following permissions:
- get/list/watch/create/update/oatch/delete ResourceSlice
- get ResourceClaim
- get Node

Ideally, write access to ResourceSlice should be limited to objects belonging
to the node. This is possible with a [validating admission
policy](https://kubernetes.io/docs/reference/access-authn-authz/validating-admission-policy/). As
this is not a core feature of the DRA KEP, instructions for how to do that will
not be included here. Instead, the DRA example driver will provide an example
and documentation.

#### Managing resources

kubelet must ensure that resources are ready for use on the node before running
the first Pod that uses a specific resource instance and make the resource
available elsewhere again when the last Pod has terminated that uses it. For
both operations, kubelet calls a kubelet plugin as explained in the next
section.

Pods that are not listed in ReservedFor or where the ResourceClaim doesn't
exist at all must not be allowed to run. Instead, a suitable event must be
emitted which explains the problem. Such a situation can occur as part of
downgrade scenarios.

If this was the last Pod on the node that uses the specific
resource instance, then NodeUnprepareResource (see below) must have been called
successfully before allowing the pod to be deleted. This ensures that network-attached resource are available again
for other Pods, including those that might get scheduled to other nodes. It
also signals that it is safe to deallocate and delete the ResourceClaim.

![kubelet](./kubelet.png)

##### NodePrepareResource

This RPC is called by the kubelet when a Pod that wants to use the specified
resource is scheduled on a node. The Plugin SHALL assume that this RPC will be
executed on the node where the resource will be used.

ResourceClaim.meta.Namespace, ResourceClaim.meta.UID, ResourceClaim.Name are
passed to the Plugin as parameters to identify
the claim and perform resource preparation.

The Plugin SHALL return fully qualified device name[s].

The Plugin SHALL ensure that there are json file[s] in CDI format
for the allocated resource. These files SHALL be used by runtime to
update runtime configuration before creating containers that use the
resource.

This operation SHALL do as little work as possible as it’s called
after a pod is scheduled to a node. All potentially failing operations
SHALL be done during allocation phase.

This operation MUST be idempotent. If the resource corresponding to
the `resource_id` has already been prepared, the Plugin MUST reply `0
OK`.

If this RPC failed, or kubelet does not know if it failed or not, it
MAY choose to call `NodePrepareResource` again, or choose to call
`NodeUnprepareResource`.

On a successful call this RPC should return set of fully qualified
CDI device names, which kubelet MUST pass to the runtime through the CRI
protocol. For version v1alpha3, the RPC should return multiple sets of
fully qualified CDI device names, one per claim that was sent in the input parameters.

```protobuf
message NodePrepareResourcesRequest {
     // The list of ResourceClaims that are to be prepared.
     repeated Claim claims = 1;
}

message Claim {
    // The ResourceClaim namespace (ResourceClaim.meta.Namespace).
    // This field is REQUIRED.
    string namespace = 1;
    // The UID of the Resource claim (ResourceClaim.meta.UUID).
    // This field is REQUIRED.
    string uid = 2;
    // The name of the Resource claim (ResourceClaim.meta.Name)
    // This field is REQUIRED.
    string name = 3;
}
```

The allocation result is intentionally not included here. The content of that
field is version-dependent. The kubelet would need to discover in which version
each plugin wants the data, then potentially get the claim multiple times
because only the apiserver can convert between versions. Instead, each plugin
is required to get the claim itself using its own credentials. In the most common
case of one plugin per claim, that doubles the number of GETs for each claim
(once by the kubelet, once by the plugin).

```
message NodePrepareResourcesResponse {
    // The ResourceClaims for which preparation was done
    // or attempted, with claim_uid as key.
    //
    // It is an error if some claim listed in NodePrepareResourcesRequest
    // does not get prepared. NodePrepareResources
    // will be called again for those that are missing.
    map<string, NodePrepareResourceResponse> claims = 1;
}

message NodePrepareResourceResponse {
    // These are the additional devices that kubelet must
    // make available via the container runtime. A claim
    // may have zero or more requests and each request
    // may have zero or more devices.
    repeated Device cdi_devices = 1 [(gogoproto.customname) = "CDIDevices"];
    // If non-empty, preparing the ResourceClaim failed.
    // cdi_devices is ignored in that case.
    string error = 2;
}

message Device {
    // The request in the claim that this device is associated with.
    string request_name = 1;

    // A single device instance may map to several CDI device IDs.
    // None is also valid.
    repeated string cdi_ids = 2;
}
```

The `request_name` here allows kubelet to find the right CDI IDs for a
container which uses device(s) from a specific request instead of all devices
of a claim.

CRI protocol MUST be extended for this purpose:

 * CDIDevice structure was added to the CRI specification
```protobuf
// CDIDevice specifies a CDI device information.
message CDIDevice {
    // Fully qualified CDI device name
    // for example: vendor.com/gpu=gpudevice1
    // see more details in the CDI specification:
    // https://github.com/container-orchestrated-devices/container-device-interface/blob/main/SPEC.md
    string name = 1;
}
```

 * CDI devices have been added to the ContainerConfig structure:

```protobuf
// ContainerConfig holds all the required and optional fields for creating a
// container.
message ContainerConfig {
    // Metadata of the container. This information will uniquely identify the
    // container, and the runtime should leverage this to ensure correct
    // operation. The runtime may also use this information to improve UX, such
    // as by constructing a readable name.
    ContainerMetadata metadata = 1 ;
    // Image to use.
    ImageSpec image = 2;
    // Command to execute (i.e., entrypoint for docker)
    repeated string command = 3;
...
    // CDI devices for the container.
    repeated CDIDevice cdi_devices = 17;
}
```

At the time of Kubernetes 1.31, most container runtimes [support this
field](https://github.com/kubernetes/kubernetes/issues/125210) so it can be
used instead of some earlier approach with annotations.

###### NodePrepareResource Errors

If the plugin is unable to complete the NodePrepareResource call
successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST
return the specified gRPC error code.  Kubelet MUST implement the
specified error recovery behavior when it encounters the gRPC error
code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Resource does not exist | 5 NOT_FOUND | Indicates that a resource corresponding to the specified `resource_id` does not exist. | Caller MUST verify that the `resource_id` is correct and that the resource is accessible and has not been deleted before retrying with exponential back off. |


##### NodeUnprepareResources

A Kubelet Plugin MUST implement this RPC call. This RPC is a reverse
operation of `NodePrepareResource`. This RPC MUST undo the work by
the corresponding `NodePrepareResource`. This RPC SHALL be called by
kubelet at least once for each successful `NodePrepareResource`. The
Plugin SHALL assume that this RPC will be executed on the node where
the resource is being used.

This RPC is called by the kubelet when the last Pod using the resource is being
deleted or has reached a final state ("Phase" is "done").

This operation MUST be idempotent. If this RPC failed, or kubelet does
not know if it failed or not, it can choose to call
`NodeUnprepareResource` again.

```protobuf
message NodeUnprepareResourcesRequest {
    // The list of ResourceClaims that are to be unprepared.
    repeated Claim claims = 1;
}

message NodeUnprepareResourcesResponse {
    // The ResourceClaims for which preparation was reverted.
    // The same rules as for NodePrepareResourcesResponse.claims
    // apply.
    map<string, NodeUnprepareResourceResponse> claims = 1;
}

message NodeUnprepareResourceResponse {
    // If non-empty, unpreparing the ResourceClaim failed.
    string error = 1;
}
```

###### NodeUnprepareResource Errors

If the plugin is unable to complete the NodeUprepareResource call
successfully, it MUST return a non-ok gRPC code in the gRPC status.
If the conditions defined below are encountered, the plugin MUST
return the specified gRPC error code.  Kubelet MUST implement the
specified error recovery behavior when it encounters the gRPC error
code.

| Condition | gRPC Code | Description | Recovery Behavior |
|-----------|-----------|-------------|-------------------|
| Resource does not exist | 5 NOT_FOUND | Indicates that a resource corresponding to the specified `resource_id` does not exist. | Caller MUST verify that the `resource_id` is correct and that the resource is accessible and has not been deleted before retrying with exponential back off. |


### Simulation with CA

The usual call sequence of a scheduler plugin when used in the scheduler is
at program startup:
- instantiate plugin
- EventsToRegister

For each new pod:
- PreEnqueue

For each pod that is ready to be scheduled, one pod at a time:
- PreFilter, Filter, etc.

Scheduling a pod gets finalized with:
- Reserve, PreBind, Bind

CA works a bit differently. It identifies all pending pods,
takes a snapshot of the current cluster state, and then simulates the effect
of scheduling those pods with additional nodes added to the cluster. To
determine whether a pod fits into one of these simulated nodes, it
uses the same PreFilter and Filter plugins as the scheduler. Other extension
points (Reserve, Bind) are not used. Plugins which modify the cluster state
therefore need a different way of recording the result of scheduling
a pod onto a node.

One option for this is to add a new optional plugin interface that is
implemented by the dynamic resource plugin. Through that interface the
autoscaler can then inform the plugin about events like starting simulation,
binding pods, and adding new nodes. With this approach, the autoscaler doesn't
need to know what the persistent state of each plugin is.

Another option is to extend the state that the autoscaler keeps for
plugins. The plugin then shouldn't need to know that it runs inside the
autoscaler. This implies that the autoscaler will have to call Reserve and
PreBind as that is where the state gets updated.

Which of these options is chosen will be decided during the implementation
phase. Autoscalers which don't use the in-tree scheduler plugin will have
to implement a similar logic.

### Test Plan

[X] I/we understand the owners of the involved components may require updates to
existing tests to make this code solid enough prior to committing the changes necessary
to implement this enhancement.

##### Prerequisite testing updates

None.

##### Unit tests

<!--
In principle every added code should have complete unit test coverage, so providing
the exact set of tests will not bring additional value.
However, if complete unit test coverage is not possible, explain the reason of it
together with explanation why this is acceptable.
-->

<!--
Additionally, for Alpha try to enumerate the core package you will be touching
to implement this enhancement and provide the current unit coverage for those
in the form of:
- <package>: <date> - <current test coverage>
The data can be easily read from:
https://testgrid.k8s.io/sig-testing-canaries#ci-kubernetes-coverage-unit

This can inform certain test coverage improvements that we want to do before
extending the production code to implement this enhancement.
-->

- `k8s.io/kubernetes/pkg/scheduler`: 2022-05-24 - 75.0%
- `k8s.io/kubernetes/pkg/scheduler/framework`: 2022-05-24 - 76.3%
- `k8s.io/kubernetes/pkg/controller`: 2022-05-24 - 69.4%
- `k8s.io/kubernetes/pkg/kubelet`: 2022-05-24 - 64.5%

##### Integration tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html
-->

##### e2e tests

<!--
This question should be filled when targeting a release.
For Alpha, describe what tests will be added to ensure proper quality of the enhancement.

For Beta and GA, add links to added tests together with links to k8s-triage for those tests:
https://storage.googleapis.com/k8s-triage/index.html

We expect no non-infra related flakes in the last month as a GA graduation criteria.
-->

End-to-end testing depends on a working DRA driver and a container runtime
with CDI support. A [test driver](https://github.com/kubernetes/kubernetes/tree/master/test/e2e/dra/test-driver)
was developed in parallel to developing the
code in Kubernetes.

That test driver simply takes parameters from ResourceClass
and ResourceClaim and turns them into environment variables that then get
checked inside containers. Tests for different behavior of an driver in various
scenarios can be simulated by running the control-plane part of it in the E2E
test itself. For interaction with kubelet, proxying of the gRPC interface can
be used, as in the
[csi-driver-host-path](https://github.com/kubernetes-csi/csi-driver-host-path/blob/16251932ab81ad94c9ec585867104400bf4f02e5/cmd/hostpathplugin/main.go#L61-L63):
then the kubelet plugin runs on the node(s), but the actual processing of gRPC
calls happens inside the E2E test.

All tests that don't involve actually running a Pod can become part of
conformance testing. Those tests that run Pods cannot be because CDI support in
runtimes is not required.

For beta:
- pre-merge with kind (optional, triggered for code which has an impact on DRA): https://testgrid.k8s.io/sig-node-dynamic-resource-allocation#pull-kind-dra
- periodic with kind: https://testgrid.k8s.io/sig-node-dynamic-resource-allocation#ci-kind-dra
- pre-merge with CRI-O: https://testgrid.k8s.io/sig-node-dynamic-resource-allocation#pull-node-dra
- periodic with CRI-O: https://testgrid.k8s.io/sig-node-dynamic-resource-allocation#ci-node-e2e-crio-dra


### Graduation Criteria

#### Alpha

- Feature implemented behind a feature flag
- Initial e2e tests completed and enabled

#### Beta

- Gather feedback from developers and surveys
- Fully implemented
- Additional tests are in Testgrid and linked in KEP

#### GA

- 3 examples of real-world usage
- Allowing time for feedback

[conformance tests]: https://git.k8s.io/community/contributors/devel/sig-architecture/conformance-tests.md

### Upgrade / Downgrade Strategy

Because of the strongly-typed versioning of resource attributes and allocation
results, the gRPC interface between kubelet and the DRA driver is tied to the
version of the supported structured models. A DRA driver has to implement all
gRPC interfaces that might be used by older releases of kubelet. The same
applies when upgrading kubelet while the DRA driver remains at an older
version.

### Version Skew Strategy

Ideally, the latest release of a DRA driver should be used and it should
support a wide range of structured type versions. Then problems due to version
skew are less likely to occur.

## Production Readiness Review Questionnaire

### Feature Enablement and Rollback

###### How can this feature be enabled / disabled in a live cluster?

- [X] Feature gate
  - Feature gate name: DynamicResourceAllocation
  - Components depending on the feature gate:
    - kube-apiserver
    - kubelet
    - kube-scheduler
    - kube-controller-manager

###### Does enabling the feature change any default behavior?

No.

###### Can the feature be disabled once it has been enabled (i.e. can we roll back the enablement)?

Yes. Applications that were already deployed and are running will continue to
work, but they will stop working when containers get restarted because those
restarted containers won't have the additional resources.

###### What happens if we reenable the feature if it was previously rolled back?

Pods might have been scheduled without handling resources. Those Pods must be
deleted to ensure that the re-created Pods will get scheduled properly.

###### Are there any tests for feature enablement/disablement?

<!--
The e2e framework does not currently support enabling or disabling feature
gates. However, unit tests in each component dealing with managing data, created
with and without the feature, are necessary. At the very least, think about
conversion tests if API types are being modified.

Additionally, for features that are introducing a new API field, unit tests that
are exercising the `switch` of feature gate itself (what happens if I disable a
feature gate after having objects written with the new field) are also critical.
You can take a look at one potential example of such test in:
https://github.com/kubernetes/kubernetes/pull/97058/files#diff-7826f7adbc1996a05ab52e3f5f02429e94b68ce6bce0dc534d1be636154fded3R246-R282
-->

Tests for apiserver will cover disabling the feature. This primarily matters
for the extended PodSpec: the new fields must be preserved during updates even
when the feature is disabled.

### Rollout, Upgrade and Rollback Planning

<!--
This section must be completed when targeting beta to a release.
-->

###### How can a rollout or rollback fail? Can it impact already running workloads?

<!--
Try to be as paranoid as possible - e.g., what if some components will restart
mid-rollout?

Be sure to consider highly-available clusters, where, for example,
feature flags will be enabled on some API servers and not others during the
rollout. Similarly, consider large clusters and how enablement/disablement
will rollout across nodes.
-->

###### What specific metrics should inform a rollback?

<!--
What signals should users be paying attention to when the feature is young
that might indicate a serious problem?
-->

###### Were upgrade and rollback tested? Was the upgrade->downgrade->upgrade path tested?

<!--
Describe manual testing that was done and the outcomes.
Longer term, we may want to require automated upgrade/rollback tests, but we
are missing a bunch of machinery and tooling and can't do that now.
-->

###### Is the rollout accompanied by any deprecations and/or removals of features, APIs, fields of API types, flags, etc.?

<!--
Even if applying deprecation policies, they may still surprise some users.
-->

### Monitoring Requirements

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### How can an operator determine if the feature is in use by workloads?

<!--
Ideally, this should be a metric. Operations against the Kubernetes API (e.g.,
checking if there are objects with field X set) may be a last resort. Avoid
logs or events for this purpose.
-->

Metrics in kube-scheduler (names to be decided):
- number of classes using structured parameters
- number of claims which currently are allocated with structured parameters

###### How can someone using this feature know that it is working for their instance?

<!--
For instance, if this is a pod-related feature, it should be possible to determine if the feature is functioning properly
for each individual pod.
Pick one more of these and delete the rest.
Please describe all items visible to end users below with sufficient detail so that they can verify correct enablement
and operation of this feature.
Recall that end users cannot usually observe component logs or access metrics.
-->

- [X] API .status
  - Other field: ".status.allocation" will be set for a claim using structured parameters
    when needed by a pod.

###### What are the reasonable SLOs (Service Level Objectives) for the enhancement?

<!--
This is your opportunity to define what "normal" quality of service looks like
for a feature.

It's impossible to provide comprehensive guidance, but at the very
high level (needs more precise definitions) those may be things like:
  - per-day percentage of API calls finishing with 5XX errors <= 1%
  - 99% percentile over day of absolute value from (job creation time minus expected
    job creation time) for cron job <= 10%
  - 99.9% of /health requests per day finish with 200 code

These goals will help you determine what you need to measure (SLIs) in the next
question.
-->

###### What are the SLIs (Service Level Indicators) an operator can use to determine the health of the service?

<!--
Pick one more of these and delete the rest.
-->

- [ ] Metrics
  - Metric name:
  - [Optional] Aggregation method:
  - Components exposing the metric:
- [ ] Other (treat as last resort)
  - Details:

###### Are there any missing metrics that would be useful to have to improve observability of this feature?

<!--
Describe the metrics themselves and the reasons why they weren't added (e.g., cost,
implementation difficulties, etc.).
-->

### Dependencies

<!--
This section must be completed when targeting beta to a release.
-->

###### Does this feature depend on any specific services running in the cluster?

<!--
Think about both cluster-level services (e.g. metrics-server) as well
as node-level agents (e.g. specific version of CRI). Focus on external or
optional services that are needed. For example, if this feature depends on
a cloud provider API, or upon an external software-defined storage or network
control plane.

For each of these, fill in the following—thinking about running existing user workloads
and creating new ones, as well as about cluster-level services (e.g. DNS):
  - [Dependency name]
    - Usage description:
      - Impact of its outage on the feature:
      - Impact of its degraded performance or high-error rates on the feature:
-->

### Scalability

<!--
For alpha, this section is encouraged: reviewers should consider these questions
and attempt to answer them.

For beta, this section is required: reviewers must answer these questions.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.
-->

###### Will enabling / using this feature result in any new API calls?

<!--
Describe them, providing:
  - API call type (e.g. PATCH pods)
  - estimated throughput
  - originating component(s) (e.g. Kubelet, Feature-X-controller)
Focusing mostly on:
  - components listing and/or watching resources they didn't before
  - API calls that may be triggered by changes of some Kubernetes resources
    (e.g. update of object X triggers new updates of object Y)
  - periodic API calls to reconcile state (e.g. periodic fetching state,
    heartbeats, leader election, etc.)
-->

###### Will enabling / using this feature result in introducing new API types?

<!--
Describe them, providing:
  - API type
  - Supported number of objects per cluster
  - Supported number of objects per namespace (for namespace-scoped objects)
-->

###### Will enabling / using this feature result in any new calls to the cloud provider?

<!--
Describe them, providing:
  - Which API(s):
  - Estimated increase:
-->

###### Will enabling / using this feature result in increasing size or count of the existing API objects?

<!--
Describe them, providing:
  - API type(s):
  - Estimated increase in size: (e.g., new annotation of size 32B)
  - Estimated amount of new objects: (e.g., new Object X for every existing Pod)
-->

###### Will enabling / using this feature result in increasing time taken by any operations covered by existing SLIs/SLOs?

<!--
Look at the [existing SLIs/SLOs].

Think about adding additional work or introducing new steps in between
(e.g. need to do X to start a container), etc. Please describe the details.

[existing SLIs/SLOs]: https://git.k8s.io/community/sig-scalability/slos/slos.md#kubernetes-slisslos
-->

###### Will enabling / using this feature result in non-negligible increase of resource usage (CPU, RAM, disk, IO, ...) in any components?

<!--
Things to keep in mind include: additional in-memory state, additional
non-trivial computations, excessive access to disks (including increased log
volume), significant amount of data sent and/or received over network, etc.
This through this both in small and large cases, again with respect to the
[supported limits].

[supported limits]: https://git.k8s.io/community//sig-scalability/configs-and-limits/thresholds.md
-->

###### Can enabling / using this feature result in resource exhaustion of some node resources (PIDs, sockets, inodes, etc.)?

<!--
Focus not just on happy cases, but primarily on more pathological cases
(e.g. probes taking a minute instead of milliseconds, failed pods consuming resources, etc.).
If any of the resources can be exhausted, how this is mitigated with the existing limits
(e.g. pods per node) or new limits added by this KEP?

Are there any tests that were run/should be run to understand performance characteristics better
and validate the declared limits?
-->

### Troubleshooting

<!--
This section must be completed when targeting beta to a release.

For GA, this section is required: approvers should be able to confirm the
previous answers based on experience in the field.

The Troubleshooting section currently serves the `Playbook` role. We may consider
splitting it into a dedicated `Playbook` document (potentially with some monitoring
details). For now, we leave it here.
-->

###### How does this feature react if the API server and/or etcd is unavailable?

###### What are other known failure modes?

<!--
For each of them, fill in the following information by copying the below template:
  - [Failure mode brief description]
    - Detection: How can it be detected via metrics? Stated another way:
      how can an operator troubleshoot without logging into a master or worker node?
    - Mitigations: What can be done to stop the bleeding, especially for already
      running user workloads?
    - Diagnostics: What are the useful log messages and their required logging
      levels that could help debug the issue?
      Not required until feature graduated to beta.
    - Testing: Are there any tests for failure mode? If not, describe why.
-->

###### What steps should be taken if SLOs are not being met to determine the problem?

## Implementation History

<!--
Major milestones in the lifecycle of a KEP should be tracked in this section.
Major milestones might include:
- the `Summary` and `Motivation` sections being merged, signaling SIG acceptance
- the `Proposal` section being merged, signaling agreement on a proposed design
- the date implementation started
- the first Kubernetes release where an initial version of the KEP was available
- the version of Kubernetes where the KEP graduated to general availability
- when the KEP was retired or superseded
-->

- Kubernetes 1.30: Code merged as extension of v1alpha2
- Kubernetes 1.31: v1alpha3 with new API and several new features

## Drawbacks

DRA driver developers have to give up some flexibility with regards to
parameters compared to opaque parameters in KEP #3063.
They have to learn and understand how structured models
work to pick something which fits their needs.

## Alternatives

### Publishing resource information in node status

This is not desirable for several reasons (most important one first):
- All data from all drivers must be in a single object which is already
  large. It might become too large, with no chance of mitigating that by
  splitting up the information.
- All watchers of node objects get the information even if they don't need it.
- It puts complex alpha quality fields into a GA API.

### Injecting vendor logic into CA

With this KEP, vendor's use resource tracking and simulation that gets
implemented in core Kubernetes. Alternatively, CA could support vendor logic in
several different ways:

- Call out to a vendor server via some RPC mechanism (similar to scheduler
  webhooks). The risk here is that simulation becomes to slow. Configuration
  and security would be more complex.

- Load code provided by a vendor as [Web Assembly
  (WASM)](https://webassembly.org/) at runtime and invoke it similar to the
  builtin controllers in this KEP.  WASM is currently too experimental and has
  several drawbacks (single-threaded, all data must be
  serialized). https://github.com/kubernetes-sigs/kube-scheduler-wasm-extension
  is currently exploring usage of WASM for writing scheduler plugins. If this
  becomes feasible, then implementing a builtin controller which delegates its
  logic to vendor WASM code will be possible.

- Require that vendors provide Go code with their custom logic and rebuild CA
  with that code included. The scheduler could continue to use
  PodSchedulingContext, as long as the custom logic exactly matches what the
  DRA driver controller does. This approach is not an option when a pre-built
  CA binary has to be used and leads to challenges around maintenance and
  support of such a rebuilt CA binary. However, technically it [becomes
  possible](https://github.com/kubernetes-sigs/kube-scheduler-wasm-extension)
  with this KEP.

### ResourceClaimTemplate

Instead of creating a ResourceClaim from a template, the
PodStatus could be extended to hold the same information as a
ResourceClaimStatus. Every component which works with that information
then needs permission and extra code to work with PodStatus. Creating
an extra object seems simpler.

### Reusing volume support as-is

ResourceClaims are similar to PersistentVolumeClaims and also a lot of
the associated logic is similar. An [early
prototype](https://github.com/intel/proof-of-concept-cdi) used a
custom CSI driver to manage resources.

The user experience with that approach is poor because per-resource
parameters must be stored in annotations of a PVC due to the lack of
custom per-PVC parameters. Passing annotations as additional parameters was [proposed
before](https://github.com/kubernetes-csi/external-provisioner/issues/86)
but were essentially [rejected by
SIG-Storage](https://github.com/kubernetes-csi/external-provisioner/issues/86#issuecomment-465836185)
because allowing apps to set custom parameters would make apps
non-portable.

The current volume support also has open issues that affect the
“volume as resource” approach: Multiple different Pods on a node are
allowed to use the same
volume. https://github.com/kubernetes/enhancements/pull/2489 will
address that, but is still work in progress.  Recovery from a bad node
selection during delayed binding may get stuck when a Pod has multiple
volumes because volumes are not getting deleted after a partial
provisioning. A proposal to fix that needs further work
(https://github.com/kubernetes/enhancements/pull/1703).  Each “fake”
CSI driver would have to implement and install a scheduler extender
because storage capacity tracking only considers volume size as
criteria for selecting nodes, which is not applicable for custom
resources.

### Extend volume support

The StorageClass and PersistentVolumeClaim structs could be extended
to allow custom parameters. Together with an extension of the CSI
standard that would address the main objection against the previous
alternative.

However, SIG-Storage and the CSI community would have to agree to this
kind of reuse and accept that some of the code maintained by them
becomes more complex because of these new use cases.

### Extend Device Plugins

The device plugins API could be extended to implement some of the
requirements mentioned in the “Motivation” section of this
document. There were certain attempts to do it, for example an attempt
to [add ‘Deallocate’ API call](https://github.com/kubernetes/enhancements/pull/1949) and [pass pod annotations to 'Allocate' API call](https://github.com/kubernetes/kubernetes/pull/61775)

However, most of the requirements couldn’t be satisfied using this
approach as they would require major incompatible changes in the
Device Plugins API. For example: partial and optional resource
allocation couldn’t be done without changing the way resources are
currently declared on the Pod and Device Plugin level.

Extending the device plugins API to use [Container Device Interface](https://github.com/container-orchestrated-devices/container-device-interface)
would help address some of the requirements, but not all of them.

NodePrepareResource and NodeUnprepareResource could be added to the Device Plugins API and only get called for
resource claims.

However, this would mean that
developers of the device plugins would have to implement mandatory
API calls (ListAndWatch, Allocate), which could create confusion
as those calls are meaningless for the Dynamic Resource Allocation
purposes.

Even worse, existing device plugins would have to implement the new
calls with stubs that return errors because the generated Go interface
will require them.

It should be also taken into account that device plugins API is
beta. Introducing incompatible changes to it may not be accepted by
the Kubernetes community.

### ResourceDriver

Similar to CSIDriver for storage, a separate object describing a DRA
driver might be useful at some point. At the moment it is not needed yet and
therefore not part of the v1alpha2 API. If it becomes necessary to describe
optional features of a DRA driver, such a ResourceDriver type might look
like this:

```
type ResourceDriver struct {
    // The name of the object is the unique driver name.
    ObjectMeta

    // Features contains a list of features supported by the driver.
    // New features may be added over time and must be ignored
    // by code that does not know about them.
    Features []ResourceDriverFeature
}

type ResourceDriverFeature struct {
    // Name is one of the pre-defined names for a feature.
    Name ResourceDriverFeatureName
    // Parameters might provide additional information about how
    // the driver supports the feature. Boolean features have
    // no parameters, merely listing them indicates support.
    Parameters runtime.RawExtension
}
```

## Infrastructure Needed

Initially, all development will happen inside the main Kubernetes
repository. The mock driver can be developed inside test/e2e/dra.  For the
generic part of that driver, i.e. the code that other drivers can reuse, and
other common code a new staging repo `k8s.io/dynamic-resource-allocation` is
needed.
