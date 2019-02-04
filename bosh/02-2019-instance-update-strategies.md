# Virtual Machine Update Strategies

## Background

Currently BOSH exposes two main strategies for updating the virtual machine that
backs an instances within an instance group.

### Delete Create

When using the `delete-create` update strategy, BOSH will perform a serial set
of actions when updating an instance that needs its virtual machine to be
recreated. Specifically BOSH will:

  1. Drain the instance
  1. Stop the instance
  1. Delete the VM associated with the instance
  1. Create the VM associated with the instance
  1. Update artifacts associated with the instance
  1. Perform `pre-start` on the instance
  1. Start the instance
  1. Perform `post-start` on the instance

Instances will be updated in parallel depending on the value of the `canaries`
property and the `max_in_flight` property.

### Create Swap Delete

When using the `create-swap-delete` update strategy, BOSH will attempt to
optimize the creation of deletion of virtual machines when possible.
Specifically BOSH will attempt to perform the following when updating an
instance:

  1. Create a new VM to be associated with the instance
  1. Update artifacts associated with the instance
  1. Drain the existing instance
  1. Stop the existing instance
  1. Orphan the existing VM to be bulk deleted by a background process
  1. Perform `pre-start` on the new instance
  1. Start the new instance
  1. Perform `post-start` on the new instance

Instances will be updated in parallel depending on the value of the `canaries`
property and the `max_in_flight` property. All new virtual machines that are
necessary for the deploy will be created in parallel at the beginning of the
deploy before instances are updated. This means that before instances are
updated, an operator will have `2*N` virtual machines created in the IAAS where
`N` is the number of instances that are being recreated using the
`create-swap-delete` strategy. Virtual machines that are orphaned will be
deleted in parallel when a background task gets executed. The background task is
scheduled to run every 5 minutes.

## Problem Statement

BOSH operators have expressed discontent at the time that it takes to update
deployments with a large number of virtual machines. Ideally, operators would
want to be able to take advantage of the parallelized creation and deletion
capabilities of the `create-swap-delete` virtual machine update strategy.
However, operators would need to be able to capacity plan for deployments that
are twice as large as what their steady state allows for. This is due to the
fact that there are no limitations on the number of virtual machines that BOSH
pre-creates when performing `create-swap-delete`.

## Proposed Workflow

In order to provide a more cohesive virtual machine update strategy, operators
should be able to specify the number of virtual machines that BOSH is able to
exceed the desired amount for any given instance group or deployment. This can
be done by exposing a new parameter for configuring the update of virtual
machines known as `surge`. Consider the following configuration in a deployment
manifest:

```
name: deployment

instance_groups:
- name: job-1
  update:
    vm:
      surge: 0
- name: job-2

update:
  vm:
    surge: 10
```

By setting a `update.vm.surge` value greater than 0, the user is requesting that
BOSH use the `create-swap-delete` update strategy for eligible virtual machines,
but it limits the number of virtual machines that BOSH is able to exceed
the desired amount. This new `update.vm.surge` property would provide a
replacement for the `update.vm_strategy`. BOSH would now decide whether or not
to use the `create-swap-delete` strategy based off of the available `vm.surge`
for any instance group.

If an operator wants to specifically disable the surging characteristics of
`create-swap-delete`, the operator can provide an `update.vm.surge` value of 0
as seen in the first instance group. In this case, BOSH will forcible perform
the serial `delete-create` behavior.
