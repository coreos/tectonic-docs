# Installing Tectonic on Microsoft Azure

This guide steps through installing a Tectonic enterprise-ready Kubernetes cluster on Microsoft Azure. The resulting basic cluster has 1 controller and 3 worker nodes and is ready for application deployment, monitoring, and scaling.

## Prerequisites

### CoreOS account

First, sign up for a CoreOS account, which provides up to 10 free nodes of production quality Tectonic. Once completed, log in to the account to obtain the License and Pull Secret required for installation.

1. Go to [https://account.coreos.com/login][account-login].
2. Click *Sign Up* and create an account using either your Google account or another email address.
3. Enter your contact information, and click *Get License* for 10 nodes.
4. Agree to the license terms.

Check your inbox for a confirmation email. Once confirmed, log in to display the account's *Overview* page. Click "Free for use for up to 10 nodes" under Tectonic, and add your contact information. Once the update has processed, the *Overview* window will refresh to display the License and Pull Secret required for installation.

### Azure account

Before beginning, create an [Azure account][azure-home] with a valid credit card.

### Azure CLI tool

The [`az` command line API tool][az] will be used to retrieve and generate credentials granting access to the Installer:

```sh
$ curl -L https://aka.ms/InstallAzureCli | bash
```

### Command line comfort

This guide assumes intermediate comfort with the Unix command line, including such tasks as setting environment variables and configuring `ssh-agent` with a key. The Tectonic installation process on Azure happens in the terminal.

### Domain name

This guide requires a domain name available for configuration, and the ability to follow steps to delegate it or a subdomain to Azure DNS name service.

## Configuring Azure DNS

The Azure DNS service allows you to perform DNS management, traffic management, availability monitoring and domain registration. DNS management is the only feature of Azure DNS required to install Tectonic.

### Create an Azure DNS Zone

When creating an Azure DNS Zone, enter a domain or subdomain that you own and can manage.

The Tectonic installation requires a domain or subdomain name in which it will create two subdomains: one for the Tectonic console, and one for the Kubernetes API server. This allows Tectonic to access and use the listed domain. This tutorial employs the domain name `example.com`. The string `example.com` should be replaced with the domain or subdomain name configured in this step wherever it appears later in the tutorial.

1. From the menu at left, selet *DNS Zones*
2. Click *Add* to create a new zone.
3. Enter an existing, registered domain or subdomain name.
4. Click *Create*.

Azure provides 4 DNS nameservers for the new zone. The domain or sub-domain must be [configured to use these nameservers][azure-dns-delegate]. Visit the domain registrar to add the Azure NS records.

1. Go to the domain registrar’s website.
2. Go to the DNS settings page and enter the four Azure nameservers as NS records for the domain or subdomain.
3. Save the updated domain settings.

Note that it may take from a few minutes to several hours for the changes to take effect, depending on the TTL setting of existing NS records.

To verify which nameservers are associated with your domain, use a tool like `dig` or `nslookup`. If no nameservers are returned when you look up your domain, changes may still be pending. Here's an example command:

```bash
$ dig -t ns [example.com]
```

The nameservers are set up correctly when the lookup yields the four hostnames provided by Azure.

## Generating Azure authentication assets

Execute `az login` to obtain an authentication token. See the [Azure CLI docs][azlogin] for more information. Once logged in, note the `id` value of the output from the `az login` command. This is a simple way to retrieve the Subscription ID for the Azure account.

### Add Active Directory Service Principal role assignment

Next, add a new Active Directory (AD) Service Principal (SP) role assignment to grant Installer access to Azure:

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

Export the following environment variables with values obtained from the output of the SP role assignment. As noted above, `ARM_SUBSCRIPTION_ID` is the `id` of the Azure account returned by `az login`.

```
# The id value in az login output
$ export ARM_SUBSCRIPTION_ID=azure-acct-sub-id
# The appID value in az ad output
$ export ARM_CLIENT_ID=generated-app-id
# The password value in az ad output
$ export ARM_CLIENT_SECRET=generated-pass
# The tenant value in az ad output
$ export ARM_TENANT_ID=generated-tenant
```

## Add a key to ssh-agent

The next step in preparing the environment for installation is to add the key to be used for logging in to each cluster node during initialization to the local `ssh-agent`.

### ssh-agent

Ensure `ssh-agent` is running by listing the known keys:

```bash
$ ssh-add -L
```

Add the SSH private key that will be used for the Tectonic installation to `ssh-agent`:

```bash
$ ssh-add ~/.ssh/id_rsa
```

Verify that the SSH key identity is available to the ssh-agent:

```bash
$ ssh-add -L
```

## Tectonic Installer

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
$ cd tectonic_1.7.9-tectonic.1
```

### Create a cluster build directory

Choose a cluster name to identify the cluster. Export an environment variable with the chosen cluster name. This tutorial names the cluster `my-cluster`.

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

Edit the parameters in `build/$CLUSTER/terraform.tfvars` with the deployment's Azure details, domain name, license, and pull secret. See the details of each value in the cluster's `terraform.tfvars`  file, or check the complete list of [Azure specific options][azure-vars] and [the common Tectonic variables][vars].

* `tectonic_azure_ssh_key` - Full path to the public key part of the key added to `ssh-agent` above
* `tectonic_base_domain` - The DNS domain or subdomain delegated to an Azure DNS zone above
* `tectonic_azure_external_dns_zone_id` - Value of `id` in `az network dns zone list` output
* `tectonic_cluster_name` - Usually matches `$CLUSTER` as set above
* `tectonic_license_path` - Full path to `tectonic-license.txt` file downloaded from Tectonic account
* `tectonic_pull_secret_path` - Full path to `config.json` container pull secret file downloaded from Tectonic account

## Deploy the cluster

First, initialize Terraform:

```
$ terraform init platforms/azure
```

Then, validate the plan before deploying:

```
$ terraform plan -var-file=build/${CLUSTER}/terraform.tfvars platforms/azure
```

Deploy the cluster – aka `apply`:

```
$ terraform apply -var-file=build/${CLUSTER}/terraform.tfvars platforms/azure
```

The apply step will run for some time and prints status on the standard output.

## Access the cluster

When `terraform apply` is complete, the Tectonic console will be available at `https://my-cluster.example.com`, as configured in the cluster build's variables file.

[**NEXT:** Deploying an application on Tectonic][first-app]


[az]: https://docs.microsoft.com/en-us/cli/azure/install-azure-cli
[azlogin]: https://docs.microsoft.com/en-us/azure/xplat-cli-connect
[azure-dns-delegate]: https://docs.microsoft.com/en-us/azure/dns/dns-delegate-domain-azure-dns
[azure-home]: https://azure.microsoft.com/
[azure-vars]: https://github.com/coreos/tectonic-installer/tree/master/Documentation/variables/azure.md
[bcrypt-tool]: https://github.com/coreos/bcrypt-tool/releases
[first-app]: first-app.md
[account-login]: https://account.coreos.com/login
[vars]: https://github.com/coreos/tectonic-installer/tree/master/Documentation/variables/config.md
[verification-key]: https://coreos.com/security/app-signing-key/
