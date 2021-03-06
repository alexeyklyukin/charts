# Patroni Helm Chart

This directory contains a Kubernetes chart to deploy a five node patroni cluster using a statefulset.

## Prerequisites Details
* Kubernetes 1.5
* PV support on the underlying infrastructure

## Statefulset Details
* https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/

## Statefulset Caveats
* https://kubernetes.io/docs/concepts/workloads/controllers/statefulset/#limitations

## Todo
* Make namespace configurable

## Chart Details
This chart will do the following:

* Implement a HA scalable PostgreSQL cluster using Kubernetes PetSets

## Installing the Chart

To install the chart with the release name `my-release`:

```console
$ helm repo add incubator http://storage.googleapis.com/kubernetes-charts-incubator
$ helm install --name my-release incubator/patroni
```

## Connecting to Postgres

Your access point is a cluster IP. In order to access it spin up another pod:

```console
$ kubectl run -i --tty ubuntu --image=ubuntu:16.04 --restart=Never -- bash -il
```

Then, from inside the pod, connect to postgres:

```console
$ apt-get update && apt-get install postgresql-client -y
$ psql -U admin -h my-release-patroni.default.svc.cluster.local postgres
<admin password from values.yaml>
postgres=>
```

## WAL shipping

This chart supports shipping WALs and producing/restoring from base backups. Right now only AWS and minikube environments
are supported.

For AWS, one needs to set `AWS.Kube2IAMAccountName` to the name of Kube2IAM (https://github.com/jtblin/kube2iam) role. That
role must allow `s3:ListBucket` for the target bucket (include `bucket_name/` and `bucket_name/*`in the list of resources),
as well as `s3.*` for the `s3://bucket_name/spilo/*`. It needs `ec2:CreateTags` and `ec2:Describe` in order to set the
`spilo-role` tags. The target bucket will be used for both base backups and WAL shipping.

For the minikube, set `Local.WalHostPath` to a non-empty value that should be a valid unix path. This path will be created
on the minikube node and mounted to all pods under `/archive`. WAL files will be written to `/archive/wal/<release-name>`.
Base backups are currently not supported with minikube (this is the limitation of WAL-E).

## Configuration

The following tables lists the configurable parameters of the patroni chart and their default values.


|       Parameter          |           Description                |                         Default                     |
|--------------------------|--------------------------------------|-----------------------------------------------------|
| `Name`                   | Service name                         | `patroni`                                           |
| `Spilo.Image`            | Container image name                 | `registry.opensource.zalan.do/acid/spilo-9.5`       |
| `Spilo.Version`          | Container image tag                  | `1.0-p5`                                            |
| `ImagePullPolicy`        | Container pull policy                | `IfNotPresent`                                      |
| `Replicas`               | k8s petset replicas                  | `5`                                                 |
| `Component`              | k8s selector key                     | `patroni`                                           |
| `Resources.Cpu`          | container requested cpu              | `100m`                                              |
| `Resources.Memory`       | container requested memory           | `512Mi`                                             |
| `Resources.Storage`      | Persistent volume size               | `1Gi`                                               |
| `StorageClass`           | Persistent volumes storage class     | default storage class will be used                  |
| `ServiceAccountName`     | Name of the service account to use   |                                                     |
| `Credentials.Superuser`  | password for the superuser           | `tea`                                               |
| `Credentials.Admin`      | password for the admin user          | `cola`                                              |
| `Credentials.Standby`    | password for the replication user    | `pinacolada`                                        |
| `Etcd.Host`              | host name of etcd cluster            | not used (Etcd.Discovery is used instead            |
| `Etcd.Discovery`         | domain name of etcd cluster          | `<release-name>-etcd.<namespace>.svc.cluster.local` |
| `AWS.Kube2IAMAccountName`| AWS account configured with Kube2IAM |                                                     |
| `AWS.S3BucketName`       | AWS S3 Bucket to put WAL files in    |                                                     |
| `Local.WALHostPath`      | Path on the node for the WAL archive |                                                     |

Specify each parameter using the `--set key=value[,key=value]` argument to `helm install`.

Alternatively, a YAML file that specifies the values for the parameters can be provided while installing the chart. For example,

```console
$ helm install --name my-release -f values.yaml incubator/patroni
```

> **Tip**: You can use the default [values.yaml](values.yaml)

## Cleanup

In order to remove everything you created a simple `helm delete <release-name>` isn't enough (as of now), but you can do the following:

```console
$ release=<release-name>
$ helm delete $release
$ grace=$(kubectl get po $release-patroni-0 --template '{{.spec.terminationGracePeriodSeconds}}')
$ kubectl delete statefulset,po -l release=$release
$ sleep $grace
$ kubectl delete pvc -l release=$release
```

## Internals

Patroni is responsible for electing a Postgres master pod by leveraging etcd.
It then exports this master via a Kubernetes service and a label query that filters for `spilo-role=master`.
This label is dynamically set on the pod that acts as the master and removed from all other pods.
Consequently, the service endpoint will point to the current master.

```console
$ kubectl get pods -l spilo-role -L spilo-role
NAME                   READY     STATUS    RESTARTS   AGE       SPILO-ROLE
my-release-patroni-0   1/1       Running   0          9m        replica
my-release-patroni-1   1/1       Running   0          9m        master
my-release-patroni-2   1/1       Running   0          8m        replica
my-release-patroni-3   1/1       Running   0          8m        replica
my-release-patroni-4   1/1       Running   0          8m        replica
```
