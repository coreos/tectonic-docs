# Vault Open Cloud Service

Tectonic's Vault Open Cloud Service provides a one-click, fully managed Vault Secret Management Service on-top of a Tectonic cluster.

* **Secure by Default:** Hands-free automated creation of TLS certificates between all components ensure all best practices are followed for secret security. Further, unseal operations are easy via the API.
* **Highly available:** Multiple instances of Vault are clustered together via an etcd backend and secured.
* **Safe Upgrades:** Rolling out a new Vault version is as easy as updating the Vault Cluster definition; everything is automatically handled using Vault best practices while pausing for unseal tokens.

## Deploying Vault OCS

Use Tectonic Console to enable Vault OCS for selected namespaces. Then, use kubectl and the [Vault Commands (CLI)][vault-cli] tool to initialize the instance and unseal the cluster.

Creating the Vault Service will automatically generate TLS certs for the Service, and create and deploy a supporting etcd cluster. It will create two Vault Pods, and a Service routing to them.

By default, objects created using the Vault OCS will be labeled `app=vault`.

Using the Vault Open Cloud Service to deploy a Vault instance will create the following Kubernetes objects:
* A Vault CRD
* 2 Pod Vault Deployment
* A Vault Replica Set
* A Vault Service of type: Cluster IP
* A Vault Config Map
* A Vault client and server TLS Secret
* An etcd client, peer, and server TLS Secret
* An etcd server TLS

## Initializing and Unsealing

Once enabled, use Console to create a new Vault Service, then use the Vault Commands (CLI) tool to initialize the instance, obtain the keys, and unseal the cluster.

### Proxy the Vault instance to your laptop

```
$ kubectl -n default get vault example -o jsonpath='{.status.nodes.sealed[0]}' | xargs -0 -I {} kubectl -n default port-forward {} 8200
```

### Point the vault CLI to the local endpoint

Because communication passes over localhost through a secured tunnel, verification may be skipped.

```
$ export VAULT_SKIP_VERIFY="true"
$ export VAULT_ADDR='https://localhost:8200'
```

### Use the Vault CLI tools to initialize the cluster

The initialization flow generates key material that is used to securely unseal the cluster. A cluster is sealed when it is first created, or when the master encryption keys are released from memory, typically due to a restart. To unseal the cluster, provide the generated keys.

```
$ vault init -key-shares=1 -key-threshold=1
Unseal Key 1: <string>
Initial Root Token: <string>

Vault initialized with 1 keys and a key threshold of 1. Please
securely distribute the above keys. When the vault is re-sealed,
restarted, or stopped, you must provide at least 1 of these keys
to unseal it again.

Vault does not store the master key. Without at least 1 keys,
your vault will remain permanently sealed.
```

### Unseal

Use the Unseal Key returned from `vault init` to unseal the cluster.

```
$ vault unseal
Key (will be hidden):
Sealed: false
Key Shares: 1
Key Threshold: 1
Unseal Progress: 0
Unseal Nonce:
```

The first node unsealed in a multi-node Vault cluster will become the active node. The active node holds the leader election lock. The other unsealed nodes become standbys with the status of `Running` but condition of `ContainersNotReady` until they need to take over.

### Repeat for each Vault instance

Repeat the unseal process by port-forwarding to each instance listed as sealed. This will make them available for failover if the active instance fails.

```
$ kubectl -n default get vault example -o jsonpath='{.status.sealedNodes}'
```

## Working with Kubernetes Services and Secrets

Use the Vault Service’s YAML manifest to obtain the clientSecret and serverSecret required to expose the Vault instance to your apps.

For more information, see [Configuring Vault nodes][configure-vault].


[configure-vault]: https://github.com/coreos-inc/vault-operator/blob/master/doc/user/vault.md#writing-secrets-to-the-active-node
[vault-cli]: https://www.vaultproject.io/docs/install/index.html
