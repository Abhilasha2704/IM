# scylla-db
This repository is left as a reference for the customizations necessary to deploy a secure production-ready ScyllaDB cluster.  Starting with version 4, ScyllaDB has provided a level 5 Kubernetes operator to manage the deployment, and this documentation is now intended as a guide to deploy a cluster using the new approach.  

By default, an insecure cluster is deployed without client / server or internode encryption in transit, an authenticator, or an authorizor.  However, the operator surfaces configuration elements to allow all of these items to be configured.

Before we actually talk about ScyllaDB, deploying ScyllaDB is usually one of the first steps of the deployment, which probably means that you haven't yet created certificates to secure your data in transit, so let's do that first.  The steps look pretty intense, but they're rather straight-forwared if followed exactly:  [Certificate Authority Instructions](README_CA.md)


## Create Secrets

If you followed the [Certificate Authority Instructions](README_CA.md) exactly, you'll end up with:
1. A CA key:  `private/ca.key.pem`
1. A CA cert:  `certs/ca.cert.pem`
1. An Intermediate CA key:  `intermediate/private/intermediate.key.pem`
1. An Intermediate CA cert:  `intermediate/certs/intermediate.cert.pem`
1. A CA certificate chain: `intermediate/certs/ca-chain.cert.pem`
1. A self-signed key:  `intermediate/private/scylla-im.key.pem`
1. A self-signed cert:  `intermediate/certs/scylla-im.cert.pem`

The CA key should never be shared, but the rest are needed to perform client / server and internode encryption in transit.  Create K8s secrets from these certificates for the ScyllaDB operator to eventually consume:

```
kubectl create namespace scylla
kubectl -n scylla create secret tls scylla-certs --cert=intermediate/certs/scylla-im.cert.pem --key=intermediate/private/scylla-im.key.pem 
kubectl -n scylla create secret tls intermediate-ca --cert=intermediate/certs/ca-chain.cert.pem --key=intermediate/private/intermediate.key.pem
```


## Setup Monitoring

Setup monitoring by performing the steps at [https://operator.docs.scylladb.com/master/monitoring.html](https://operator.docs.scylladb.com/master/monitoring.html) while skipping section 3 to install the Scylla service monitor.  The Scylla service monitor is created by the operator if `serviceMonitor.create` is set to true in `values.cluster.yaml` below (Scylla Manager service monitor is not).


## Customize Operator

The ScyllaDB operator can be customized to align with your current (secure) deployments or IBM Security best-practices in general.  The basic examples are found in the [scylla-operator](https://github.com/scylladb/scylla-operator/tree/master/examples/helm) repository, which will be used as the foundation.  I've customized the yaml files for a typical ScyllaDB installation on IBM Cloud, which are found at [deployment.tgz](https://github.ibm.com/cloud-operations/scylla-db/raw/master/examples/scylla-db/deployment.tgz).  Download and then extract the archive via `tar -zxvf deployment.tgz`.  The only files that will need to be customized according to your requirements are:

 ### examples/helm/values.operator.yaml

Customize:

1. `serviceAccount.create`: Used to create service account for scylla installation. For openshift set it to `false`.

### examples/helm/values.cluster.yaml

Customize:
 1. `scyllaImage.tag`:  Use the latest stable from [ScylladB Docker Hub](https://hub.docker.com/r/scylladb/scylla/tags?page=1&ordering=last_updated)
 1. `nameOverride`:  Used to customize the name of the actual deployment for running multiple ScyllaDB clusters on the same K8s cluster
 1. `serviceAccount.create`: Used to create service account for scylla installation. For openshift set it to `false`.
 1. `datacenter`:  Set your IBM Cloud data center
 1. `racks.name`:  Set the individual IBM Cloud AZs
 1. `racks.members`:  Set the number of nodes to deploy in each AZ
 1. `racks.storage.capacity`:  Set the amount of storage to be provisioned on each ScyllaDB node.  You probably want at least `100Gi` for a Production cluster
 1. `racks.storage.storageClassName`:  Set the [tier of storage](https://cloud.ibm.com/docs/vpc?topic=vpc-block-storage-profiles#tiers) to provision in IBM Cloud.  View available options on you cluster via `kubectl get storageclass`
 1. `racks.resources.limits.cpu`:  Set the max amount of CPU for each ScyllaDB node
 1. `racks.resources.limits.memory`:  Set the max amount of memory for each ScyllaDB node.  You want at least `24Gi` for a production cluster
 1. `racks.resources.requests.cpu`:  Set the initial amount of CPU for each ScyllaDB node
 1. `racks.resources.requests.memory`:  Set the initial amount of memory for each ScyllaDB node.  You want at least `10Gi` for a production cluster

 ### examples/helm/scylla.yaml

Customize:

1. `cluster_name`:  Set the name of the cluster to provision.  It doesn't necessarily need to match `nameOverride` from `values.cluster.yaml`, but it should for consistency

## Pre-requisite
For openshift, please refer to https://github.ibm.com/cloud-operations/scylla-db/blob/master/README_Openshift.md#pre-requisite-for-deployment

## Deploy

Create the ConfigMap with our customizations for the ScyllaDB operator to merge in during the deployment:
```
kubectl -n scylla create configmap scylla-config --from-file=examples/helm/scylla.yaml
```

If you choose to deply with node affinity, add [custom storage classes](https://cloud.ibm.com/docs/openshift?topic=openshift-vpc-block#vpc-customize-default) so each rack provisions storage in the correct zone:
```
kubectl apply -f ibmc-vpc-block-10iops-tier-us-south-1
kubectl apply -f ibmc-vpc-block-10iops-tier-us-south-2
```

Perform the actual deployments:

```
helm repo add scylla https://scylla-operator-charts.storage.googleapis.com/stable
helm repo update
kubectl apply -f examples/common/cert-manager.yaml
kubectl wait -n cert-manager --for=condition=ready pod -l app=cert-manager --timeout=60s
helm install scylla-operator scylla/scylla-operator --values examples/helm/values.operator.yaml --create-namespace --namespace scylla-operator
kubectl wait -n scylla-operator --for=condition=ready pod -l app.kubernetes.io/name=scylla-operator --timeout=240s
helm install scylla scylla/scylla --values examples/helm/values.cluster.yaml --create-namespace --namespace scylla
helm install scylla-manager scylla/scylla-manager --values examples/helm/values.manager.yaml --create-namespace --namespace scylla-manager
kubectl -n scylla-operator get all
kubectl -n scylla-manager get all
kubectl -n scylla get all
```

Once all nodes are deployed, hop onto the first node and check things out via:
```
kubectl -n scylla exec -it scylla-im-scylla-us-south-us-south-1-0 bash
export SSL_CERTFILE=/etc/intermediate-ca/ca-chain.cert.pem
cqlsh --username=cassandra --password=cassandra --ssl
```


## References

 * [OpenSSL Instructions](https://jamielinux.com/docs/openssl-certificate-authority/introduction.html)
 * [Deploying Scylla stack using Helm Charts](https://operator.docs.scylladb.com/master/helm.html)
 * [Deploying Scylla on a Kubernetes Cluster](https://operator.docs.scylladb.com/master/generic.html)
 * [Scylla Cluster CRD](https://operator.docs.scylladb.com/master/scylla_cluster_crd.html)
 * [Cert Manager Installation](https://cert-manager.io/docs/installation/kubernetes/)
 * [ScyllaDB Operator GitHub Repo](https://github.com/scylladb/scylla-operator)
 * [Kubernetes Secrets](https://kubernetes.io/docs/concepts/configuration/secret/)
 * [Kubernetes Volumes](https://kubernetes.io/docs/concepts/storage/volumes/)
 * [How to Use Kubernetes Secrets](https://newrelic.com/blog/how-to-relic/how-to-use-kubernetes-secrets)
 * [ScylladB Docker Hub](https://hub.docker.com/r/scylladb/scylla/tags?page=1&ordering=last_updated)
 * [Enable ScyllaDB Authentication](https://docs.scylladb.com/operating-scylla/security/authentication/)
 * [Operator Capability Levels](https://sdk.operatorframework.io/docs/advanced-topics/operator-capabilities/operator-capabilities/)
 * [Kubed](https://appscode.com/products/kubed/v0.12.0/guides/config-syncer/intra-cluster/)
 * [scylla-driver Python driver](https://python-driver.docs.scylladb.com/stable/installation.html)
 * [ScyllaDB API documentation](https://python-driver.docs.scylladb.com/stable/api/cassandra/cluster.html#cassandra.cluster.Session.execute)

