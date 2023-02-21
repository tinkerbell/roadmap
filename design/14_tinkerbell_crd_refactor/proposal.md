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

The new set of CRDs will be defined as part of a `v1alpha2` API.

#### `Hardware`

```go
// Hardware is a logical representation of a machine that can execute Workflows.
type Hardware struct {
	HardwareSpec
}

type HardwareSpec struct {
	// NetworkInterfaces defines the desired DHCP and netboot configuration for a network interface.
	// It is necessary to specify at least one NetworkInterface.
	NetworkInterfaces NetworkInterfaces

	// IPXE provides iPXE script override fields.
	// Optional.
	IPXE IPXE

	// OSIE describes the Operating System Installation Environment to be netbooted.
	OSIE OSIE

	// Instance describes instance specific data that is generally unused by Tinkerbell core.
	Instance Instance

	// StorageDevices is a list of storage devices that will be available in the OSIE.
	// Optional.
	StorageDevices []StorageDevice

	// BMCRef references a Rufio Machine object. It exists in the current API and will not be changed
	// with this proposal.
	BMCRef LocalObjectReference
}

// NetworkInterface is the desired configuration for a particular network interface.
type NetworkInterface struct {
	// DHCP is the basic network information for serving DHCP requests.
	DHCP DHCP

	// DisableDHCP disables DHCP for this interface. Implies DisableNetboot.
	// Default false.
	DisableDHCP bool

	// DisableNetboot disables netbooting for this interface. The interface will still receive
	// network information speified on by DHCP.
	// Default false.
	DisableNetboot bool
}

// DHCP describes basic network configuration to be served in DHCP offers.
type DHCP struct {
	IP      string
	Netmask string

	// Optional.
	Gateway string

	// Optional.
	Hostname string

	// Optional.
	VLANID int

	// Optional.
	Nameservers []string

	// Optional.
	Timeservers []string

	// Defaults to max allowed DHCP lease time.
	LeaseTime int32
}

// OSIE describes an OSIE to be used with a Hardware. The environment data
// is dependent on the OSIE being used and should be updated with the OSIE reference object.
type Netboot struct {
	// OSIERef is a reference to an OSIE object.
	OSIERef LocalObjectReference

	// KernelParams passed to the kernel when launching the OSIE. Parameters are joined with a
	// space.
	// Optional.
	KernelParams []KernelParam
}

type IPXE struct {
	// Inline is an inline iPXE script that will be served as specified on this property.
	Inline *string

	// URL is a URL to an hosted iPXE script.
	URL *string
}

// Instance describes instance specific data. Instance specific data is typically dependent on the
// permanent OS that a piece of hardware runs. This data is often served by an instance metadata
// service such as Tinkerbell's Hegel. The core Tinkerbell stack does not leverage this data.
type Instance struct {
	// Userdata is data with a structure understood by the producer and consumer of the data.
	Userdata string

	// Vendordata is data with a structure understood by the producer and consumer of the data.
	Vendordata string
}

// NetworkInterfaces maps a MAC address to a NetworkInterface.
type NetworkInterfaces map[MAC]NetworkInterface

// MAC is a Media Access Control address.
type MAC string

// StorageDevice describes a storage device path that will be present in the OSIE.
type StorageDevice string

// KernelParam defines an atomic kernel parameter that will be passed to the OSIE.
type KernelParam string
```

#### `OSIE`

`OSIE` is a new CRD. It exists to ensure OSIE URLs can be re-used and easily updated across `Hardware` instances.

```go
// OSIE describes and Operating System Initialization Environment. It is used by Tinkerbell
// to provision machines and should launch the Tink Worker component.
type OSIE struct {
	Spec OSIESpec
}

type OSIESpec struct {
	// KernelURL is a URL to a kernel image.
	KernelURL string

	// InitrdURL is a URL to an initrd image.
	InitrdURL string
}
```

#### `Template`

```go
// Template defines a set of actions to be run on a target machine. The template is rendered
// prior to execution where it is exposed to Hardware and user defined data. All fields within
// TemplateSpec may contain template values. See https://pkg.go.dev/text/template for more details.
type Template struct {
	Spec TemplateSpec
}

type TemplateSpec struct {
	// Actions defines the set of actions to be run on a target machine. Actions are run sequentially
	// in the order they are specified. At least 1 action must be specified. Names of actions
	// must be unique within a Template.
	Actions []Action

	// Volumes to be mounted on all actions. If an action specifies the same volume it will take
	// precedence.
	// Optional.
	Volumes []Volume

	// Env defines environment variables to be available in all actions. If an action specifies
	// the same environment variable it will take precedence.
	// Optional.
	Env map[string]string
}

// Action defines an individual action to be run on a target machine.
type Action struct {
	// Name is a unique name for the action.
	Name ActionName

	// Image is an OCI image.
	Image string

	// Command defines the command to use when launching the image.
	// Optional.
	Command string

	// Args are a set of arguments to be passed to the container on launch.
	Args []string

	// Env defines environment variables used when launching the container.
	// Optional.
	Env map[string]string

		// Volumes defines the volumes to mount into the container.
	// Optional.
	Volumes []Volume

	// NetworkNamespace defines the network namespace to run the container in. This enables access 
	// to the host network namespace.
	// See https://man7.org/linux/man-pages/man7/namespaces.7.html.
	NetworkNamespace string
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

// ActionName is unique name within the context of a workflow.
type ActionName string
```

#### `Workflow`

```go
// Workflow describes a set of actions to be run on a specific Hardware. Workflows execute
// once and should be considered ephemeral.
type Workflow struct {
	Spec   WorkflowSpec
	Status WorkflowStatus
}

type WorkflowSpec struct {
	// HardwareRef is a reference to a Hardware resource this workflow will execute on.
	// If no namespace is specified the Workflow's namespace is assumed.
	HardwareRef LocalObjectReference

	// TemplateRef is a reference to a Template resource used to render workflow actions.
	// If no namespace is specified the Workflow's namespace is assumed.
	TemplateRef LocalObjectReference

	// TemplateData is user defined data that is injected during template rendering. The data
	// structure should be marshalable.
	// Optional.
	TemplateData map[string]any

	// Timeout defines the time the workflow has to complete. The timer begins when the first action
	// is requested. When set to 0, no timeout is applied.
	// Default 0.
	Timeout int32
}

type WorkflowStatus struct {
	// Actions is the list of rendered actions and their status.
	Actions RenderedActions

	// StartedAt is the time the first action was requested. Nil indicates the workflow has not
	// started.
	StartedAt *metav1.Time

	// State describes the current state of the workflow. This fields represents a summation of
	// action states. Specifically, if all actions succeeded, the workflow will succeed. If one
	// action fails, the workflow fails irrespective of previous action status'.
	State State

	// Reason describes the reason for failure. It is only relevant when Result is ResultFailed.
	// It is propogated from the failed action.
	Reason Reason
}

// RenderedActions is a map of action name to RenderedAction.
type RenderedActions map[ActionName]ActionStatus

// ActionStatus describes status information about an action.
type ActionStatus struct {
	// Rendered is the rendered action.
	Rendered Action

	// StartedAt is the time the action was requested. Nil indicates the action has not started.
	StartedAt *metav1.Time

	// State describes the current state of the action.
	State State

	// Reason describes the reason for failure. It is only relevant when Result is ResultFailed.
	Reason Reason

	// Message is a freeform user friendly message describing the specific issue that caused the
	// failure. It is only relevant when Result is ResultFailed.
	Message string
}

// State describes the point in time state of a workflow or action.
type State string

const (
	StatePending   State = "Pending"
	StateRunning   State = "Running"
	StateSucceeded State = "Succeeded"
	StateFailed    State = "Failed"
)

// Reason is a one-word TitleCase string indicating why a failure occurred. It is not restricted
// to the values defined with this API.
type Reason string

const (
	ReasonUnknown Reason = "Unknown"
	ReasonTimeout Reason = "Timeout"
)
```

### Workflow state transition

![Workflow state transitions](https://raw.githubusercontent.com/tinkerbell/roadmap/b7b2362997c01ffa52758e10259b8f55e81f7447/design/14_tinkerbell_crd_refactor/workflow_state_machine.png)

Actions follow a similar state transition model.

### Webhooks

We will introduce a set of basic webhooks for validating each CRD. The webhooks will only validate the `v1alpha2` API version. They will be implemented as part of the `tink-controller` component.

### Data and functions available during template rendering

Templates will have access to a subset of `Hardware` data when they are rendered. Injecting `Hardware` data was outlined in a [previous proposal](https://github.com/tinkerbell/proposals/tree/e24b19a628c6b1ecaafd566667155ca5d6fd6f70/proposals/0028). The existing implementation only injects disk that will, based on this proposal, be sources from the `.Hardware.StorageDevices` list.

The previous proposal did not outline a set of custom functions injected into templates. The custom functions are detailed in [`template_funcs.go`](https://github.com/tinkerbell/tink/blob/main/internal/workflow/template_funcs.go) and include:

* `contains`: determine if a string contains a substring.
* `hasPrefix`: determine if a string has a prefix.
* `hasSuffix`: determine if a string has a suffix.
* `formatPartition`: formats a string representing a device path with a specific partition. For example, `{{ formatPartition "/dev/sda" 1 }}` will result in `/dev/sda1`.

These functions will continue to be available during template rendering.

### Hegel Changes

Hegel will undergo a reduction in the endpoints it serves because the data is no longer available in the `Hardware` resource. Minimally, Hegel will serve the following endpoints:

* `/2009-04-04/meta-data/instance-id`
* `/2009-04-04/user-data`

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

## Implementation Plan

1. All services that have unreleased changes will be released under a new version. This provides a baseline of functional services that can be used in the Tinkerbell Helm charts.

2. Develop the code to leverage the `v1alpha2` API.

3. Hard cut over to the new API version. This means versions beyond those specified during (1) will no longer support the `v1alpha1` API.

## Future Work

Introducing mechanisms to propagate reasons and user readable messages for action failure. As these mechanisms do not exist currently and will not be realized through this proposal, the `.Workflow.Status.Reason` and `.Workflow.Status.Message` will have minimal benefit to the end user. The separate proposal will address the contract between actions and Tink Worker that can be used to propogate a reason and message to workflows.

Maintainers have informally discussed inverting the relationship between `Hardware` and Rufio CRDs. This is necessary for defining a precedent on how to extend Tinkerbell functionality without expanding the scope of core Tinkerbell CRDs.

Introduction of user defined metadata to be served by Hegel that could facilitate user defined actions. Similarly, injection of additional `Hardware` data when templates are rendered will be addressed on a separate ad-hoc basis.

Separation of `Hardware.Instance` data into a separate CRD possibly owned by Hegel. Given the instance data is unused by the core Tinkerbell stack it 
