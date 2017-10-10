# Azure private clusters

Use Tectonic Installer to create clusters within an existing private Azure network. In addition to the standard Azure requirements, private clusters also require:
* An existing Virtual Network with a pre-configured and connected VPN link.
* A DNS domain pre-registered into Azure DNS, or a 3rd party DNS server that supports DNS UPDATE operations.

Once these requirements are met, follow the guide to [Install Tectonic on Azure with Terraform][azure-terraform].

## Connected VPN link

Private Azure networks require that an external VPN be available and running prior to running Tectonic Installer.

For more information, see the Microsoft Azure document [Connect an on-premises network to Azure using a VPN gateway][connect-azure].

## DNS domain or server

Private clusters must use an external DNS provider, rather than Azure’s default DNS implementation.

Tectonic will use DNS only, rather than Load Balancers, to route traffic to your private clusters on Azure. Boot a cluster that uses an API Fully Qualified Domain Name that points to the list of master private IP addresses.

Follow the instructions to set up [DNS delegation and custom zones via Azure DNS][azure-dns].

## Tectonic Installer

When creating your cluster, set `tectonic_azure_private_cluster` to `true` to ensure that no public endpoints will be created as a result of the installation process.

## Node Management

Tectonic uses DNS to manage nodes in private clusters.

### Etcd nodes
* Etcd cluster nodes are managed by the terraform module `modules/azure/etcd`.
* Node VMs are created as an Availability Set (stand-alone instances, deployed across multiple fault domains).
* Etcd nodes are provided a simple discovery mechanism using a VIP + DNS record.

### Master nodes
* Master node VMs are managed by the templates in `modules/azure/master-as`.
* Node VMs are created as an Availability Set (stand-alone instances, deployed across multiple fault domains).
* Master nodes are provided direct access to both the API and the Ingress controller through DNS.
* The API Load Balancer is configured with SourceIP session stickiness, to ensure that TCP (including SSH) sessions from the same client land reliably on the same master node. This allows for provisioning the assets and starting bootkube reliably via SSH.

### Worker nodes
* Worker node VMs are managed by the templates in `modules/azure/worker-as`.
* Node VMs are created as an Availability Set (stand-alone instances, deployed across multiple fault domains).
* Worker nodes may be accessed through SSH from any of the master nodes.

## Private Cluster variables

Use `tectonic_azure_private_cluster` to ensure the privacy of the cluster.

`tectonic_azure_private_cluster`: (optional string) Set to `true` to create NO public facing endpoints. All traffic is contained within the VNET. A VNET with an already configured and active VPN connection is required and must be supplied using `tectonic_azure_external_vnet_id`. DNS is currently required, either the Azure managed one or configured via the generic DNS module. Default: `false`.


[azure-dns]: azure-terraform.md#DNS
[azure-terraform]: azure-terraform.md
[connect-azure]: https://docs.microsoft.com/en-us/azure/architecture/reference-architectures/hybrid-networking/vpn
