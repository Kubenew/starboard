# Getting Started

## Before you Begin

You need to have a Kubernetes cluster, and the kubectl command-line tool must be configured to communicate with your
cluster. If you do not already have a cluster, you can create one by installing [minikube] or [kind], or you can use one
of these Kubernetes playgrounds:

* [Katacode](https://www.katacoda.com/courses/kubernetes/playground)
* [Play with Kubernetes](http://labs.play-with-k8s.com/)

You also need the Starboard Operator to be installed in the `starboard-operator` namespace, e.g. with
[static YAML manifests](./installation/kubectl.md).

## Workloads Scanning

Assuming that you installed the operator in the `starboard-operator` namespace, and it's configured to discover
Kubernetes workloads in the `default` namespace, let's create the `nginx` Deployment that we know is vulnerable:

```
kubectl create deployment nginx --image nginx:1.16
```

When the first ReplicaSet controlled by the `nginx` Deployment is created, the operator immediately detects that and
creates the Kubernetes Job in the `starboard-operator` namespace to scan the `nginx:1.16` image for vulnerabilities.
It also creates the Job to audit the Deployment's configuration for common pitfalls such as running the `nginx`
container as root:

```console
$ kubectl get job -n starboard-operator
NAME                                 COMPLETIONS   DURATION   AGE
scan-configauditreport-c4956cb9d     0/1           1s         1s
scan-vulnerabilityreport-c4956cb9d   0/1           1s         1s
```

If everything goes fine, the scan Jobs are deleted, and the operator saves scan reports as custom resources in the
`default` namespace, named after the Deployment's active ReplicaSet. For image vulnerability scans, the operator creates
a VulnerabilityReport for each different container defined in the active ReplicaSet. In this example there is just one
container image called `nginx`:

```console
$ kubectl get vulnerabilityreports -o wide
NAME                                REPOSITORY      TAG    SCANNER   AGE   CRITICAL   HIGH   MEDIUM   LOW   UNKNOWN
replicaset-nginx-7ff78f74b9-nginx   library/nginx   1.16   Trivy     12s   4          40     26       90    0
```

Similarly, the operator creates a ConfigAuditReport holding the result of auditing the configuration of the active
ReplicaSet controlled by the `nginx` Deployment:

```console
$ kubectl get configauditreports -o wide
NAME                          SCANNER   AGE   DANGER   WARNING   PASS
replicaset-nginx-7ff78f74b9   Polaris   33s   1        9         7
```

Notice that scan reports generated by the operator are controlled by Kubernetes workloads. In our example,
VulnerabilityReport and ConfigAuditReport objects are controlled by the active ReplicaSet of the `nginx` Deployment:

```console
$ kubectl tree deploy nginx
NAMESPACE  NAME                                                       READY  REASON  AGE
default    Deployment/nginx                                           -              51s
default    └─ReplicaSet/nginx-6d4cf56db6                              -              51s
default      ├─ConfigAuditReport/replicaset-nginx-6d4cf56db6          -              46s
default      ├─VulnerabilityReport/replicaset-nginx-6d4cf56db6-nginx  -              31s
default      └─Pod/nginx-6d4cf56db6-fhbm9                             True           51s
```

!!! note
    The [tree] command is a kubectl plugin to browse Kubernetes object hierarchies as a tree.

Moving forward, let's update the container image of the `nginx` Deployment from `nginx:1.16` to `nginx:1.17`. This will
trigger a rolling update of the Deployment and eventually create another ReplicaSet.

```
kubectl set image deployment nginx nginx=nginx:1.17
```

Even this time the operator will pick up changes and rescan our Deployment with updated configuration:

```console
$ kubectl tree deploy nginx
NAMESPACE  NAME                                                       READY  REASON  AGE
default    Deployment/nginx                                           -              86s
default    ├─ReplicaSet/nginx-6d4cf56db6                              -              86s
default    │ ├─ConfigAuditReport/replicaset-nginx-6d4cf56db6          -              81s
default    │ └─VulnerabilityReport/replicaset-nginx-6d4cf56db6-nginx  -              66s
default    └─ReplicaSet/nginx-db749865c                               -              19s
default      ├─ConfigAuditReport/replicaset-nginx-db749865c           -              17s
default      ├─VulnerabilityReport/replicaset-nginx-db749865c-nginx   -              9s
default      └─Pod/nginx-db749865c-lfcp5                              True           19s
```

By following this guide you could realize that the operator knows how to attach VulnerabilityReport and
ConfigAuditReport objects to build-in Kubernetes objects so that looking them up is easy. What's more, in this
approach where a custom resource inherits a life cycle of the built-in resource we could leverage Kubernetes garbage
collection. For example, when the previous ReplicaSet named `nginx-6d4cf56db6` is deleted the VulnerabilityReport named
`replicaset-nginx-6d4cf56db6-nginx` as well as the ConfigAuditReport named `replicaset-nginx-6d4cf56db6` are
automatically garbage collected.

!!! tip
    You can get and describe `vulnerabilityreports` and `configauditreports` as built-in Kubernetes objects:
    ```
    kubectl get vulnerabilityreport replicaset-nginx-db749865c-nginx -o json
    kubectl describe configauditreport replicaset-nginx-db749865c
    ```

Notice that scaling up the `nginx` Deployment will not schedule new scan Jobs because all replica Pods refer to the
same Pod templated defined by the `nginx-db749865c` ReplicaSet.

```
kubectl scale deploy nginx --replicas 3
```

```console
$ kubectl tree deploy nginx
NAMESPACE  NAME                                                       READY  REASON  AGE
default    Deployment/nginx                                           -              2m22s
default    ├─ReplicaSet/nginx-6d4cf56db6                              -              2m22s
default    │ ├─ConfigAuditReport/replicaset-nginx-6d4cf56db6          -              2m17s
default    │ └─VulnerabilityReport/replicaset-nginx-6d4cf56db6-nginx  -              2m2s
default    └─ReplicaSet/nginx-db749865c                               -              75s
default      ├─ConfigAuditReport/replicaset-nginx-db749865c           -              73s
default      ├─VulnerabilityReport/replicaset-nginx-db749865c-nginx   -              65s
default      ├─Pod/nginx-db749865c-lfcp5                              True           75s
default      ├─Pod/nginx-db749865c-tn5k7                              True           12s
default      └─Pod/nginx-db749865c-vjlr9                              True           12s
```

Finally, when you delete the `nginx` Deployment, orphaned security reports will be deleted in the background by the
Kubernetes garbage collection controller.

```
kubectl delete deploy nginx
```

```console
$ kubectl get vuln,configaudit
No resources found in default namespace.
```

!!! Tip
    Use `vuln` and `configaudit` as short names for `vulnerabilityreports` and `configauditreports` resources.

## Infrastructure Scanning

The operator discovers also Kubernetes nodes and runs CIS Kubernetes Benchmark checks on each of them. The results are
stored as CISKubeBenchReport objects. In other words, for a cluster with 3 nodes the operator will eventually create
3 benchmark reports:

```console
$ kubectl get node
NAME                 STATUS   ROLES    AGE     VERSION
kind-control-plane   Ready    master   3h27m   v1.18.8
kind-worker          Ready    <none>   3h26m   v1.18.8
kind-worker2         Ready    <none>   3h26m   v1.18.8
```

```console
$ kubectl get ciskubebenchreports -o wide
NAME                 SCANNER      AGE   FAIL   WARN   INFO   PASS
kind-control-plane   kube-bench   8s    12     40     0      70
kind-worker          kube-bench   9s    2      27     0      18
kind-worker2         kube-bench   9s    2      27     0      18
```

Notice that each CISKubeBenchReport is named after a node and is controlled by that node to inherit its life cycle:

```console
$ kubectl tree node kind-control-plane -A
NAMESPACE        NAME                                              READY  REASON        AGE
                 Node/kind-control-plane                           True   KubeletReady  48m
                 ├─CISKubeBenchReport/kind-control-plane           -                    44m
                 ├─CSINode/kind-control-plane                      -                    48m
kube-node-lease  ├─Lease/kind-control-plane                        -                    48m
kube-system      ├─Pod/etcd-kind-control-plane                     True                 48m
kube-system      ├─Pod/kube-apiserver-kind-control-plane           True                 48m
kube-system      ├─Pod/kube-controller-manager-kind-control-plane  True                 48m
kube-system      └─Pod/kube-scheduler-kind-control-plane           True                 48m
```

## What's Next?

- Find out how the operator scans workloads that use container images from [private registries](./../integrations/private-registries.md).
- By default, the operator uses Trivy as [vulnerability scanner](./../integrations/vulnerability-scanners/index.md)
  and Polaris as [configuration checker](./../integrations/config-checkers/index.md), but you can choose other tools that
  are integrated with Starboard or even implement you own plugins.

[minikube]: https://minikube.sigs.k8s.io/docs/
[kind]: https://kind.sigs.k8s.io/docs/
[tree]: https://github.com/ahmetb/kubectl-tree