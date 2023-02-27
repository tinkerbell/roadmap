# Tink CRD Refactor

## Table of contents

- [Tink CRD Refactor](#tink-crd-refactor)
  - [Table of contents](#table-of-contents)
  - [Overview](#overview)
  - [Context](#context)
  - [Goals/Non-goals](#goalsnon-goals)
  - [Proposal](#proposal)
    - [Custom Resource Definitions](#custom-resource-definitions)
      - [`Hardware`](#hardware)
      - [`OSIE`](#osie)
      - [`Template`](#template)
      - [`Workflow`](#workflow)
    - [Workflow state transition](#workflow-state-transition)
    - [Webhooks](#webhooks)
    - [Data and functions available during template rendering](#data-and-functions-available-during-template-rendering)
    - [Hegel Changes](#hegel-changes)
  - [Migrating from v1alpha1 to v1alpha2](#migrating-from-v1alpha1-to-v1alpha2)
  - [Rationale](#rationale)
    - [Comparison with existing resources](#comparison-with-existing-resources)
    - [Use of explicit `State`, `Reason` and `Message` fields vs conditions](#use-of-explicit-state-reason-and-message-fields-vs-conditions)
  - [Implementation Plan](#implementation-plan)
  - [Future Work](#future-work)

## Overview

Tinkerbell's backend is rooted in 3 Custom Resource Definitions (CRDs): Hardware, Workflow and Template. The CRDs were developed as part of the KRM proposal that introduced a Kubernetes backend option to the Tinkerbell stack and mirrored the Postgres database schema (now deprecated and removed). As the CRDs were a reflection of the Postgres schema, they inherited the schemas flaws. This proposal attempts to remediate the flaws by refactoring the CRDs.

## Context

When users interact with Tinkerbell the primary interface is `kubectl` and Tinkerbell's CRDs: Hardware, Workflow and Template. The CRDs are hard to understand because they contain duplicate fields, obsolete fields, unclear semantics and, consequently, unclear system behavior expectations.

Some specific issues with the CRDs are summarized as:

**Hardware**

1. Network information is specified in both `.Spec.Interfaces` and `.Spec.Metadata.Instance.Ips`. Only `.Spec.Interfaces` is used.
1. Disk information is specified in both `.Spec.Disks` and `.Spec.Metadata.Instance.Storage`. `.Spec.Metadata.Instance.Storage` is unused.
1. Userdata can be specified in both `.Spec.UserData` and `.Spec.Instance.Userdata`. `.Spec.Instance.Userdata` is unused.
1. `.Spec.TinkVersion` has no functional use.
1. `.Spec.Resources` was intended for use in CAPT as part of its Hardware selection algorithm but is yet to be implemented.
1. `.Spec.Interfaces[].Netboot.{AllowPXE,AllowWorkflow}` and `.Spec.Metadata.Instance.AlwaysPxe` are seemingly related but reside on different objects and how they impact eachother is unclear.
1. `.Spec.Metadata.Custom` defines specific fields that are related to other parts of Hardware.
1. `.Status.State`, `.Spec.Metadata.Instance.State` and `.Spec.Metadata.State` have unclear semantics. The `.Spec` state fields impact the machine provisioning process but nothing in the core Tinkerbell stack sets their values. `.Status.State` is unused.

**Template**

1. Template defines a single field, `.Spec.Data`. The format of `.Spec.Data` is entirely ambiguous requiring the user to understand implementation detail. In summary, `.Spec.Data` is composed of a list of tasks that can run on different machines; a task is composed of a list of actions that perform a function such as streaming a raw image. The multi-machine capability has no known use-cases.
1. The Template objects `.Status` field is unused.

**Workflow**

1. The `.Spec.HardwareMap` historically defines a template value used to render a tasks `WorkerAddr`. The `WorkerAddr` should be the MAC of the machine that should run the task. This creates a hard to understand relationship between Workflow and Template that users must understand to successfully execute Workflows.
1. The `.Spec.GlobalTimeout` is unused and its origin is unclear (status fields are typically populated by Kubernetes controllers to build understanding of the current object state).
1. Actions leverage the `WorkflowState` type that is intended to describe the overall state of the Workflow.

Users resort to Q&A in the Tinkerbell Slack to determine what fields are required and how tweaking them impacts the system. The CRDs should be simple enough and sufficiently documented to aid users in understanding how they can manipulate the system.

## Goals/Non-goals

**Goals**

- To de-duplicate Tink custom resource definition fields and data structures.
- To provide clear behavioral expectations when manipulating custom resources.
- To remove obsolete fields and data structures.

**Non-goals**

- To change the existing relationship between Tink and Rufio.
- To introduce additional object status data typically found on the `.Status` field.
- To support new technologies such as IPv6.

## Proposal

### Custom Resource Definitions

The CRDs will be developed under a `v1alpha2` API version.

#### `Hardware`

```go
// Hardware is a logical representation of a machine that can execute Workflows.
type Hardware struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	HardwareSpec `json:"spec,omitempty"`
}

type HardwareSpec struct {
	// NetworkInterfaces defines the desired DHCP and netboot configuration for a network interface.
	//+kubebuilder:validation:MinItems=1
	NetworkInterfaces NetworkInterfaces `json:"networkInterfaces,omitempty"`

	// IPXE provides iPXE script override fields. This is useful for debugging or netboot
	// customization.
	//+optional.
	IPXE IPXE `json:"ipxe,omitempty"`

	// OSIE describes the Operating System Installation Environment to be netbooted.
	OSIE OSIE `json:"osie,omitempty"`

	// Instance describes instance specific data that is generally unused by Tinkerbell core.
	//+optional
	Instance Instance `json:"instance,omitempty"`

	// StorageDevices is a list of storage devices that will be available in the OSIE.
	//+optional.
	StorageDevices []StorageDevice `json:"storageDevices,omitempty"`

	// BMCRef references a Rufio Machine object. It exists in the current API and will not be changed
	// with this proposal.
	//+optional.
	BMCRef LocalObjectReference `json:"bmcRef,omitempty"`
}

// NetworkInterface is the desired configuration for a particular network interface.
type NetworkInterface struct {
	// DHCP is the basic network information for serving DHCP requests.
	DHCP DHCP `json:"dhcp,omitempty"`

	// DisableDHCP disables DHCP for this interface. Implies DisableNetboot.
	//+kubebuilder:default=false
	DisableDHCP bool `json:"disableDhcp,omitempty"`

	// DisableNetboot disables netbooting for this interface. The interface will still receive
	// network information speified on by DHCP.
	//+kubebuilder:default=false
	DisableNetboot bool `json:"disableNetboot,omitempty"`
}

// DHCP describes basic network configuration to be served in DHCP offers.
type DHCP struct {
	// IP is an IPv4 address.
	//+kubebuilder:validation:Pattern="(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)(\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)){3}"
	IP      string `json:"ip,omitempty"`

	// Netmask is an IPv4 netmask.
	//+kubebuilder+validation:Pattern="^(255)\.(0|128|192|224|240|248|252|254|255)\.(0|128|192|224|240|248|252|254|255)\.(0|128|192|224|240|248|252|254|255)"
	Netmask string `json:"netmask,omitempty"`

	// Gateway is the default gateway address.
	//+kubebuilder:validation:Pattern="(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)(\.(25[0-5]|2[0-4][0-9]|[01]?[0-9][0-9]?)){3}"
	//+optional
	Gateway string `json:"gateway,omitempty"`

	//+kubebuilder:validation:Pattern="^(([a-zA-Z0-9]|[a-zA-Z0-9][a-zA-Z0-9\-]*[a-zA-Z0-9])\.)*([A-Za-z0-9]|[A-Za-z0-9]"[A-Za-z0-9\-]*[A-Za-z0-9])$"
	//+optional
	Hostname string `json:"hostname,omitempty"`

	// VLANID is a VLAN ID between 0 and 4096.
	//+kubebuilder:validation:Pattern="^(([0-9][0-9]{0,2}|[1-3][0-9][0-9][0-9]|40([0-8][0-9]|9[0-6]))(,[1-9][0-9]{0,2}|[1-3][0-9][0-9][0-9]|40([0-8][0-9]|9[0-6]))*)$"
	//+optional
	VLANID string `json:"vlanId,omitempty"`

	//+optional
	Nameservers []Nameserver `json:"nameservers,omitempty"`

	//+optional
	Timeservers []Timeserver `json:"timeservers,omitempty"`

	// 24h default
	//+kubebuilder:default=86400
	//+kubebuilder:validation:Minimum=0
	LeaseTime int32 `json:"leaseTime"`
}

// OSIE describes an OSIE to be used with a Hardware. The environment data
// is dependent on the OSIE being used and should be updated with the OSIE reference object.
type Netboot struct {
	// OSIERef is a reference to an OSIE object.
	OSIERef LocalObjectReference `json:"osieRef,omitempty"`

	// KernelParams passed to the kernel when launching the OSIE. Parameters are joined with a
	// space.
	//+optional
	KernelParams []string `json:"kernelParams,omitempty"`
}

// IPXE describes overrides for IPXE scripts. At least 1 option must be specified.
type IPXE struct {
	// Content is an inline iPXE script.
	//+optional
	Content *string `json:"inline,omitempty"`

	// URL is a URL to a hosted iPXE script.
	//+optional
	URL *string `json:"url,omitempty"`
}

// Instance describes instance specific data. Instance specific data is typically dependent on the
// permanent OS that a piece of hardware runs. This data is often served by an instance metadata
// service such as Tinkerbell's Hegel. The core Tinkerbell stack does not leverage this data.
type Instance struct {
	// Userdata is data with a structure understood by the producer and consumer of the data.
	//+optional
	Userdata string `json:"userdata,omitempty"`

	// Vendordata is data with a structure understood by the producer and consumer of the data.
	//+optional
	Vendordata string `json:"vendordata,omitempty"`
}

// NetworkInterfaces maps a MAC address to a NetworkInterface.
type NetworkInterfaces map[MAC]NetworkInterface

// MAC is a Media Access Control address.
//+kubebuilder:validation:Pattern="^([0-9A-Fa-f]{2}[:-]){5}([0-9A-Fa-f]{2})$"
type MAC string

// Nameserver is an IP or hostname.
//+kubebuilder:validation:Pattern="^(([a-zA-Z0-9]|[a-zA-Z0-9][a-zA-Z0-9\-]*[a-zA-Z0-9])\.)*([A-Za-z0-9]|[A-Za-z0-9][A-Za-z0-9\-]*[A-Za-z0-9])$|^(([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])\.){3}([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])$"
type Nameserver string

// Timeserver is an IP or hostname.
//+kubebuilder:validation:Pattern="^(([a-zA-Z0-9]|[a-zA-Z0-9][a-zA-Z0-9\-]*[a-zA-Z0-9])\.)*([A-Za-z0-9]|[A-Za-z0-9][A-Za-z0-9\-]*[A-Za-z0-9])$|^(([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])\.){3}([0-9]|[1-9][0-9]|1[0-9]{2}|2[0-4][0-9]|25[0-5])$"
type Timeserver string

// StorageDevice describes a storage device path that will be present in the OSIE.
// StorageDevices must be valid linux paths.
//+kubebuilder:validation:Pattern="^(/[^/ ]*)+/?$"
type StorageDevice string
```

#### `OSIE`

`OSIE` is a new CRD. It enables re-use of OSIE URLs across `Hardware` instances.

```go
// OSIE describes an Operating System Installation Environment. It is used by Tinkerbell
// to provision machines and should launch the Tink Worker component.
type OSIE struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec OSIESpec `json:"spec,omitempty"`
}

type OSIESpec struct {
	// KernelURL is a URL to a kernel image.
	KernelURL string `json:"kernelUrl,omitempty"`

	// InitrdURL is a URL to an initrd image.
	InitrdURL string `json:"initrdUrl,omitempty"`
}
```

#### `Template`

```go
// Template defines a set of actions to be run on a target machine. The template is rendered
// prior to execution where it is exposed to Hardware and user defined data. All fields within
// TemplateSpec may contain template values. See https://pkg.go.dev/text/template for more details.
type Template struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec TemplateSpec `json:"spec,omitempty"`
}

type TemplateSpec struct {
	// Actions defines the set of actions to be run on a target machine. Actions are run sequentially
	// in the order they are specified. At least 1 action must be specified. Names of actions
	// must be unique within a Template.
	//kubebuilder:validation:MinItems=1
	Actions []Action `json:"actions,omitempty"`

	// Volumes to be mounted on all actions. If an action specifies the same volume it will take
	// precedence.
	//+optional
	Volumes []Volume `json:"volumes,omitempty"`

	// Env defines environment variables to be available in all actions. If an action specifies
	// the same environment variable it will take precedence.
	//+optional
	Env map[string]string `json:"env,omitempty"`
}

// Action defines an individual action to be run on a target machine.
type Action struct {
	// Name is a unique name for the action.
	Name string `json:"name,omitempty"`

	// Image is an OCI image.
	Image string `json:"image,omitempty"`

	// Cmd defines the command to use when launching the image.
	//+optional
	Cmd string `json:"cmd,omitempty"`

	// Args are a set of arguments to be passed to the container on launch.
	//+optional
	Args []string `json:"args,omitempty"`

	// Env defines environment variables used when launching the container.
	//+optional
	Env map[string]string `json:"env,omitempty"`

	// Volumes defines the volumes to mount into the container.
	//+optional
	Volumes []Volume `json:"volumes,omitempty"`

	// NetworkNamespace defines the network namespace to run the container in. This enables access 
	// to the host network namespace.
	// See https://man7.org/linux/man-pages/man7/namespaces.7.html.
	//+optional
	NetworkNamespace string `json:"networkNamespace,omitempty"`
}

// Volume is a specification for mounting a volume in an action. Volumes take the form
// {SRC-VOLUME-NAME | SRC-HOST-DIR}:TGT-CONTAINER-DIR:OPTIONS. When specifying a VOLUME-NAME that 
// does not exist it will be created for you.
//
// Examples
//
// Read-only bind mount bound to /data
//   /etc/data:/data:ro
//
// Writable volume name bound to /data
//   shared_volume:/data
//
// See https://docs.docker.com/storage/volumes/ for additional details
type Volume string

```

#### `Workflow`

```go
// Workflow describes a set of actions to be run on a specific Hardware. Workflows execute
// once and should be considered ephemeral.
type Workflow struct {
	metav1.TypeMeta   `json:",inline"`
	metav1.ObjectMeta `json:"metadata,omitempty"`

	Spec   WorkflowSpec `json:"spec,omitempty"`
	Status WorkflowStatus `json:"status,omitempty"`
}

type WorkflowSpec struct {
	// HardwareRef is a reference to a Hardware resource this workflow will execute on.
	// If no namespace is specified the Workflow's namespace is assumed.
	HardwareRef LocalObjectReference `json:"hardwareRef,omitempty"`

	// TemplateRef is a reference to a Template resource used to render workflow actions.
	// If no namespace is specified the Workflow's namespace is assumed.
	TemplateRef LocalObjectReference `json:"templateRef,omitempty"`

	// TemplateData is user defined data that is injected during template rendering. The data
	// structure should be marshalable.
	//+optional
	TemplateData map[string]any `json:"templateData,omitempty"`

	// Timeout defines the time the workflow has to complete. The timer begins when the first action
	// is requested. When set to 0, no timeout is applied.
	//+kubebuilder:default=0
	//+kubebuilder:validation:Minimum=0
	Timeout time.Duration `json:"timeout,omitempty"`
}

type WorkflowStatus struct {
	// Actions is the map of rendered actions and their status'.
	Actions map[string]ActionStatus `json:"actions,omitempty"`

	// StartedAt is the time the first action was requested. Nil indicates the workflow has not
	// started.
	StartedAt *metav1.Time `json:"startedAt,omitempty"`

	// State describes the current state of the workflow. For the workflow to enter the 
	// WorkflowStateSucceeded state all actions must be in ActionStateSucceeded. The Workflow will
	// enter a WorkflowStateFailed if 1 or more Actions fails.
	State WorkflowState `json:"state,omitempty"`

	// Reason is a short CamelCase word or phrase describing why the Workflow entered
	// WorkflowStateFailed. It is not relevant when the State field is not WorkflowStateFailed.
	Reason string `json:"reason,omitempty"`

	// Message is a free-form user friendly message describing why the Workflow entered the
	// WorkflowStateFailed state. Typically, this is an elaboration on the Reason. It is not
	// relevant when the State field is not WorkflowStateFailed.
	Message string `json:"message,omitempty"`
}

// ActionStatus describes status information about an action.
type ActionStatus struct {
	// Rendered is the rendered action.
	Rendered Action `json:"rendered,omitempty"`

	// StartedAt is the time the action was requested. Nil indicates the action has not started.
	StartedAt *metav1.Time `json:"startedAt,omitempty"`

	// State describes the current state of the action.
	State ActionState `json:"state,omitempty"`

	// Reason is a short CamelCase word or phrase describing why the Action entered
	// ActionStateFailed. It is not relevant when the State field is not ActionStateFailed. 
	Reason string `json:"reason,omitempty"`

	// Message is a free-form user friendly message describing why the Action entered the
	// ActionStateFailed state. Typically, this is an elaboration on the Reason. It is not
	// relevant when the State field is not ActionStateFailed.
	Message string `json:"message,omitempty"`
}

// State describes the point in time state of a Workflow.
type WorkflowState string

const (
	WorkflowStatePending   WorkflowState = "Pending"
	WorkflowStateRunning   WorkflowState = "Running"
	WorkflowStateSucceeded WorkflowState = "Succeeded"
	WorkflowStateFailed    WorkflowState = "Failed"
)

// ActionState describes a point in time state of an Action.
type ActionState string

const (
	ActionStatePending      ActionState = "Pending"
	ActionStateRunning      ActionState = "Running"
	ActionStateSucceeded    ActionState = "Succeeded"
	ActionStateFailed       ActionState = "Failed"
)
```

### Workflow state machine

![Workflow state machine](https://raw.githubusercontent.com/tinkerbell/roadmap/latest/design/14_tinkerbell_crd_refactor/workflow_state_machine.png)

### Action state machine

![Action state machine](https://raw.githubusercontent.com/tinkerbell/roadmap/latest/design/14_tinkerbell_crd_refactor/workflow_state_machine.png)

### Resource validation

Each CRD will leverage [CEL](https://kubernetes.io/blog/2022/09/23/crd-validation-rules-beta/) for all validations supported by CEL. For validations such as ensuring MAC address uniqueness on `Hardware` resources a webhook will be used. The webhook will be served from the `tink-controller` component.

### Data and functions available during template rendering

Templates will have access to a subset of `Hardware` data when they are rendered. Injecting `Hardware` data was outlined in a [previous proposal](https://github.com/tinkerbell/proposals/tree/e24b19a628c6b1ecaafd566667155ca5d6fd6f70/proposals/0028). The existing implementation only injects disk that will, based on this proposal, be sourced from the `.Hardware.StorageDevices` list.

The previous proposal did not outline a set of custom functions injected into templates. The custom functions are detailed in [`template_funcs.go`](https://github.com/tinkerbell/tink/blob/main/internal/workflow/template_funcs.go) and include:

* `contains`: determine if a string contains a substring.
* `hasPrefix`: determine if a string has a prefix.
* `hasSuffix`: determine if a string has a suffix.
* `formatPartition`: formats a string representing a device path with a specific partition. For example, `{{ formatPartition "/dev/sda" 1 }}` will result in `/dev/sda1`.

These functions will continue to be available during template rendering.

### Hegel Changes

Hegel will undergo a reduction in the endpoints it serves because the data is no longer available on the `Hardware` resource. Minimally, Hegel will serve the following endpoints:

* `/2009-04-04/meta-data/instance-id`
* `/2009-04-04/user-data`

This ensures compatability with the cloud-init that explores the API via the `instance-id` endpoint.

## Migrating from v1alpha1 to v1alpha2

Given the relative immaturity of the Tinkerbell API we will not provide any migration tooling from `v1alpha1` to `v1alpha2`. Users will be required to manually convert `Hardware` and `Template` resources. `Workflow` resources are considered ephemeral and can generally be deleted.

## Rationale

### Comparison with existing resources

**Hardware**

The features available to toggle DHCP behavior have been reduced to 2 options specified on the `NetworkInterface` data structure: `DisableNetboot` and `DisableDHCP`. This is in contrast to the existing `Hardware` that defines 4 different properties across various data structures including the `State` fields that require specific, undocumented string values to alter behavior.

We rationalized that a machine can only boot a single OSIE at a time because operators do not regularly change the OSIE they want to use, and that there are weak use-cases for multi-OSIE support. Consequently, the ability to support different OSIEs per network interface has been removed in favor of supporting a single OSIE definition.

Numerous fields served by Hegel have been removed with only the fields necessary for honoring cloud-init remaining under the `Instance` field. This significantly reduces the data available for Hegel to serve and _will_ break some Tinkerbell maintained actions that leverage Hegel data to configure disks. A separate proposal will address user defined metadata and how Hegel will serve that data.

**Template**

Templates are explicitly defined as part of the CRD in contrast to the free-form string that required a specific, largely undocumented, format in the existing definition. This will signficantly improve the user experience as it establishes clear expectations and feature support.

[Tasks](https://github.com/tinkerbell/tink/blob/main/internal/workflow/types.go#L13) used for multi-worker workflows have been removed. Multi-worker workflows have been unsupported since the transition to the Kubernetes backend. This, subsequently, removes the need to specify device IDs (device IDs are specified on the `Workflow`) when rendering templates simplifying the `Workflow` and `Template` relationship.

The `PID` field has been removed in favor of a `Network` field. This facilitates known use-cases that require the container to run in the host network namespace.

`On*` fields have been removed. The `On*` fields provided rudimentary event based mechanisms for executing arbitrary logic. The semantics were unclear and use-cases unknown.

All fields on the `TemplateSpec` will support templatable values in accordance with the [`test/template` Go standard library package](https://pkg.go.dev/text/template).

**Workflow**

Tasks no longer feature on `WorkflowStatus` as they pertain to multi-worker workflows that are no longer supported.

Reasons for failures have been introduced as a core concept, via the `Reason` field, around status reporting of actions. This enables programmatic failure reason identification and comparison. A future proposal will address how users can provide customized machine comparable reasons for failed actions.

The `Workflow` provides a `TemplateData` field that can be used to inject arbitrary data into templates. This facilitates users wanting to model variable data on `Template`s that has per-`Workflow` values.

### Use of explicit `State`, `Reason` and `Message` fields vs conditions

Workflows operate as a state machine (detailed in [Workflow state transition](#workflow-state-transition)). [Conditions represent observations](https://github.com/kubernetes/community/blob/master/contributors/devel/sig-architecture/api-conventions.md#typical-status-properties) about a resource and are not state machines in and of themeselves. Conditions should generally be complimentary to existing resource information, not replace it. Conditions in the Kubernetes API have a [well defined set of fields](https://github.com/kubernetes/community/blob/4c9ef2d/contributors/devel/sig-architecture/api-conventions.md#typical-status-properties). Some tools such as [Cluster API](https://github.com/kubernetes-sigs/cluster-api/blob/main/docs/proposals/20200506-conditions.md) break away by defining their own condition expectations.

For these reasons, we think it would be more appropriate to have explicit fields to represent state and consider a separate conditions proposal that compliments the CRDs.

## Implementation Plan

1. All services that have unreleased changes will be released under a new version. This provides a baseline of functional services that can be used in the Tinkerbell Helm charts.

2. Develop the code to leverage the `v1alpha2` API.

3. Hard cut over to the new API version. This means versions beyond those specified during (1) will no longer support the `v1alpha1` API.

## Future Work

Introducing mechanisms to propagate reasons and user readable messages for action failure. As these mechanisms do not exist currently and will not be realized through this proposal, the `.Workflow.Status.Reason` and `.Workflow.Status.Message` will have minimal benefit to the end user. The separate proposal will address the contract between actions and Tink Worker that can be used to propogate a reason and message to workflows.

Maintainers have informally discussed inverting the relationship between `Hardware` and Rufio CRDs. This is necessary for defining a precedent on how to extend Tinkerbell functionality without expanding the scope of core Tinkerbell CRDs.

Introduction of user defined metadata to be served by Hegel that could facilitate user defined actions. Similarly, injection of additional `Hardware` data when templates are rendered will be addressed on a separate ad-hoc basis.

Separation of `Hardware.Instance` data into a separate CRD possibly owned by Hegel. Given the instance data is unused by the core Tinkerbell stack it 