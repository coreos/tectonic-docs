# Install Tectonic on bare metal with Terraform

Use this guide to deploy a Tectonic cluster on virtual or physical hardware using the command line and Terraform.

## Prerequisites

For a complete list of requirements, see [Bare Metal Installation requirements][bare-requirements].

* [CoreOS Tectonic Account][account-login], with access to its License and Pull Secret.
* The Terraform version included in the Tectonic Installer tarball. See the [Tectonic Installer release notes][release-notes] for information about which Terraform versions are compatible.
* [Matchbox v0.6+][matchbox-latest] installation with TLS client credentials and the gRPC API enabled.
* [PXE network boot environment][network-setup] with DHCP, TFTP, and DNS services.
* [DNS][dns] records for the Kubernetes controller(s) and Tectonic Ingress worker(s).
* Machines with BIOS options set to boot from disk normally, but PXE prior to installation.
* Machines with known MAC addresses and stable domain names.
* A SSH keypair whose private key is present in your system's [ssh-agent][ssh-agent].

## Getting Started

### Create a CoreOS account

Tectonic Installer requires the License and Pull Secret provided with a CoreOS account. To obtain this information and up to 10 free nodes, create a CoreOS account.

1. Go to [https://account.coreos.com/login][account-login], and click *Sign Up*.

2. Check your inbox for a confirmation email. Click through to accept the terms of the license, activate your account, and be redirected to the *Account Overview* page.

3. Click "Free for use up to 10 nodes" under Tectonic. Enter your contact information, and click *Get License for 10 nodes*.

Once the update has processed, the *Overview* window will refresh to include links to download the License and Pull Secret.

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

Start by setting the `INSTALLER_PATH` to the location of your platform's Tectonic installer. The platform should be `linux` or `darwin`.

```bash
$ export INSTALLER_PATH=$(pwd)/tectonic-installer/linux/installer
$ export PATH=$(pwd)/tectonic-installer/linux:$PATH
```

Next, get the modules that Terraform will use to create the cluster resources:

```bash
$ terraform init ./platforms/metal
Downloading modules...
Get: modules/ignition
Get: modules/ignition
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

Now, specify the cluster configuration.

## Customize the deployment

Create a build directory to hold your customizations and copy the example file into it:

```
$ export CLUSTER=my-cluster
$ mkdir -p build/${CLUSTER}
$ cp examples/terraform.tfvars.metal build/${CLUSTER}/terraform.tfvars
```

Customizations should be made to `build/${CLUSTER}/terraform.tfvars`. Edit the following variables to correspond to your matchbox installation:

* `tectonic_matchbox_http_url`
* `tectonic_matchbox_rpc_endpoint`
* `tectonic_matchbox_client_cert`
* `tectonic_matchbox_client_key`
* `tectonic_matchbox_ca`

Edit additional variables to specify DNS records, list machines, and set an SSH key and Tectonic Console email and password.

Several variables are currently required, but their values are not used.

* `tectonic_are_domain`
* `tectonic_master_count`
* `tectonic_worker_count`
* `tectonic_etcd_count`

### Set Console login secrets

Set these sensitive values in the environment. The `tectonic_admin_password` will be encrypted before storage or transport:

* `TF_VAR_tectonic_admin_email` - String giving the email address used as user name for the initial Console login
* `TF_VAR_tectonic_admin_password` - Plaintext password string for initial Console login

For example, in the `bash(1)` shell, replace the quoted values with those for the cluster being deployed and run the following commands:

```bash
$ export TF_VAR_tectonic_admin_email="admin@example.com"
$ export TF_VAR_tectonic_admin_password="pl41nT3xt"
```

## Deploy the cluster

Test out the plan before deploying everything:

```
$ terraform plan -var-file=build/${CLUSTER}/terraform.tfvars platforms/metal
```

Next, deploy the cluster:

```
$ terraform apply -var-file=build/${CLUSTER}/terraform.tfvars platforms/metal
```

This will write machine profiles and matcher groups to the matchbox service.

## Power On

Power on the machines with `ipmitool` or `virt-install`. Machines will PXE boot, install Container Linux to disk, and reboot.

```
ipmitool -H node1.example.com -U USER -P PASS power off
ipmitool -H node1.example.com -U USER -P PASS chassis bootdev pxe
ipmitool -H node1.example.com -U USER -P PASS power on
```

Terraform will wait for the disk installation and reboot to complete and then be able to copy credentials to the nodes to bootstrap the cluster. You may see `null_resource.kubeconfig.X: Still creating...` during this time.

Run `terraform apply` until all tasks complete. Your Tectonic cluster should be ready. If you encounter any issues, check the [Tectonic troubleshooting guides][troubleshooting].

## Access the cluster

The Tectonic Console should be up and running after the containers have downloaded. Access it at the DNS name `https://<tectonic_cluster_name>.<tectonic_base_domain>`, configured in the `terraform.tfvars` variables file.


Inside of the `/generated` folder you should find any credentials, including the CA if generated, and a kubeconfig. You can use this to control the cluster with `kubectl`:

```
$ export KUBECONFIG=generated/auth/kubeconfig
$ kubectl cluster-info
```

## Work with the cluster

For more information on working with installed clusters, see [Scaling Tectonic bare metal clusters][scale-metal], and [Uninstalling Tectonic][uninstall].


[conventions]: ../../conventions.md
[generic]: ../../generic-platform.md
[account-login]: https://account.coreos.com/login
[vars]: https://github.com/coreos/tectonic-installer/tree/master/Documentation/variables/config.md
[troubleshooting]: ../../troubleshooting/faq.md
[uninstall]: uninstall.md
[scale-metal]: ../../admin/bare-metal-scale.md
[release-notes]: https://coreos.com/tectonic/releases/
[ssh-agent]: requirements.md#ssh-agent
[bare-requirements]: requirements.md
[network-setup]: https://coreos.com/matchbox/docs/latest/network-setup.html
[matchbox-latest]: https://coreos.com/matchbox/docs/latest/
[dns]: index.md#dns
[verification-key]: https://coreos.com/security/app-signing-key/
