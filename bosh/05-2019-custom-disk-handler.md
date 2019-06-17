# Custom Disk Provisioning

## Background

Currently BOSH allows users to request disks to be attached to instances for
both ephemeral and persistent disks.

### Persistent Disks

#### Configuration

Currently there are three ways for the persistent disks to be defined.

* `persistent_disk_type`: This property allows a user to request a disk type
  defined in the cloud config as the persistent disk for the instance group.
* `persistent_disk_size`: This property allows a user to request a persistent
  disk of a certain size with no custom configuration of the disk type.
* `persistent_disks`: This property allows a user to request multiple persistent
  disks with specific names and types defined in the cloud config for the
  instance group.

Disk definitions in the cloud config are as follows:

```
disk_types:
- name: default
  disk_size: 3000
  cloud_properties:
    ...
```

#### Lifecycle

Once a persistent disk is requested for an instance, the disk is handled in the
following ways:

* Ability to list persistent disks that are currently mounted
* Ability to attach a new persistent disk
  * Disk is attached through the CPI
  * Disk is added into persistent disk settings maintained by bosh-agent
* Ability to mount a new persistent disk
  * Includes mounting and partitioning disk for usage
* Ability to unmount an old persistent disk
  * Unmounts the persistent disk from the deployed virtual machine
* Ability to detach an old persistent disk
  * Disk is detached through the CPI
* Ability to migrate the contents from an old disk to a new disk
  * Assumes new disk is already mounted
  * Mounts old disk as read-only
  * Copies contents of old disk to new disk
  * Unmounts the old disk and remounts the new disk
* Ability to live resize a disk through the CPI
  * Disk partitions and filesystem are not currently managed by the agent

### Ephemeral Disks

#### Configuration

Currently ephemeral disks are defined in the `vm_types` specified in the cloud
config. For example on VSphere this configuration appears as follows:

```
vm_types:
- name: default
  cloud_properties:
    disk: 10240
```

Conversely, the same configuration on Amazon Web Services is as follows:

```
vm_types:
- name: default
  cloud_properties:
    ephemeral_disk:
      size: 10240
```

In order to configure these disks further, users will often use `vm_extensions`
to customize ephemeral disks for specific instance groups:

```
vm_extensions:
- name: 50GB_ephemeral_disk
  cloud_properties:
    disk: 50000
```

These can then be referenced in the deployment manifest as follows:

```
instance_groups:
- name: instance
  vm_extensions:
  - 50GB_ephemeral_disk
```

#### Lifecycle

When the bosh-agent performs bootstrapping, the agent handles ephemeral disks in
one of three ways:

* Raw ephemeral disks
  * Skips setup if configured to do so
  * Partitions the disk if not partitioned with the label `raw-ephemeral-n`
* Normal ephemeral disks
  * Skips setup if configured to do so
  * Partitions the ephemeral disk if necessary
  * Mounts the ephemeral disk in the correct location
* Ephemeral mount on the root disk
  * If configured to do so, the agent will partition the ephemeral disk on the
    root device

## Problem Statement

In certain situations, BOSH operators wish to be able to customize the manner in
which disks are mounted and partitioned. For example:

* Operators wish to use custom encryption software in order to encrypt the
  persistent disk at rest. This involves allowing the software to mount the
  filesystem before BOSH jobs are able to run and access the disk.
* Operators wish to mount the persistent storage as a different filesystem type.

Through the current configuration of the BOSH agent, operators are only capable
of accepting the default provisioning that the agent delivers, or completely
ignoring provisioning by the agent.

When configuring the agent to ignore the provisioning of the disk, it is up to
the operator to provide a custom job that partitions and mounts the device
correctly before the co-located jobs are able to run. At this point, BOSH is
only capable of running all co-located jobs in parallel, so there is no
lifecycle for disk provisioning jobs to be injected in order to handle disks
correctly before dependent software begins executing.

## Proposed Workflows

* [Disk Provisioners](#disk-provisioner)
* [Priority Groups](#priority-groups)

### Disk Provisioner

In order to allow disks to be provisioned by custom software, we could allow
users to define a specific provisioner for the disk that they are requesting.
This can either be done through the `disk_type` definition:

```
disk_types:
- name: encrypted-disk
  provisioner:
  - name: custom-encryption
    release: encryption-providers
  cloud_properties:
    ...
```

or the instance group definition:

```
instance_groups:
- name: instance
  instances: 2
  persistent_disk_type: default
  persistent_disk_provisioner:
  - name: custom-encryption
    release: encryption-providers
```

The benefits to defining the provisioner on the instance group is that it could
apply to any `disk_type` definition defined in the cloud config.

The benefits to defining the provisioner on the `disk_type` is that it would
reduce duplication amongst consumers of the `disk_type` definition.

#### Disk Provisioner Interface

Disk provisioner authors need to be able to deliver their provisioner to the
BOSH deployed virtual machine. In order to do this, provisioner's would be
regular BOSH jobs that implement scripts that satisfy a disks lifecycle. Ideally
this interface would be:

* `bin/mount`: A custom script to partition and mount a disk
* `bin/unmount`: A custom script to unmount a disk
* `bin/resize`: A custom script to resize the filesystem and partitions on a
  given disk

In order to consume information about a given disk, disk provisioners would need
to consume a link that contains the information about the persistent disk they
are provisioning. For example:

```
#!/bin/bash

<% disk_name = link('disk').p('name') %>
<% device_path = link('disk').p('path') %>

parted <%= device_path %> --label <%= disk_name %> ...
mount <%= device_path %> ...
```

#### Agent Interactions

When calling out to `add_persistent_disk`, the director needs to pass along a
new optional argument to identify the custom provisioner that it wants to use
for the persistent disk.

Before calling out to the `mount_disk`, `unmount_disk`, or `migrate_disk`
methods on the agent, the director would need to call `prepare` or `apply` to
ensure that the packages associated with the custom provisioner are present on
the deployed virtual machine.

When executing `mount_disk`, `unmount_disk`, or `migrate_disk` the agent will
defer to the lifecycle hooks defined by the custom provisioner in order to
handle the persistent disk.

#### Open Questions

* Should the disk be consumed through a link? Or should it be arguments passed
  to the lifecycle scripts by the agent itself?
* Should the persistent partitioner be attached to the disk type or the instance
  group?
* Can we apply this partitioning logic to the ephemeral disk? Do we want to?

### Priority Groups

Priroity groups provide an interface for operators to be able to specify a well
known priority that would result in their software starting in a certain order.
The configuration for this option would be as follows:

```
instance_groups:
- name: instance
  instances: 2
  jobs:
  - name: disk-encryption
    release: encryption-r-us
    priority: <disk|network|none>
```

For each job specified in the list of jobs, the operator can specify one of either
`disk`, `network`, or `none` for the optional `priority` property. The default value
for this property would be `none`.

When `priority` is specified the director would start jobs in the following order:

* Send rendered templates to the deployed VM
* Start (pre-start, start, post-start) all jobs in the `disk` priority group in parallel
* Start (pre-start, start, post-start) all jobs in the `network` priority group in parallel
* Start (pre-start, start, post-start) all jobs in the `none` priority group in parallel

When `priority` is specified the director would stop jobs in the following order:

* Stop (pre-stop, drain, stop, post-stop) all jobs in the `none` priority group in parallel
* Stop (pre-stop, drain, stop, post-stop) all jobs in the `network` priority group in parallel
* Stop (pre-stop, drain, stop, post-stop) all jobs in the `disk` priority group in parallel

Specifically for a job that is in the `disk` priority group, it's responsiblities would include:

* Partitioning, mounting, and preparing the disk before post-start completes
* Unmounting the disk before post-stop completes

#### Agent Interactions

In order to successfully provision disks with custom software, the operator would need to
leverage the unmanaged disk feature in order to force the agent to defer disk management
to the co-located jobs.

Therefore during `mount_disk`, `unmount_disk`, and `resize_disk` the agent would no longer
act on the disk that is being provisioned.

#### Open Questions

* How does the disk job in the priority group know which disk to provision?
  * All disks? Consume a disk link? How is this configuration specified in the manifest?
* How does an operator resize a disk in this setup?
  * `resize_disk` currently mounts both disks and copies the data from one to the other. 
     How does the co-located job hook into this action? Is resizing disks in this manner no longer supported?
  * IAAS native resizing would still require increasing the partition and FS size, 
    potentially possible with unmanaged disks.
* Which priority groups are reasonable? Should we start with only `disk`?
* What is the behavior of specifying multiple co-located jobs in the same priority group?
