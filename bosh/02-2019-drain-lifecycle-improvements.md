# Drain Life Cycle Improvements

## Background

Currently BOSH exposes a drain life cycle hook that occurs before an instance is
stopped. This script gives release authors an unbounded amount of time to
perform any actions that are necessary for a graceful shutdown of their
software. Currently the drain interface is as follows:

* Release authors can optionally have an executable associated with their job
  that lives at `/var/vcap/jobs/<job-name>/bin/drain`.
* Before stopping the associated process, BOSH will execute the drain script if
  it is present.
* The drain script will be executed with the following arguments:
  * Job Change: The first argument is a string argument that indicates the
    status of the job during the deploy. The two valid values are:
    * `job_new`: This value indicates that the job has never been previously
      deployed. **Note**: How does this ever work if the drain script is
      present?
    * `job_changed`: This is the default value for the first argument and it
      covers every other call to the drain script.
  * Hash Change: The second argument is a string argument that indicates the
    status of the configuration hash between deploys. The valid values are:
    * `hash_new`: This value indicates that the previously deployed job did not
      contain a configuration hash. **Note**: How does this ever work if the
      drain script is present?
    * `hash_unchanged`: This value indicates that the previously deployed job's
      configuration matches the desired job's configuration.
    * `hash_changed`: This value indicates that the desired job's configuration
      contains changes that are not present in the previously deployed
      configuration.
  * Updated Packages: The following variadic argument is the list of associated
    packages that are changing as a result of the following instance update.
    Valid values of this argument are the package names associated with the
    deployed job.
* The drain script will be executed with the following environment variables
  available:
  * `PATH`: The default path for the drain script is always `/usr/sbin:/usr/bin:/sbin:/bin`.
  * `BOSH_JOB_STATE`: This environment contains the JSON representation of the
    jobs previously applied state.
  * `BOSH_JOB_NEXT_STATE`: This environment contains the JSON representation of
    the jobs desired next state. The most common use case for this environment
    variable is to check the value of the `persistent_disk` to detect when the
    virtual machine / instance is being deleted.
* The release author must implement the drain script so that it behaves in the
  following ways:
  * The script must exit with a `0` exit code on success.
  * The script must exit with a non-zero exit code on failure.
  * The only output that may be outputted to standard output is an integer.
    * If the integer is non-negative it will trigger static draining in which
      BOSH will wait `N` seconds before continuing.
    * If the integer is negative it will trigger dynamic draining in which BOSH
      will wait `N` seconds before recalling the drain script.
  * **Notes**: There is an existing bug in the drain script execution in which
    non-zero exit codes can be ignored. See the [following
    issue](https://github.com/cloudfoundry/bosh/issues/1896).

Please consult the [documentation](https://bosh.io/docs/drain) for further clarification on the drain
script interface.

## Problem Statement

BOSH release authors often have to manage complex clustered software in which a
graceful shutdown is dependent on the process in which BOSH updates an instance.
Currently BOSH does not expose enough information to the release author to make
these draining decisions in a stable and ideal manner. Some examples of
information that is currently lacking from the interface:

* Whether or not the instance is being removed
* Whether or not the virtual machine is being recreated
* Current and desired states of other instances in the instance group
* Current and desired states of other co-located jobs on the instance
* Complete current and desired state of the job being drained

Some examples of where this information is lacking and release authors have
worked around these limitations or implemented inefficient draining logic:

* [kubelet drain
  script](https://github.com/cloudfoundry-incubator/kubo-release/blob/master/jobs/kubelet/templates/bin/drain.erb):
  The kubelet drain script is inefficient in that it always drains the kubelet
  node and removes the kubelet for the cluster when the instance is updated.
  This involves complex logic around unmounting attached persistent volumes and
  ensuring that the kubelet is in a safe state before continuing. Realistically
  the kubelet should only drain when the node is being recreated or removed. On
  simple updates, the kubelet should be able to be restarted without draining
  with no impact on the running workload. The necessary information to make
  these decisions is not present at this time.
* [cfcr-etcd-release drain
  script](https://github.com/cloudfoundry-incubator/cfcr-etcd-release/blob/master/jobs/etcd/templates/bin/drain.erb):
  When draining an ETCD node, the node must be removed manually from the cluster
  if it is being removed. Currently the only way to detect this is by checking
  the value of the `persistent_disk` key in the `JOB_NEXT_STATE` environment
  variable. If the value is `0` then we know that the persistent disk is being
  removed and thus the node can be successfully removed from the cluster. This
  is a very brittle way to detect that a node is being removed from a cluster.
  This strategy only works if a node has a persistent disk, and does not cover
  instances that only rely on ephemeral data.

You can find more user research on issues with the current drain life cycle in
the following [Google drive
folder](https://drive.google.com/drive/folders/1N5vB_vFGn8akUbbi8UG1YEnl2Z25R1ej).

## Proposed Workflow

In order to expose more information to the release author, we should provide a
JSON structured environment variable that exposes more information about the
deployment process so that release authors can write more efficient and
effective drain scripts.

**TODO**: Rename the following environment variable.

Release authors should have access to a `BOSH_SOMETHING` that contains data in
the following format:

```
{
  "version": <integer>,
  "instances_updates": {
    "vm": {
      "recreated": true
    },
    "packages": [
      {"name": "golang", "changed": true},
      {"name": "bpm", "changed": false}
    ]
    ...
  }
  ...
}
```

**Note**: The structure of this data is a suggestion, we should decide how to
structure this data in the most effective format to encapsulate all of this
data.

**Note**: The structure of this data includes the version that it was generated
as. This allows operators to build robust scripts that can react to the changes
in the structure of this data.

### Backwards Compatibility

One of the complex issues associated with this change is that the current drain
action implemented by the bosh-agent does not expose enough information in it's
interface to provide the data detected above.

For this change, we should implement a new drain action with a more expressive
interface so that we can pass more information to the bosh-agent so that it can
populate the data correctly.

This new drain action will still need to provide the older arguments and
`BOSH_JOB_STATE` and `BOSH_JOB_NEXT_STATE` environment variables for some period
of time to allow BOSH release authors the ability to transition their drain
scripts to the new interface.

The BOSH director will need to attempt to perform the new drain action. If the
agent responds with an error due to the action not being implemented, the BOSH
director will need to perform the old drain action.

Older agents will still provide the previous interface that has been defined in
the background section of this document.

Release authors will need to write their scripts to branch off of the presence
of the new environment variable to take advantage of this new information. If
the new environment variable is not present, the drain scripts will have to rely
on the previous arguments and environment variables provided by older agents.

## Open Questions

* What should the `BOSH_SOMETHING` environment variable be named?
* What should the structure of the `BOSH_SOMETHING` environment variable be?
  What data should be included? We should plan a generic structure so that
  future information can be added without breaking the interface. Consider using
  arrays instead of maps for structured data.
* Are there any missing concerns with the backwards compatibility section above?
* When and how do we deprecate and remove the old format of the drain script? Do
  we need to?
* How do we successfully version the `BOSH_SOMETHING` structure?
* Should we provide a shared assets for release authors to take advantage of to
  parse and deal with the structure JSON data?
