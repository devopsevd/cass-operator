# Cass Operator
[![Gitter](https://badges.gitter.im/cass-operator/community.svg)](https://gitter.im/cass-operator/community?utm_source=badge&utm_medium=badge&utm_campaign=pr-badge)

The DataStax Kubernetes Operator for Apache Cassandra&reg;

## Getting Started

Quick start:
```console
kubectl create -f https://raw.githubusercontent.com/datastax/cass-operator/26ad52bfc8f450852f5573fa2904a5df407ce2d3/docs/user/cass-operator-manifests.yaml
# *** This is for GKE -> Adjust based on your cloud or storage options
kubectl create -f https://raw.githubusercontent.com/datastax/cass-operator/26ad52bfc8f450852f5573fa2904a5df407ce2d3/operator/k8s-flavors/gke/storage.yaml
kubectl -n cass-operator create -f https://raw.githubusercontent.com/datastax/cass-operator/26ad52bfc8f450852f5573fa2904a5df407ce2d3/operator/example-cassdc-yaml/cassandra-3.11.6/example-cassdc-minimal.yaml
```

### Loading the operator

Installing the Cass Operator itself is straightforward. Apply the provided manifest as follows:

```console
kubectl apply -f https://raw.githubusercontent.com/datastax/cass-operator/26ad52bfc8f450852f5573fa2904a5df407ce2d3/docs/user/cass-operator-manifests.yaml
```

Note that since the manifest will install a [Custom Resource Definition](https://kubernetes.io/docs/concepts/extend-kubernetes/api-extension/custom-resources/), the user running the above command will need cluster-admin privileges.

This will deploy the operator, along with any requisite resources such as Role, RoleBinding, etc., to the `cass-operator` namespace. You can check to see if the operator is ready as follows:

```console
$ kubectl -n cass-operator get pods --selector name=cass-operator
NAME                             READY   STATUS    RESTARTS   AGE
cass-operator-555577b9f8-zgx6j   1/1     Running   0          25h
```

### Creating a storage class

You will need to create an appropriate storage class which will define the type of storage to use for Cassandra nodes in a cluster. For example, here is a storage class for using SSDs in GKE, which you can also find at [operator/deploy/k8s-flavors/gke/storage.yaml](operator/k8s-flavors/gke/storage.yaml):

```yaml
apiVersion: storage.k8s.io/v1
kind: StorageClass
metadata:
  name: server-storage
provisioner: kubernetes.io/gce-pd
parameters:
  type: pd-ssd
  replication-type: none
volumeBindingMode: WaitForFirstConsumer
reclaimPolicy: Delete
```

Apply the above as follows:

```
kubectl apply -f https://raw.githubusercontent.com/datastax/cass-operator/26ad52bfc8f450852f5573fa2904a5df407ce2d3/operator/k8s-flavors/gke/storage.yaml
```

### Creating a CassandraDatacenter

The following resource defines a Cassandra 3.11.6 datacenter with 3 nodes on one rack, which you can also find at [operator/example-cassdc-yaml/cassandra-3.11.6/example-cassdc-minimal.yaml](operator/example-cassdc-yaml/cassandra-3.11.6/example-cassdc-minimal.yaml):

```yaml
apiVersion: cassandra.datastax.com/v1beta1
kind: CassandraDatacenter
metadata:
  name: dc1
spec:
  clusterName: cluster1
  serverType: cassandra
  serverVersion: 3.11.6
  managementApiAuth:
    insecure: {}
  size: 3
  storageConfig:
    cassandraDataVolumeClaimSpec:
      storageClassName: server-storage
      accessModes:
      - ReadWriteOnce
      resources:
        requests:
          storage: 5Gi
  config:
    cassandra-yaml:
      authenticator: org.apache.cassandra.auth.PasswordAuthenticator
      authorizer: org.apache.cassandra.auth.CassandraAuthorizer
      role_manager: org.apache.cassandra.auth.CassandraRoleManager
    jvm-options:
      initial_heap_size: 800M
      max_heap_size: 800M
```

Apply the above as follows:

```console
kubectl -n cass-operator apply -f https://raw.githubusercontent.com/datastax/cass-operator/26ad52bfc8f450852f5573fa2904a5df407ce2d3/operator/example-cassdc-yaml/cassandra-3.11.6/example-cassdc-minimal.yaml
```

You can check the status of pods in the Cassandra cluster as follows:

```console
$ kubectl -n cass-operator get pods --selector cassandra.datastax.com/cluster=cluster1
NAME                         READY   STATUS    RESTARTS   AGE
cluster1-dc1-default-sts-0   2/2     Running   0          26h
cluster1-dc1-default-sts-1   2/2     Running   0          26h
cluster1-dc1-default-sts-2   2/2     Running   0          26h
```

You can check to see the current progress of bringing the Cassandra datacenter online by checking the `cassandraOperatorProgress` field of the `CassandraDatacenter`'s `status` sub-resource as follows:

```console
$ kubectl -n cass-operator get cassdc/dc1 -o "jsonpath={.status.cassandraOperatorProgress}"
Ready
```

(`cassdc` and `cassdcs` are supported short forms of `CassandraDatacenter`.)

A value of "Ready", as above, means the operator has finished setting up the Cassandra datacenter.

You can also check the Cassandra cluster status using `nodetool` by invoking it on one of the pods in the Cluster as follows:

```console
$ kubectl -n cass-operator exec -it -c cassandra cluster1-dc1-default-sts-0 -- nodetool status
Datacenter: dc1
===============
Status=Up/Down
|/ State=Normal/Leaving/Joining/Moving/Stopped
--  Address         Load       Tokens       Owns (effective)  Host ID                               Rack
UN  10.233.105.125  224.82 KiB  1            65.4%             5e29b4c9-aa69-4d53-97f9-a3e26115e625  r1
UN  10.233.92.96    186.48 KiB  1            61.6%             b119eae5-2ff4-4b06-b20b-c492474e59a6  r1
UN  10.233.90.54    205.1 KiB   1            73.1%             0a96e814-dcf6-48b9-a2ca-663686c8a495  r1
```

### (Optional) Loading the operator via Helm

Helm may be used to load the operator.  The destination namespace must be created first.

```console
kubectl create namespace my-custom-namespace
helm install --namespace=my-custom-namespace cass-operator ./charts/cass-operator-chart
```

The following Helm default values may be overridden:

```yaml
serviceAccountName: cass-operator
clusterRoleName: cass-operator-cluster-role
clusterRoleBindingName: cass-operator
roleName: cass-operator
roleBindingName: cass-operator
deploymentName: cass-operator
deploymentReplicas: 1
image: "datastax/cass-operator:1.0.0"
imagePullPolicy: IfNotPresent
```

NOTE: Helm does not install a storage-class for the cassandra pods.

## Features

- Proper token ring initialization, with only one node bootstrapping at a time
- Seed node management - one per rack, or three per datacenter, whichever is more
- Server configuration integrated into the CassandraDatacenter CRD
- Rolling reboot nodes by changing the CRD
- Store data in a rack-safe way - one replica per cloud AZ
- Scale up racks evenly with new nodes
- Replace dead/unrecoverable nodes
- Multi DC clusters (limited to one Kubernetes namespace)

All features are documented in the [User Documentation](docs/user/README.md).

### Containers

The operator is comprised of the following container images working in concert:
* The operator, built from sources in the [operator](operator/) directory.
* The config builder init container, built from sources in [datastax/cass-config-builder](https://github.com/datastax/cass-config-builder).
* Cassandra, built from
  [datastax/management-api-for-apache-cassandra](https://github.com/datastax/management-api-for-apache-cassandra),
  with Cassandra 3.11.6 support, and experimental support for Cassandra
  4.0.0-alpha3.
* ... or DSE, built from [datastax/docker-images](https://github.com/datastax/docker-images).

## Requirements

- Kubernetes cluster, 1.12 or newer.
- Users who want to use a Kubernetes version from before 1.15 can use a manifest that supports x-preserve-unknown-fields on the CassandraDatacenter CRD - [manifest](docs/user/cass-operator-manifests-pre-1.15.yaml)

## Contributing

As of version 1.0, Cass Operator is maintained by a team at DataStax and it is
part of what powers [DataStax
Astra](https://www.datastax.com/cloud/datastax-astra). We would love for open
source users to contribute bug reports, documentation updates, tests, and
features.

### Developer setup

Almost every build, test, or development task requires the following
pre-requisites...

* Golang 1.13
* Docker, either the docker.io packages on Ubuntu, Docker Desktop for Mac,
  or your preferred docker distribution.
* [mage](https://magefile.org/): There are some tips for using mage in
  [docs/developer/mage.md](docs/developer/mage.md)

### Building

The operator uses [mage](https://magefile.org/) for its build process.

#### Build the Operator Container Image
This build task will create the operator container image, building or rebuilding
the binary from golang sources if necessary:

``` bash
mage operator:buildDocker
```

#### Build the Operator Binary
If you wish to perform ONLY to the golang build or rebuild, without creating
a container image:

``` bash
mage operator:buildGo
```

### Testing

``` bash
mage operator:testGo
```

#### End-to-end Automated Testing

Run fully automated end-to-end tests...

```bash
mage integ:run
```

Docs about testing are [here](tests/README.md). These work against any k8s
cluster with six or more worker nodes.

#### Manual Local Testing
There are a number of ways to run the operator, see the following docs for
more information:
* [kind](docs/developer/kind.md): Kubernetes in Docker is the recommended
  Kubernetes distribution for use by software engineers working on the operator.
  KIND can simulate a k8s cluster with multiple worker nodes on a single
  physical machine, though it's necessary to dial down the database memory
  requests.

The [user documentation](docs/user/README.md) also contains information on
spinning up your first operator instance that is useful regardless of what
Kubernetes distribution you're using to do so.


## Not (Yet) Supported Features

- Cassandra:
  - Integrated data repair solution
  - Integrated backup and restore solution
- DSE:
  - Advanced Workloads, like Search / Graph / Analytics

## Uninstall

*This will destroy all of your data!*

Delete your CassandraDatacenters first, otherwise Kubernetes will block deletion because we use a finalizer.
```
kubectl delete cassdcs --all-namespaces
```

Remove the operator Deployment, CRD, etc.
```
kubectl delete -f https://raw.githubusercontent.com/datastax/cass-operator/26ad52bfc8f450852f5573fa2904a5df407ce2d3/docs/user/cass-operator-manifests.yaml
```

## Contacts

For development questions, please reach out on [Gitter](https://gitter.im/cass-operator/community), or by opening an issue on GitHub.

For usage questions, please visit our Community Forums: https://community.datastax.com

## License

Copyright DataStax, Inc.

Licensed under the Apache License, Version 2.0 (the "License"); you may not use this file except in compliance with the License. You may obtain a copy of the License at

http://www.apache.org/licenses/LICENSE-2.0

Unless required by applicable law or agreed to in writing, software distributed under the License is distributed on an "AS IS" BASIS, WITHOUT WARRANTIES OR CONDITIONS OF ANY KIND, either express or implied. See the License for the specific language governing permissions and limitations under the License.
