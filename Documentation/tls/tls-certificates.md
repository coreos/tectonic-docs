# Transport Layer Security (TLS) Certificates

Tectonic secures all communications between cluster components. At install time, certificates can be provided in the following ways, from most automated to most customized:

1. Installer generates a new CA, intermediate CAs and all certificates (default)
2. Installer uses your provided CA (or intermediate) to generate all certificates
3. Certificates are generated out-of-band (guides below) and provided to the Installer

Read about the [TLS topology][tls-topology] for more technical information about the intermediate CAs used between components.

## Components and Certificate Sources

| Type of Traffic | Source of Certificate |
|:----------|:----------------------|
| API communication (external and internal) | Generated (default) </br> [User-provided][tls-identity] |
| API to etcd cluster | Generated (default) </br> [User-provided][tls-etcd] |
| Kubelet to anything | Generated (default) </br> [User-provided][tls-kube] |
| Using the Tectonic Console and Ingress | Generated (default) </br> [User-provided][tls-ingress] |
| Pod to Pod communication | Optionally secured by the application deployer |
| Ingress to Pod communication | Optionally secured by the application deployer |

### Default certificate generation

The default behavior is to generate a new CA and use this to sign all certificates used in the cluster. This CA and the certificates will be stored on the machine where Tectonic Installer runs for your safe keeping.

Users accessing the Tectonic Console may see browser warnings about untrusted self-signed certificates.

### Provide your own Certificate Authority (CA)

Provide a Certificate Authority Certificate and Key (in PEM format) during Tectonic installation. This CA may be entered using either the GUI or the Terraform CLI installation process. If provided, Tectonic will use this CA to sign all generated certificates for the cluster. This feature is provided to allow users who manage their own PKI infrastructure to create an intermediate CA for use in a Tectonic cluster. This enables all clients that trust the root CA to also trust any provided intermediate CA via the trust chain.

### Provide your own certificates

Certificates that have been generated or purchased out-of-band can be provided for certain cluster components. The Installer will generate certificates for all other components unless provided by using the following guides:

 * [User-provided Identity certificates][tls-identity]
 * [User-provided etcd certificates][tls-etcd]
 * [User-provided API certificates][tls-kube]
 * [User-provided Console/Ingress certificates][tls-ingress]

For more general Kubernetes information about TLS, see the project's [Manage TLS Certificates in a Cluster][manage-tls] guide.


[manage-tls]: https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/
[tls-etcd]: tls-etcd.md
[tls-identity]: tls-identity.md
[tls-ingress]: tls-ingress.md
[tls-kube]: tls-kube.md
[tls-topology]: tls-topology.md
