# Deploy CF to K8s

The purpose of this document is to take an exploratory look at what it would
take to deploy [cf-deployment](https://github.com/cloudfoundry/cf-deployment) to
Kubernetes. Currently cf-deployment is deployed and orchestrated on virtual
machines by [BOSH](https://bosh.io).

The approach that this document will take is to analyze each instance group and
the software that is managed by it and imagine a way to deploy this software to
Kubernetes. For the purpose of simplicity, we will assume that the packaging
format used for each of these pieces of software will be docker images that can
be deployed by Kubernetes. Complex lifecycle management will be in scope for
this document, but I am considering container image building and base image
layers out of scope for this exploration.

## Directory Structure

`./`
`./mysql/deployment.yaml`
`./mysql/mysql-pv.yaml`
`./mysql/secrets.yaml`
`./namespace.yml`
`./nats/00-prereqs.yaml`
`./nats/10-deployment.yaml`
`./nats/clients-auth.json`
`./nats/cluster.yaml`

## Creating a CF Namespace

`namespace.yml`:

```
apiVersion: v1
kind: Namespace
metadata:
  name: cf
    labels:
        name: cf
```

`kubectl create -f namespace.yml`

## Cloud Foundry Components

### NATS

cf-deployment deploys a simple NATS cluster with 2 instances that is accessible
using basic auth with the "nats" user.

NATS defines a [NATS operator](https://github.com/nats-io/nats-operator) that
can be used to deploy a NATS cluster to Kubernetes.

Steps:
* `kubectl apply -f nats/00-prereqs.yaml`
* `kubectl apply -f nats/10-deployment.yaml`
* `kubectl create -n cf secret generic nats-clients-auth --from-file=nats/clients-auth.json`
* `kubectl create -n cf -f nats/cluster.yaml`

Notes:
* For the purpose of deploying CF, we can rely on the "namespace-scoped"
  operator to manage NatsCluster resources in the CF namespace.
* Change `namespace: default` to `namespace: cf` in `00-prereqs.yaml` and
  `10-deployment.yaml`
* The NATS operator does allow you to create more complex NATS clusters with
  [mutual auth TLS enabled](https://github.com/nats-io/nats-operator#tls-support).
  These secrets are either manually created or managed by [Cert Manager](https://github.com/jetstack/cert-manager)
* The NATS cluster deployment must be in the same namespace as the CRD.
* The nats-operator is also available through a [Helm chart](https://github.com/nats-io/nats-operator/tree/master/helm/nats-operator)
* Schema for configuring the cluster can be found
  [here](https://github.com/nats-io/nats-operator/blob/7ecd1fc69f0afcc65ea183db1d7bd0a99e7242d5/pkg/apis/nats/v1alpha2/cluster.go#L82-L140).
* How do I set [sysctl properties](https://kubernetes.io/docs/tasks/configure-pod-container/security-context/)
  for pods created by this CRD?

### MySQL

cf-deployment deploys an internal MySQL database with 1 instance that is
accessible as a persistence layer for co-located web servers. Ideally this
deployment can be replace with a multi instance highly available service that is
resilient to downtime.

The Kubernetes documentation provides an [example mysql deployment](https://kubernetes.io/docs/tasks/run-application/run-single-instance-stateful-application/)
using a stateful set and persistent volume.

Steps:
* `kubectl apply -n cf -f mysql/secrets.yaml`
* `kubectl apply -n cf -f mysql/mysql-pv.yaml`
* `kubectl apply -n cf -f mysql/deployment.yaml`

Notes:
* In order to configure the mysql instance, one needs to provide a custom mysql
  configuration file and mount it into the container.
* The [mysql docker image](https://hub.docker.com/_/mysql) has more information
  about how to configure the instance.

### UAA

cf-deployment deploys UAA as an identity service.

## Miscellaneous Notes

* `kubectl get all` to list all resources
  * When listing resources, it will not show all namespaces by default
  * Use `kubectl get all -n cf` or `kubectl get all --all-namespaces` to show
    resources

## Tools for Deploying to K8s

* helm
* kapp
* kf
* ReplicaSets, Deployments, CRDs
