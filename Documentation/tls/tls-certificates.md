# Transport Layer Security (TLS) Certificates

Tectonic will ensure that Kubernetes is configured to use TLS to secure all communications between Kubernetes components. By default, Tectonic will generate these certificates. These defaults may be overridden using the methods listed below.

1. Installer generates a new CA, intermediate CAs and all certificates (default)
2. Installer uses your provided CA (or intermediate) to generate all certificates
3. Certificates are generated out-of-band (guides below) and provided to the Installer

Read about the [TLS topology][tls-topology] for more technical information about the intermediate CAs used between components.

## Components and Certificate Sources

| Type of Traffic | Source of Certificate |
|:----------|:----------------------|
| User authentication with the cluster <br><br> (Console, API Server, and Identity Pods) | Generated (default) <br><br> [User-provided][tls-identity] |
| API to etcd cluster | Generated (default) <br><br> [User-provided][tls-etcd] |
| Kubelet to anything (including the API Server) | Generated (default) <br><br> [User-provided][tls-kube] |
| Tectonic Ingress certificate | Generated (default) <br><br> [User-provided][tls-ingress] |
| Pod to Pod communication | Optionally secured by the application deployer |
| Ingress to Pod communication | Optionally secured by the application deployer |

Note that if the kubelet cert is configured, all kubelets will authenticate with the same cert. They are not unique to each node.

### Default certificate generation

The default behavior is to generate a new CA and use this to sign all certificates used in the cluster. This CA and the certificates will be stored on the machine where Tectonic Installer runs for your safe keeping.

This CA is not used again after the installation process is complete. The keys are made available in the assets bundle but are not installed in the cluster. No certificates are created or used during the life of the cluster. They are used only while Terraform is running.

Users accessing the Tectonic Console may see browser warnings about untrusted self-signed certificates. To provide a different certificate for Tectonic Console, follow the instructions to [enable custom Tectonic Console TLS certificates][tls-ingress].

### Provide your own Certificate Authority (CA)

Provide a Certificate Authority Certificate and Key (in PEM format) during Tectonic installation. This CA may be entered using either the GUI or the Terraform CLI installation process. If provided, Tectonic will use this CA to sign all generated certificates for the cluster. This feature is provided to allow users who manage their own PKI infrastructure to create an intermediate CA for use in a Tectonic cluster. This enables all clients that trust the root CA to also trust any provided intermediate CA via the trust chain.

This enables all clients that trust the root CA to also trust any certificates signed by the provided intermediate CA. In this model, Tectonic uses the CA key only during the Terraform stage of the cluster bringup. Tectonic does not generate or sign certificates during the cluster's lifecycle.

### Provide your own certificates

Certificates that have been generated and signed by either a corporate CA or purchased from a vendor out-of-band can be used for certain cluster components. Unless overridden as described, Tectonic installer will sign the certificates with a self signed CA. The Installer will generate certificates for all other components unless provided by using the following guides:

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
