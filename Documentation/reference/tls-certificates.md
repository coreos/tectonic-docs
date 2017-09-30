# Transport Layer Security (TLS) Certificates

Tectonic secures all communications between:

* Anything and the kube apiserver (either external or internal to the cluster)
* Anything and any kubelet
* The apiserver and the etcd cluster

With Tectonic, TLS certs may be provided for Ingress, and through Ingress for Tectonic Identity (Dex), and Console.

To enable custom TLS certs, provide a Certificate Authority Certificate and Key (in PEM format) during Tectonic installation. This CA may be entered using either the GUI or the Terraform CLI installation process. If provided, Tectonic will use this CA to sign all generated certificates for the cluster. This feature is provided to allow users who manage their own PKI infrastructure to create an intermediate CA for use in a Tectonic cluster. This enables all clients that trust the root CA to also trust any provided intermediate CA via the trust chain.

Selecting the default, self-signed, will enable TLS, and will prevent user-provided TLS from being enabled for the cluster.

If provided for the Tectonic Ingress controller, Tectonic will use user-provided certificate to secure Ingress in front of the Tectonic Console.

You may also provide a certificate only for the Tectonic Ingress controller. If available, Tectonic will use this user-provided certificate to secure Ingress for the Tectonic Console. This will also secure communication to Tectonic Identity, Prometheus, and any other service that uses the Tectonic Ingress controller while leveraging the Console URL.

Both Tectonic Identity and Prometheus are routed through Tectonic Ingress, and will therefore be secured by either the provided Ingress certificate or the self-signed certificate provided by Tectonic.

At the Kubernetes level, TLS certs may be enabled for:
* API server to etcd
* Kubelet to API
* User to API
* Tectonic Console/Ingress

TLS certs may not be enabled for:
* Pod to Pod communication
* Ingress to Pod communication


For more information, see the Kubernetes document: [Manage TLS Certificates in a Cluster][manage-tls].

## Providing a TLS certification for Tectonic Ingress

First, comment out the existing self-signed Ingress TLS in your platform. For example: `platforms/aws/tectonic.tf`.

```
/*
module "ingress_certs" {
  source = "../../modules/tls/ingress/self-signed"

  base_address = "${module.masters.ingress_internal_fqdn}"
  ca_cert_pem  = "${module.kube_certs.ca_cert_pem}"
  ca_key_alg   = "${module.kube_certs.ca_key_alg}"
  ca_key_pem   = "${module.kube_certs.ca_key_pem}"
}
*/
```

Then, configure the user provided certificates in your platform:

```
module "ingress_certs" {
  source = "../../modules/tls/ingress/user-provided"

  ca_cert_pem = <<EOF
-----BEGIN CERTIFICATE-----
<contents of the public CA certificate in PEM format>
-----END CERTIFICATE-----
EOF

  cert_pem = <<EOF
-----BEGIN CERTIFICATE-----
<contents of the public ingress certificate signed by the above CA in PEM format>
-----END CERTIFICATE-----
EOF

  key_pem = <<EOF
-----BEGIN RSA PRIVATE KEY-----
<contents of the private ingress key used to generate the above certificate PEM format>
-----END RSA PRIVATE KEY-----
EOF
}
```


[manage-tls]: https://kubernetes.io/docs/tasks/tls/managing-tls-in-a-cluster/
