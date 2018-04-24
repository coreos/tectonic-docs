<br>
<div class="alert alert-info" role="alert">
<i class="fa fa-exclamation-triangle"></i><b> Note:</b> This documentation is for an alpha feature. For questions and feedback on the Multi-cluster Alpha program, email <a href="mailto:tectonic-alpha-feedback@coreos.com">tectonic-alpha-feedback@coreos.com</a>.
</div>

# Installing the multi-cluster registry

Use Tectonic's multi-cluster registry to:
* maintain a list of available clusters
* enable cluster administrators to configure base security policies, and enforce other standardization policies across multiple clusters

Use this guide to install and configure the registry.

## Install the registry

First, install the multi-cluster registry by downloading and applying the example manifest:
* [multicluster.yaml][multicluster-manifest]

Once downloaded, use `kubectl apply` to install the registry:

```
kubectl apply --record -f <url to multicluster.yaml>
```

Check that the sync software deployed successfully:

```
kubectl get deployments -n tectonic-system -l app=directory-sync
NAME             DESIRED   CURRENT   UP-TO-DATE   AVAILABLE   AGE
directory-sync   1         1         1            1           39s
```

After the sync software starts, it should load the cluster list into the replica cluster:

```
kubectl get clusters
NAME                    AGE
west-coast-production   10s
east-coast-production   10s
```

## Set up the directory cluster

Select one of your clusters to be the "directory cluster", which is the cluster that is the source of truth for your policies. Other clusters will be configured to connect to this one. Check that kubectl is set up correctly and can talk to the cluster:

```
export KUBECONFIG=/path/to/kubeconfig
kubectl get nodes
```

Next, read back the list of Clusters. There shouldn’t be any yet:

```
$ kubectl get clusters
No resources found.
```

Create a new Cluster object that represents this cluster. Make sure this matches the name you installed with:

```
kubectl --namespace=tectonic-system get configmap/tectonic-config -o jsonpath="{..data.clusterName}"
west-coast-production
```

Give the cluster a set of labels. These labels will be used in Cluster Policies:

```
apiVersion: multicluster.coreos.com/v1
kind: Cluster
metadata:
  name: west-coast-production
  labels:
    stage: production
    cloud: aws
    region: us-west-1
    role: replica
  annotations:
    multicluster.coreos.com/console-url: https://west-coast-production.west.example.com
    multicluster.coreos.com/directory: true
spec:
  KubernetesAPIEndpoints:
    ServerEndpoints:
      - ServerAddress: https://west-coast-production-api.west.example.com:443
```

Last, save the file and submit it to the cluster:

```
kubectl apply -f clusters/west-coast-production.yaml
cluster “west-coast-production” created
```

Before setting up a replica cluster, register it in the directory. Change the labels and endpoints as necessary. Be sure that  the `multicluster.coreos.com/directory` annotation is only included on the directory cluster. Submit it to the cluster:

```
kubectl apply -f clusters/east-coast-production.yaml
cluster “east-coast-production” created
```

List the clusters:

```
kubectl get clusters
NAME                    AGE
west-coast-production   1m
east-coast-production   10s
```

The directory cluster is all set up!

## Generate credentials

Next, create credentials for replica clusters to read the directory objects. To do this, use a service account created on the directory cluster, then export it for use on the replica cluster.

Create a Service Account for the replica cluster. By convention, give the Service Account the same name as the replica cluster.

```
kubectl create serviceaccount -n tectonic-multi-cluster east-coast-production
serviceaccount "east-coast-production" created

kubectl get serviceaccounts -n tectonic-multi-cluster
NAME                    SECRETS   AGE
default                 1         15m
east-coast-production   1         12m
```

Bind the new Service Account to the “view clusters” role:

```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: ClusterRoleBinding
metadata:
  name: east-coast-production-view-clusters
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: ClusterRole
  name: tectonic-view-clusters
subjects:
- kind: ServiceAccount
  name: east-coast-production
  namespace: tectonic-multi-cluster

kubectl apply -f east-coast-production-binding.yaml
Clusterrolebinding "east-coast-production-view-clusters" created
```

Download a kubeconfig for the east-coast-production service account from Tectonic Console. This kubeconfig will be submitted as a secret to the replica cluster to read from the directory cluster’s API.

## Set up replica cluster(s)

Switch the kubeconfig to the replica cluster. Check that it is set up correctly and can talk to the cluster:

```
export KUBECONFIG=/path/to/kubeconfig
kubectl get nodes
```

Upload the kubeconfig that was just downloaded as a secret, for the syncer to use:

```
kubectl -n tectonic-system create secret generic \
    directory-sync --from-file=kubeconfig=/path/to/sa-east-coast-production-token-<id>-<clustername>-kubeconfig
secret "tectonic-multi-cluster-local-sync" created
```

## De-register a cluster

Because the cluster directory uses a pull-only model, de-registering is done by first removing the cluster object from the directory cluster:

```
kubectl delete clusters/east-coast-production
cluster “east-coast-production” deleted
```

Next, remove the credentials used for the pull process. If these credentials were ever leaked, this is also one way to prevent malicious access:

```
kubectl delete secrets/east-coast-production-token-fxp4x
Secret "east-coast-production-token-fxp4x" deleted
```

## Next Steps

Use [Cluster Policies][cluster-policies] to set up a common set of Namespaces, RBAC rules, and Resource Limits that apply to the automatically updated list of clusters.


[cluster-policies]: multi-cluster-policy.md
[multicluster-manifest]: ../files/multi-cluster/multicluster.yaml
