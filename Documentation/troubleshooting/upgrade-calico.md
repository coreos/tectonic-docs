# Upgrading Calico Manually

Tectonic included alpha support for Network Security Policy and BGP routing functionality in 1.7.x releases, powered by Calico.

Under the hook, Calico utilized Third Party Resources (TPRs) for configuration. These TPRs need to be upgraded to Custom Resource Definitions (CRDs) before upgrading to 1.8.x. This process is automated, but must be started manually.

After the alpha period, upgrades to Tectonic's network functionality will be fully supported by Tectonic's trademark Automated Operations.

## Back up TPRs

As with all upgrades, start by backing up your TPRs:

```
$ kubectl get globalconfig --all-namespaces --export -o yaml > global-configs.yaml
$ kubectl get ippool --all-namespaces --export -o yaml > ip-pools.yaml
$ kubectl get systemnetworkpolicy.alpha --all-namespaces --export -o yaml > system-network-policies.yaml
```

## Run the migration Job

An automated migration is run as a Kubernetes Job. [Download the Job object][download-job] and create it in the `default` namespace:

```
$ kubectl -n default apply -f upgrade-job.yaml
```

You can check the status of the Job and it's output logs:

```
$ kubectl describe job/calico-upgrade-v2.5
```

After the job is complete, verify that new data exists as CRDs:

```
$ kubectl get ippools.crd.projectcalico.org
$ kubectl get globalfelixconfigs.crd.projectcalico.org
```

## Upgrade the Role Based Access Control rules

[Download the new RBAC rule][download-rules] which adds access to manipulate the new CRD types:

```
$ kubectl -n kube-system apply -f rbac-canal-tectonic-18.yaml
```

## Perform the rolling update

First, update the images to reference the new Calcio version:

```
$ kubectl -n kube-system set image daemonset/kube-calico kube-calico=quay.io/calico/node:v2.6.1 \
    install-cni=quay.io/calico/cni:v1.11.0
```

Watch progress of the update process:

```
$ kubectl -n kube-system get pods | grep calico
```

## Verify the update

To verify the process went smoothly, check that a Pod can be started on the cluster:

```
$ kubectl run --rm -i --tty busybox --image=busybox
```

## Clean up

Clean up the unneeded TPRs and the upgrade job:

```
$ kubectl delete thirdpartyresources/global-bgp-config.projectcalico.org
$ kubectl delete thirdpartyresources/global-bgp-peer.projectcalico.org
$ kubectl delete thirdpartyresources/global-config.projectcalico.org
$ kubectl delete thirdpartyresources/ip-pool.projectcalico.org
$ kubectl delete thirdpartyresources/system-network-policy.alpha.projectcalico.org
$ kubectl delete -f upgrade-job.yaml
```

## Proceed with your Tectonic upgrade

This process ensures that TPRs utilized by Calcio are removed. If you are using other TPRs, remove or migrate them before attempting to upgrade to Tectonic 1.8.

[download-rules]: calico-manifests/rbac-canal-tectonic-18.yaml
[download-job]: calico-manifests/calico-upgrade-job.yaml