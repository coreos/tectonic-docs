# Install Tectonic on Azure with Terraform

This guide deploys a Tectonic cluster on an Azure account.

The Azure platform templates generally adhere to the standards defined by the project [conventions][conventions] and [generic platform requirements][generic]. This document aims to clarify the implementation details specific to the Azure platform.

## Prerequisites

### CoreOS account

Tectonic Installer requires the License and Pull Secret provided with a CoreOS account. To obtain this information and up to 10 free nodes, create a CoreOS account.

1. Go to [https://account.coreos.com/login][account-login], and click *Sign Up*.

2. Check your inbox for a confirmation email. Click through to accept the terms of the license, activate your account, and be redirected to the *Account Overview* page.

3. Click "Free for use up to 10 nodes" under Tectonic. Enter your contact information, and click *Get License for 10 nodes*.

Once the update has processed, the *Overview* window will refresh to include links to download the License and Pull Secret.

### Terraform

Tectonic Installer includes the required version of Terraform. See the [Tectonic Installer release notes][release-notes] for information about which Terraform versions are compatible.

### DNS

Two methods of providing DNS for the Tectonic installation are supported:

#### Azure-provided DNS

This is Azure's default DNS implementation. For more information, see the [Azure DNS overview][azure-dns].

To use Azure-provided DNS, `tectonic_base_domain` must be set to `""`(empty string).

#### DNS delegation and custom zones via Azure DNS

To configure a custom domain and the associated records in an Azure DNS zone (e.g., `${cluster_name}.foo.bar`):

* The custom domain must be specified using `tectonic_base_domain`
* The domain must be publicly discoverable. The Tectonic installer uses the created record to access the cluster and complete configuration. See the Microsoft Azure documentation for instructions on how to [delegate a domain to Azure DNS][domain-delegation].
* An Azure DNS zone matching the chosen `tectonic_base_domain` must be created prior to running the installer. The full resource ID of the DNS zone must then be referenced in `tectonic_azure_external_dns_zone_id`

### Azure CLI

The [Azure CLI][azure-cli] is required to generate Azure credentials.

### ssh-agent

Ensure `ssh-agent` is running:
```
$ eval $(ssh-agent)
```

Add the SSH key that will be used for the Tectonic installation to `ssh-agent`:
```
$ ssh-add <path-to-ssh-private-key>
```

Verify that the SSH key identity is available to the ssh-agent:
```
$ ssh-add -L
```

Reference the absolute path of the **_public_** component of the SSH key in `tectonic_azure_ssh_key`.

Without this, terraform is not able to SSH copy the assets and start bootkube.
Also ensure the SSH known_hosts file doesn't have old records for the API DNS name, because key fingerprints will not match.

## Deploying the Tectonic cluster

### Download and extract Tectonic Installer

Open a new terminal and run the following command to download Tectonic Installer.

```bash
$ curl -O https://releases.tectonic.com/releases/tectonic_1.7.9-tectonic.1.tar.gz
$ curl -O https://releases.tectonic.com/releases/tectonic_1.7.9-tectonic.1.tar.gz.sig
```

Verify the release has been signed by the [CoreOS App Signing Key][verification-key].

```bash
$ gpg2 --keyserver pgp.mit.edu --recv-key 18AD5014C99EF7E3BA5F6CE950BDD3E0FC8A365E
$ gpg2 --verify tectonic_1.7.9-tectonic.1.tar.gz.sig tectonic_1.7.9-tectonic.1.tar.gz
# gpg2: Good signature from "CoreOS Application Signing Key <security@coreos.com>"
```

Extract the tarball and navigate to the `tectonic` directory.

```bash
$ tar xzvf tectonic_1.7.9-tectonic.1.tar.gz
$ cd tectonic
```

### Initialize and configure Terraform

Add the `terraform` binary to the `PATH`. The platform is one of `darwin` or `linux`.

```bash
$ export PATH=$(pwd)/tectonic-installer/darwin:$PATH # Put the `terraform` binary on PATH
```

Initialize the Tectonic Terraform modules.

```bash
$ terraform init platforms/azure
Downloading modules...
Get: modules/bootstrap-ssh
Get: modules/azure/resource-group
Get: modules/azure/vnet
Get: modules/azure/etcd
Get: modules/ignition
Get: modules/azure/master-as
Get: modules/ignition
Get: modules/azure/worker-as
Get: modules/dns/azure
Get: modules/tls/kube/self-signed
Get: modules/tls/etcd
Get: modules/tls/ingress/self-signed
Get: modules/tls/identity/self-signed
Get: modules/bootkube
Get: modules/tectonic
Get: modules/net/flannel-vxlan
Get: modules/net/calico-network-policy

Initializing provider plugins...
   Checking for available provider plugins on https://releases.hashicorp.com...
```

Terraform will download any available plugins, and report when initialization is complete.

### Generate credentials with Azure CLI

Execute `az login` to obtain an authentication token. See the [Azure CLI docs][login] for more information. Once logged in, note the `id` field of the output from the `az login` command. This is a simple way to retrieve the Subscription ID for the Azure account.

Next, add a new role assignment for the Installer to use:

```
$ az ad sp create-for-rbac -n "http://tectonic" --role contributor
Retrying role assignment creation: 1/24
Retrying role assignment creation: 2/24
{
 "appId": "generated-app-id",
 "displayName": "azure-cli-2017-01-01",
 "name": "http://tectonic-coreos",
 "password": "generated-pass",
 "tenant": "generated-tenant"
}
```

Export the following environment variables with values obtained from the output of the role assignment. As noted above, `ARM_SUBSCRIPTION_ID` is the `id` of the Azure account returned by `az login`.

```
# id field in az login output
$ export ARM_SUBSCRIPTION_ID=abc-123-456
# appID field in az ad output
$ export ARM_CLIENT_ID=generated-app-id
# password field in az ad output
$ export ARM_CLIENT_SECRET=generated-pass
# tenant field in az ad output
$ export ARM_TENANT_ID=generated-tenant
```

With the Azure environment set, specify the deployment details for the cluster.

## Tailor the deployment

Choose a cluster name to identify the cluster. Export an environment variable with the chosen cluster name. This example names the cluster `my-cluster`.

```
$ export CLUSTER=my-cluster
```

Create a build directory for the new cluster and copy the example file into it:

```
$ mkdir -p build/${CLUSTER}
$ cp examples/terraform.tfvars.azure build/${CLUSTER}/terraform.tfvars
```

### Key values for basic Azure deployment

These are the basic values that must be adjusted for each Tectonic deployment on Azure.

#### Environment variables

Set these sensitive values in the environment. The `tectonic_admin_password` will be encrypted before storage or transport:

* `TF_VAR_tectonic_admin_email` - String giving the email address used as user name for the initial Console login
* `TF_VAR_tectonic_admin_password` - Plaintext password string for initial Console login
* `TF_VAR_tectonic_azure_client_secret` - Generated, obfuscated password string matching `ARM_CLIENT_SECRET` and `password` value from `az ad` output, above
* `TF_VAR_tectonic_azure_location` - Lowercase catenated string giving the Azure location name (example: `centralus`)

For example, in the `bash(1)` shell, replace the quoted values with those for the cluster being deployed and run the following commands:

```bash
$ export TF_VAR_tectonic_admin_email="admin@example.com"
$ export TF_VAR_tectonic_admin_password="pl41nT3xt"
$ export TF_VAR_tectonic_azure_client_secret=${ARM_CLIENT_SECRET}
$ export TF_VAR_tectonic_azure_location="centralus"
...
```

#### Terraform variables file

Edit the parameters in `build/$CLUSTER/terraform.tfvars` with the deployment's Azure details, domain name, license, and pull secret. See the details of each value below in the [terraform.tfvars][terraform-tvars] file, or check the complete list of [Azure specific options][azure-vars] and [the common Tectonic variables][vars].

* `tectonic_azure_ssh_key` - Full path to the public key part of the key added to `ssh-agent` above
* `tectonic_base_domain` - The DNS domain or subdomain delegated to an Azure DNS zone above
* `tectonic_azure_external_dns_zone_id` - Value of `id` in `az network dns zone list` output
* `tectonic_cluster_name` - Usually matches `$CLUSTER` as set above
* `tectonic_license_path` - Full path to `tectonic-license.txt` file downloaded from Tectonic account
* `tectonic_pull_secret_path` - Full path to `config.json` container pull secret file downloaded from Tectonic account

## Deploy the cluster

Check the plan before deploying:

```
$ terraform plan -var-file=build/${CLUSTER}/terraform.tfvars platforms/azure
```

Next, deploy the cluster:

```
$ terraform apply -var-file=build/${CLUSTER}/terraform.tfvars platforms/azure
```

This should run for a short time.

## Access the cluster

The Tectonic Console will be up and running after the containers have downloaded. Access it at the DNS name `https://<tectonic_cluster_name>.<tectonic_base_domain>` (or external DNS values), configured in the `terraform.tfvars` variables file.

### CLI cluster operations with kubectl

Cluster credentials, including any generated CA, are written beneath the `generated/` directory. These credentials allow connections to the cluster with `kubectl`:

```
$ export KUBECONFIG=generated/auth/kubeconfig
$ kubectl cluster-info
```

## Scale an existing cluster on Azure

To scale worker nodes, adjust `tectonic_worker_count` in the cluster build's `terraform.tfvars` file.

Use the `terraform plan` subcommand to check configuration syntax:

```
$ terraform plan \
  -var-file=build/${CLUSTER}/terraform.tfvars \
  -target module.workers \
  platforms/azure
```

Use the `apply` subcommand to deploy the new configuration:

```
$ terraform apply \
  -var-file=build/${CLUSTER}/terraform.tfvars \
  -target module.workers \
  platforms/azure
```

The new nodes should automatically show up in the Tectonic Console shortly after they boot.

## Delete the cluster

Deleting a cluster will remove only the infrastructure elements created by Terraform. For example, an existing DNS resource group is not removed.

To delete the Azure cluster specified in `build/$CLUSTER/terraform.tfvars`, run the following `terraform destroy` command:

```
$ terraform destroy -var-file=build/${CLUSTER}/terraform.tfvars platforms/azure
```

## Under the hood

### Top-level templates

* The top-level templates that invoke the underlying component modules reside `./platforms/azure`
* Point terraform to this location to start applying: `terraform apply ./platforms/azure`

### Etcd nodes

* Etcd cluster nodes are managed by the terraform module `modules/azure/etcd`
* Node VMs are created as an Availability Set (stand-alone instances, deployed across multiple fault domains)
* A load-balancer fronts the etcd nodes to provide a simple discovery mechanism, via a VIP + DNS record.
* Currently, the LB is configured with a public IP address. Future work is planned to convert this to an internal LB.

### Master nodes

* Master node VMs are managed by the templates in `modules/azure/master-as`
* Node VMs are created as an Availability Set (stand-alone instances, deployed across multiple fault domains)
* Master nodes are fronted by one load balancer for the API and one for the Ingress controller.
* The API LB is configured with SourceIP session stickiness, to ensure that TCP (including SSH) sessions from the same client land reliably on the same master node. This allows for provisioning the assets and starting bootkube reliably via SSH.

### Worker nodes

* Worker node VMs are managed by the templates in `modules/azure/worker-as`
* Node VMs are created as an Availability Set (stand-alone instances, deployed across multiple fault domains)
* Worker nodes are not fronted by an LB and don't have public IP addresses. They can be accessed through SSH from any of the master nodes.

## Known issues and workarounds

See the [installer troubleshooting][troubleshooting] document for known problem points and workarounds.


[azure-cli]: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli
[azure-dns]: https://docs.microsoft.com/en-us/azure/dns/dns-overview
[azure-vars]: https://github.com/coreos/tectonic-installer/tree/master/Documentation/variables/azure.md
[bcrypt]: https://github.com/coreos/bcrypt-tool/releases/tag/v1.0.0
[conventions]: ../../conventions.md
[copy-docs]: https://www.terraform.io/docs/commands/apply.html
[domain-delegation]: https://docs.microsoft.com/en-us/azure/dns/dns-delegate-domain-azure-dns
[generic]: ../../generic-platform.md
[install-go]: https://golang.org/doc/install
[login]: https://docs.microsoft.com/en-us/cli/azure/get-started-with-azure-cli
[plan-docs]: https://www.terraform.io/docs/commands/plan.html
[release-notes]: https://coreos.com/tectonic/releases/
[terraform-tvars]: https://github.com/coreos/tectonic-installer/tree/master/Documentation/variables/azure.md
[troubleshooting]: ../../troubleshooting/installer-terraform.md
[vars]: https://github.com/coreos/tectonic-installer/tree/master/Documentation/variables/config.md
[verification-key]: https://coreos.com/security/app-signing-key/
[account-login]: https://account.coreos.com/login
