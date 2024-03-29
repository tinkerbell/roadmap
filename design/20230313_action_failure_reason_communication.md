# Action Failure Reason Communication

## Context

Tink Worker is the client launched by an Operating System Installation Environment (OSIE) that communicates with Tink Server to retrieve actions to run on the node. Actions are OCI images that Tink Worker can launch using a container runtime such as Docker. In the event an action fails, Tinkerbell does not provide mechanisms for the user defined action to communicate why it failed. This makes debugging `Workflow`s difficult as users leveraging `kubectl` cannot observe a specific failure reason and must resort to running commands directly on the node.

As part of the Tink CRD Refactor proposal we introduced `Reason` and `Message` fields to indiciate why an action entered a failure state. The proposal does not detail how these fields are populated but envisages at minimum the `Reason` being used to communicate timeout failures. 

* A `Reason` is a machine readable CamelCase word or phrase that succinctly describes the failure reason. 
* A `Message` is a human readable string that elaborates on the failure reason to provide specifics.

This proposal lays out a contract for action containers to communicate why it exited with a non-zero exit code.

## Goals/Non-goals

**Goals**

- Define a contract for action images to communicate failure information that is exposed via associated custom resource definitions and is therefore inspectable with `kubectl`.

## Proposal

Actions communicate a `Reason` by writing it to `/tinkerbell/failure-reason`. The reason must follow the same formatting expectations as defined in the CRD refactor proposal: most importantly, it must not contain spaces or new lines. If the reason does not follow the convention we will report `InvalidActionReason` in-place of the reported reason.

Actions communicate a failure message by writing to `/tinkerbell/failure-message`. It must not contain new lines.

When an action exits with a non-zero exit code, Tink Worker will arrange to read the reason and message provided by the action image and transmit them, with the action result, to Tink Server. Tink Server will update the action state, reason and message. Providing the reason and message on the action status will ensure the controller populates the `Succeeded` condition as detailed in the Tink CRD Refactor proposal.

![Reason propagation](https://raw.githubusercontent.com/tinkerbell/roadmap/7e4e769305edf5c5679a406ebf0564eb754fe57a/design/images/tink_worker_failure_reasons/reason_propagation.png)

The reason and message files will be mounted with `0666` permissions granting read write access to everyone. This ensures images launched with a different UID will still be able to write a reason and message.

The implementation behind the reason and message files will be transparent to the action maintainer. For example, the file may be backed by unix domain sockets or a host text file.