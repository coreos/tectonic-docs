# Azure private clusters

Use Tectonic Installer to create clusters within an existing private Azure network. In addition to the standard Azure requirements, private clusters require:
* An existing Virtual Network with a configured and connected VPN link.
* A DNS domain managed by Azure DNS, or, or a 3rd party DNS server that supports DNS UPDATE operations.

Once these requirements are met, follow the guide to [Install Tectonic on Azure with Terraform][azure-terraform].

The machine running Tectonic Installer must be behind the VPN connection to the VNet. This machine must have direct IP connectivity to the VMs in the cluster VNet.

## Connected VPN link

Private Azure networks require that an external VPN be available and running prior to running Tectonic Installer.

For best results, use an OpenVPN connection to an OpenVPN Access Server that is deployed into the cluster VNet.

For more information, see the Microsoft Azure document [Deploy an Openvpn Access Server][deploy-openvpn].

## DNS domain or server

Private clusters must use an external DNS provider, rather than Azure’s default hostname resolution. The selected DNS server must be configured as the default resolver for the nodes.

Tectonic will use DNS only, rather than Load Balancers, to balance traffic to the master nodes of your private clusters on Azure.

Follow the instructions to set up [DNS delegation and custom zones via Azure DNS][azure-dns].

## Tectonic Installer

When creating your cluster, set `tectonic_azure_private_cluster` in the [terraform.tfvars][terraform-tvars] file to `true` to ensure that no public endpoints will be created as a result of the installation process.

## Node discovery

Tectonic uses DNS to discover nodes in private clusters.

### Etcd nodes

* Etcd cluster nodes are managed by the terraform module `modules/azure/etcd`.
* Node VMs are created as an Availability Set (stand-alone instances, deployed across multiple fault domains).
* Etcd nodes are provided a simple discovery mechanism using a VIP + DNS record.

### Master nodes

* Master node VMs are managed by the templates in `modules/azure/master-as`.
* Node VMs are created as an Availability Set (stand-alone instances, deployed across multiple fault domains).
* Master nodes are provided direct access to both the API and the Ingress controller through DNS.

### Worker nodes

* Worker node VMs are managed by the templates in `modules/azure/worker-as`.
* Node VMs are created as an Availability Set (stand-alone instances, deployed across multiple fault domains).
* Worker nodes may be accessed through SSH from any of the master nodes.

## Private Cluster variables

Set `tectonic_azure_private_cluster` in [terraform.tfvars][terraform-tvars] to `true` to ensure the privacy of the cluster.

`tectonic_azure_private_cluster`: (optional string) Set to `true` to create NO public facing endpoints. All traffic is contained within the VNET. A VNET with an already configured and active VPN connection is required and must be supplied using `tectonic_azure_external_vnet_id`. DNS is currently required, either the Azure managed one or configured via the generic DNS module. Default: `false`.


[azure-dns]: azure-terraform.md#DNS
[azure-terraform]: azure-terraform.md
[deploy-openvpn]: https://azure.microsoft.com/en-us/resources/templates/openvpn-access-server-ubuntu/
[terraform-tvars]: https://github.com/coreos/tectonic-installer/tree/master/Documentation/variables/azure.md
