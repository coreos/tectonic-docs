# Custom DNS service with RFC 2136 UPDATES ("DDNS")

This document describes replacing the a cloud provider (Azure, AWS) DNS Terraform module with the DDNS module, allowing provisining configuration of custom, external DNS name servers supporting the [RFC 2136 `UPDATE` protocol][rfc2136].

## Selecting the DDNS module

Currently, switching from the standard Azure DNS module to the DDNS module requires editing the platform's `main.tf` configuration.

### Remove platform DNS module

Remove the existing DNS module in your platform, e.g., `platforms/azure/main.tf`, by commenting out lines matching these:

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

Configure the DDNS module to replace the standard DNS module for the platform by editing the platform Terraform file, e.g., `platforms/azure/main.tf`, to add these lines:

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

Now that the platform will use the DDNS module, configure the key DDNS variables in the `terraform.tfvars` variables file for the cluster:

* `tectonic_base_domain`: The domain or subdomain name chosen for the cluster.
* `tectonic_ddns_server`: The address of the DNS name server for the chosen domain.
* `tectonic_ddns_key_name`: The authentication key for DNS `UPDATE`s.
* `tectonic_ddns_key_secret`: The decryption secret for the authentication key.
* `tectonic_ddns_key_algorithm`: Authentication key's [encryption algorithm][key-algo].

## Continue Tectonic installation

With DNS configured to use the chosen name server, proceed with remaining configuration and Tectonic installation on either [AWS][install-aws] or [Azure][install-azure].


[key-algo]: https://www.iana.org/assignments/dns-sec-alg-numbers/dns-sec-alg-numbers.xhtml
[install-aws]: ../aws/aws-terraform.md
[install-azure]: ../install/azure/azure-terraform.md
[rfc2136]: https://tools.ietf.org/html/rfc2136
