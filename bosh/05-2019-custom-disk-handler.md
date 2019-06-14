# Custom Disk Provisioning

## Background

Currently BOSH allows users to request disks to be attached to instances for
both ephemeral and persistent disks.

### Persistent Disks

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

Once a persistent disk is requested for an instance, the agent handles the disk
as follows:

*

### Ephemeral Disks

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

## Problem Statement


