# TLS Certificate Rotation in Tectonic

__WARNING:__ Rotating certificates by hand can break component connectivity and leave the cluster in an unrecoverable state. Before performing any of these instructions on a live cluster backup your cluster state and migrate critical workloads to another cluster.

This guide will examine rotating the certificates for the following components.

  * Tectonic and Kubernetes components
  * etcd

## Generating new certs

Generating certificates requires the following:

* The `openssl` tool
* SSH access to the cluster
* The original Tectonic CA certificate and private key
* The cluster name and base domain of the cluster
* The API server's cluster internal IP address
* The etcd CA certificate (optional)

Pivoting cluster components to a new CA after the loss of a CA private key is not covered by this guide.

The API server's cluster IP is a hard-coded IP which is dependent on the cluster's service CIDR range. To determine your API server's cluster IP, run the following command:

```
$ kubectl -n default get services kubernetes -o=jsonpath="{.spec.clusterIP}{'\n'}"
10.3.0.1
```

The cluster name and base domain can be inferred from the DNS name of your cluster, since the Tectonic installer constructs the API server DNS name as `${CLUSTER_NAME}-api.${BASE_DOMAIN}` and the console's DNS name as `${CLUSTER_NAME}.${BASE_DOMAIN}`. For example, given the following kubectl configuration the cluster name would be `my-cluster` and the base domain would be `example.coreos.com`:

```yaml
$ kubectl config view
apiVersion: v1
clusters:
- cluster:
    certificate-authority-data: REDACTED
    server: https://my-cluster-api.example.coreos.com:443
  name: my-cluster
contexts:
- context:
    cluster: my-cluster
    user: kubelet
  name: ""
current-context: ""
kind: Config
preferences: {}
users:
- name: kubelet
  user:
    client-certificate-data: REDACTED
    client-key-data: REDACTED
```

Generate a new set of certificates using the [`gencerts.sh`](./rotate/gencerts.sh) script and [`openssl.conf`](./rotate/openssl.conf) provided with these docs.

```
export APISERVER_INTERNAL_IP=10.3.0.1

export CA_CERT=$HOME/infra/tectonic/tectonic_1.8.4-tectonic.3/generated/tls/ca.crt
export CA_KEY=$HOME/infra/tectonic/tectonic_1.8.4-tectonic.3/generated/tls/ca.key

export BASE_DOMAIN=example.coreos.com
export CLUSTER_NAME=my-cluster

# ETCD_CA_CERT is optional and should only be used for etcd clusters
# provisioned by the Tectonic installer.
export ETCD_CA_CERT=$HOME/infra/tectonic/tectonic_1.8.4-tectonic.3/generated/tls/etcd-client-ca.crt

./gencerts.sh generated
```

The script creates a directory of generated TLS assets. If you provided the etcd CA, this will include etcd certificates and manifest patches.

```
generated/
├── auth
│   └── kubeconfig
├── patches
│   ├── identity-grpc-client.patch
│   ├── identity-grpc-server.patch
│   ├── ingress-tls.patch
│   └── kube-apiserver-secret.patch
└── tls
    ├── apiserver.crt
    ├── apiserver.key
    ├── apiserver.txt
    ├── identity-grpc-client.crt
    ├── identity-grpc-client.key
    ├── identity-grpc-client.txt
    ├── identity-grpc-server.crt
    ├── identity-grpc-server.key
    ├── identity-grpc-server.txt
    ├── ingress-server.crt
    ├── ingress-server.key
    ├── ingress-server.txt
    ├── kubelet.crt
    ├── kubelet.key
    └── kubelet.txt

3 directories, 20 files
```

To verify the script worked correctly, use the generated kubeconfig to query the API server.

```
$ kubectl --kubeconfig generated/auth/kubeconfig get nodes
NAME                                        STATUS    ROLES     AGE       VERSION
ip-10-0-44-164.us-west-1.compute.internal   Ready     node      14m       v1.8.4+coreos.0
ip-10-0-45-138.us-west-1.compute.internal   Ready     node      14m       v1.8.4+coreos.0
ip-10-0-52-88.us-west-1.compute.internal    Ready     node      14m       v1.8.4+coreos.0
ip-10-0-6-59.us-west-1.compute.internal     Ready     master    14m       v1.8.4+coreos.0
```

Errors encountered during this step indicate that one or more of the input values was incorrect.

## Rotating certificates for Tectonic and Kubernetes components

TLS assets for components running on top of Kubernetes can be updated using `kubectl`, including self-hosted control plane components such as the API server. To rotate those certificates, patch the manifests and roll the deployments.

__WARNING:__ The following commands MUST use `kubectl patch` and NOT other `kubectl` creation subcommands.

```
kubectl --kubeconfig generated/auth/kubeconfig \
    patch -f generated/patches/identity-grpc-client.patch \
    -p "$( cat generated/patches/identity-grpc-client.patch )"
kubectl --kubeconfig generated/auth/kubeconfig \
    patch -f generated/patches/identity-grpc-server.patch \
    -p "$( cat generated/patches/identity-grpc-server.patch )"
kubectl --kubeconfig generated/auth/kubeconfig \
    patch -f generated/patches/ingress-tls.patch \
    -p "$( cat generated/patches/ingress-tls.patch )"
kubectl --kubeconfig generated/auth/kubeconfig \
    patch -f generated/patches/kube-apiserver-secret.patch \
    -p "$( cat generated/patches/kube-apiserver-secret.patch )"
```

To force the deployments to restart and pick up the new TLS assets, force the rotation of the deployments' components. Note that the API server may become temporarily unavailable after this action.

```
kubectl --kubeconfig generated/auth/kubeconfig \
    delete pods -n kube-system -l k8s-app=kube-apiserver
kubectl --kubeconfig generated/auth/kubeconfig \
    patch deployments -n tectonic-system tectonic-identity \
    -p "{\"spec\":{\"template\":{\"metadata\":{\"labels\":{\"date\":\"`date +'%s'`\"}}}}}"
kubectl --kubeconfig generated/auth/kubeconfig \
    patch deployments -n tectonic-system tectonic-console \
    -p "{\"spec\":{\"template\":{\"metadata\":{\"labels\":{\"date\":\"`date +'%s'`\"}}}}}"
```

## Rotating kubelet certs

Unlike other cluster components, kubelets are configured through host files and require SSH access to modify. Because Tectonic often deploys worker nodes behind firewalls, this document uses one of the control plane nodes as a [bastion host][bastion-host] for access to the cluster.

First, choose one of the control plane nodes to act as an SSH bastion host.

```
BASTION=$( kubectl --kubeconfig generated/auth/kubeconfig get nodes -o=jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}' | awk '{ print $1 }' )
echo "Bastion is $BASTION"

PROXY_COMMAND="ssh -o StrictHostKeyChecking=no -q -x core@${BASTION} -W %h:22"
```

If the cluster's nodes don't populate `ExternalIP`, manually set this environment variable.

For each of the nodes, copy the generated `kubeconfig` to the host and restart the kubelet.

```
for IP in $( kubectl --kubeconfig generated/auth/kubeconfig get nodes -o=jsonpath='{.items[*].status.addresses[?(@.type=="InternalIP")].address}' ); do
    echo "Kubelet on $IP restarting"
    scp -o StrictHostKeyChecking=no -o ForwardAgent=yes \
        -o ProxyCommand="$PROXY_COMMAND" generated/auth/kubeconfig core@$IP:/home/core/kubeconfig

    ssh -A -o StrictHostKeyChecking=no -o ProxyCommand="$PROXY_COMMAND" \
        core@$IP sudo cp /etc/kubernetes/kubeconfig /etc/kubernetes/kubeconfig.bak

    ssh -A -o StrictHostKeyChecking=no -o ProxyCommand="$PROXY_COMMAND" \
        core@$IP sudo mv /home/core/kubeconfig /etc/kubernetes/kubeconfig

    ssh -A -o StrictHostKeyChecking=no -o ProxyCommand="$PROXY_COMMAND" \
        core@$IP sudo systemctl restart kubelet
    echo "Kubelet on $IP restarted"
    sleep 5
done
```

__WARNING:__ Clusters with workers in auto-scaling groups will continue to have old `kubeconfig` files delivered through scripts in their user-data. When new workers are brought online, they must be manually reconfigured with the new `kubeconfig ` or the user-data must be modified.

Several node agents re-use the kubelet's kubeconfig for authentication against the API server. Rotate these as well:

```
kubectl --kubeconfig generated/auth/kubeconfig \
    patch daemonsets -n kube-system kube-proxy \
    -p "{\"spec\":{\"template\":{\"metadata\":{\"labels\":{\"date\":\"`date +'%s'`\"}}}}}" 
kubectl --kubeconfig generated/auth/kubeconfig \
    patch daemonsets -n kube-system pod-checkpointer \
    -p "{\"spec\":{\"template\":{\"metadata\":{\"labels\":{\"date\":\"`date +'%s'`\"}}}}}" 
kubectl --kubeconfig generated/auth/kubeconfig \
    patch daemonsets -n tectonic-system node-agent \
    -p "{\"spec\":{\"template\":{\"metadata\":{\"labels\":{\"date\":\"`date +'%s'`\"}}}}}" 
```

## Rotating the etcd CA

Due to a [bug in the Tectonic Installer][3156], many old installs will have rendered an etcd CA certificate without the private key. As a result, to update certificates this guide rotates the entire CA, not just the individual certificates. This requires:

* Introducing the new CA to the certificate bundle of the etcd members and API server.
* Updating the serving and client certs.

If `ETCD_CA_CERT` was supplied to the generation script above, it will have generated a new etcd CA with associated certificates.

```
generated/etcd/
├── ca_bundle.pem
├── patches
│   ├── etcd-ca.patch
│   └── etcd-client-cert.patch
└── tls
    ├── ca.crt
    ├── ca.key
    ├── ca.txt
    ├── client.crt
    ├── client.key
    ├── client.txt
    ├── peer.crt
    ├── peer.key
    ├── peer.txt
    ├── server.crt
    ├── server.key
    └── server.txt

2 directories, 15 files
```

First, introduce the new etcd CA bundle and restart the API server. This bundle includes both the new and old CA certificates.

```
kubectl --kubeconfig generated/auth/kubeconfig \
    patch -f generated/etcd/patches/etcd-ca.patch \
    -p "$( cat generated/etcd/patches/etcd-ca.patch )"
kubectl --kubeconfig generated/auth/kubeconfig \
    delete pods -n kube-system -l k8s-app=kube-apiserver
```

Etcd members also require using a bastion host for access. The same instructions as above can be used to identify the IP of a control plane instance.

```
BASTION=$( kubectl --kubeconfig generated/auth/kubeconfig get nodes -o=jsonpath='{.items[*].status.addresses[?(@.type=="ExternalIP")].address}' | awk '{ print $1 }' )
echo "Bastion is $BASTION"

PROXY_COMMAND="ssh -o StrictHostKeyChecking=no -q -x core@${BASTION} -W %h:22"
```

Inspect the API server's --etcd-servers flag to find the address of a cluster's etcd instances.

```
$ kubectl --kubeconfig generated/auth/kubeconfig \
    get daemonsets -n kube-system kube-apiserver -o yaml | grep etcd-servers
        - --etcd-servers=https://my-cluster-etcd-0.example.coreos.com:2379,https://my-cluster-etcd-1.example.coreos.com:2379,https://my-cluster-etcd-2.example.coreos.com:2379
```

Ensure the generated server and peer certs match these DNS names:

```
$ grep DNS generated/etcd/tls/server.txt 
                DNS:*.example.coreos.com
$ grep DNS generated/etcd/tls/peer.txt 
                DNS:*.example.coreos.com
```

For each of the nodes, copy the new CA bundle and restart etcd. The `ETCD_INSTANCES` environment variable should be set to the addresses of your etcd instances. 

```
ETCD_INSTANCES="$( kubectl --kubeconfig generated/auth/kubeconfig \
    get daemonsets -n kube-system kube-apiserver -o yaml | grep etcd-servers \
    | sed 's/^.*=//g' | sed 's/,/ /g' | sed 's/https:\/\///g' | sed 's/:[0-9]*//g' )"
echo "Rotating etcd CA bundles for instances $ETCD_INSTANCES"

for ADDR in $ETCD_INSTANCES; do
    echo "etcd on $ADDR restarting"
    scp -o StrictHostKeyChecking=no -o ForwardAgent=yes \
        -o ProxyCommand="$PROXY_COMMAND" generated/etcd/ca_bundle.pem core@$ADDR:/home/core/ca.crt

    ssh -A -o StrictHostKeyChecking=no -o ProxyCommand="$PROXY_COMMAND" \
        core@$ADDR sudo chown etcd:etcd /home/core/ca.crt

    ssh -A -o StrictHostKeyChecking=no -o ProxyCommand="$PROXY_COMMAND" \
        core@$ADDR sudo cp -r /etc/ssl/etcd /etc/ssl/etcd.bak

    ssh -A -o StrictHostKeyChecking=no -o ProxyCommand="$PROXY_COMMAND" \
        core@$ADDR sudo mv /home/core/ca.crt /etc/ssl/etcd/ca.crt

    ssh -A -o StrictHostKeyChecking=no -o ProxyCommand="$PROXY_COMMAND" \
        core@$ADDR sudo systemctl restart etcd-member
    echo "etcd on $ADDR restarted"
    sleep 10
done
```

Once all etcd instances are seeded with the new CA certificate, rotate the API server's client certs:

```
kubectl --kubeconfig generated/auth/kubeconfig \
    patch -f generated/etcd/patches/etcd-client-cert.patch \
    -p "$( cat generated/etcd/patches/etcd-client-cert.patch )"
kubectl --kubeconfig generated/auth/kubeconfig \
    delete pods -n kube-system -l k8s-app=kube-apiserver
```

Finally, for each etcd instance, rotate the peer and serving certs:

```
for ADDR in $ETCD_INSTANCES; do
    echo "etcd on $ADDR restarting"
    scp -o StrictHostKeyChecking=no -o ForwardAgent=yes \
        -o ProxyCommand="$PROXY_COMMAND" \
        generated/etcd/tls/{peer.crt,peer.key,server.crt,server.key} \
        core@$ADDR:/home/core

    ssh -A -o StrictHostKeyChecking=no -o ProxyCommand="$PROXY_COMMAND" \
        core@$ADDR sudo chown etcd:etcd /home/core/{peer.crt,peer.key,server.crt,server.key}

    ssh -A -o StrictHostKeyChecking=no -o ProxyCommand="$PROXY_COMMAND" \
        core@$ADDR sudo chmod 0400 /home/core/{peer.crt,peer.key,server.crt,server.key}

    ssh -A -o StrictHostKeyChecking=no -o ProxyCommand="$PROXY_COMMAND" \
        core@$ADDR sudo mv /home/core/{peer.crt,peer.key,server.crt,server.key} \
        /etc/ssl/etcd/

    ssh -A -o StrictHostKeyChecking=no -o ProxyCommand="$PROXY_COMMAND" \
        core@$ADDR sudo systemctl restart etcd-member
    echo "etcd on $ADDR restarted"
    sleep 10
done
```

[bastion-host]: https://en.wikipedia.org/wiki/Bastion_host
[3156]: https://github.com/coreos/tectonic-installer/issues/3156
