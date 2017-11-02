# Custom DNS service with RFC 2136 UPDATES ("DDNS")

This document describes replacing the default DNS Terraform module with the DDNS module, allowing provisioning of DNS name servers that support the [RFC 2136 `DNS UPDATE` protocol][rfc2136]. This can be used, for example, to replace a cloud provider's included DNS service with a custom external DNS service.

## Selecting the DDNS module

Currently, switching from Terraform's DNS module to the DDNS module requires editing the platform's `main.tf` configuration.

### Remove platform DNS module

Remove the existing DNS module for the platform by commenting out the DNS module stanza from the platform's Terraform file, e.g., `platforms/azure/main.tf`:

```go
/*
module "dns" {
  source = "../../modules/dns/azure"

  etcd_count   = "${var.tectonic_self_hosted_etcd != "" ? 0 : var.tectonic_etcd_count}"
  master_count = "${var.tectonic_master_count}"
  worker_count = "${var.tectonic_worker_count}"

  etcd_ip_addresses    = "${module.vnet.etcd_endpoints}"
  master_ip_addresses  = "${module.vnet.master_private_ip_addresses}"
  worker_ip_addresses  = "${module.vnet.worker_private_ip_addresses}"
  api_ip_addresses     = "${module.vnet.api_ip_addresses}"
  console_ip_addresses = "${module.vnet.console_ip_addresses}"

  base_domain  = "${var.tectonic_base_domain}"
  cluster_name = "${var.tectonic_cluster_name}"
  cluster_id   = "${module.tectonic.cluster_id}"

  location             = "${var.tectonic_azure_location}"
  external_dns_zone_id = "${var.tectonic_azure_external_dns_zone_id}"

  extra_tags = "${var.tectonic_azure_extra_tags}"
}
*/
```

### Add Terraform DDNS module

Configure the DDNS module to replace the removed DNS module by editing the platform's Terraform file to add this DDNS module stanza:

```go
module "dns" {
  source = "../../modules/dns/ddns"

  etcd_count   = "${var.tectonic_self_hosted_etcd != "" ? 0 : var.tectonic_etcd_count}"
  master_count = "${var.tectonic_master_count}"
  worker_count = "${var.tectonic_worker_count}"

  etcd_ip_addresses    = "${module.vnet.etcd_endpoints}"
  master_ip_addresses  = "${module.vnet.master_private_ip_addresses}"
  worker_ip_addresses  = "${module.vnet.worker_private_ip_addresses}"
  api_ip_addresses     = "${module.vnet.api_ip_addresses}"
  console_ip_addresses = "${module.vnet.console_ip_addresses}"

  base_domain  = "${var.tectonic_base_domain}"
  cluster_name = "${var.tectonic_cluster_name}"

  dns_server        = "${var.tectonic_ddns_server}"
  dns_key_name      = "${var.tectonic_ddns_key_name}"
  dns_key_secret    = "${var.tectonic_ddns_key_secret}"
  dns_key_algorithm = "${var.tectonic_ddns_key_algorithm}"
}
```

## Configure DDNS variables

Now that the platform will use the DDNS module, configure the key DDNS variables in the cluster's `terraform.tfvars` variables file:

* `tectonic_base_domain`: The domain or subdomain name chosen for the cluster.
* `tectonic_ddns_server`: The address of the DNS nameserver for the chosen domain.
* `tectonic_ddns_key_name`: The Transaction Signature (TSIG) key name, e.g., `example.com.`.
* `tectonic_ddns_key_secret`: The decryption secret string for the TSIG key.
* `tectonic_ddns_key_algorithm`: TSIG key's encryption algorithm. One of `hmac-md5`, `hmac-sha1`, `hmac-sha256`, or `hmac-sha512`.

## Continue Tectonic installation

With the DDNS module configured to use the chosen name server, proceed with Tectonic installation for the platform:

* [Tectonic on AWS][install-aws]
* [Tectonic on Azure][install-azure]
* [Tectonic on bare metal][install-bm]


[install-aws]: ../aws/aws-terraform.md
[install-azure]: ../azure/azure-terraform.md
[install-bm]: ../bare-metal/index.md
[rfc2136]: https://tools.ietf.org/html/rfc2136
