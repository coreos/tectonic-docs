# Install Tectonic on bare metal with Terraform

Use this guide to deploy a Tectonic cluster on virtual or physical hardware using the command line and Terraform.

## Prerequisites

For a complete list of requirements, see [Bare Metal Installation requirements][bare-requirements].

* [Tectonic Account][account].
* The Terraform version included in the Tectonic Installer tarball. See the [Tectonic Installer release notes][release-notes] for information about which Terraform versions are compatible.
* [Matchbox v0.6+][matchbox-latest] installation with TLS client credentials and the gRPC API enabled.
* [PXE network boot environment][network-setup] with DHCP, TFTP, and DNS services.
* [DNS][dns] records for the Kubernetes controller(s) and Tectonic Ingress worker(s).
* Machines with BIOS options set to boot from disk normally, but PXE prior to installation.
* Machines with known MAC addresses and stable domain names.
* A SSH keypair whose private key is present in your system's [ssh-agent][ssh-agent].

## Getting Started

### Download and extract Tectonic Installer

Open a new terminal and run the following command to download Tectonic Installer.

```bash
$ curl -O https://releases.tectonic.com/releases/tectonic_1.7.3-tectonic.3.tar.gz # download
```

Verify the release has been signed by the [CoreOS App Signing Key][verification-key].

```bash
$ gpg2 --keyserver pgp.mit.edu --recv-key 18AD5014C99EF7E3BA5F6CE950BDD3E0FC8A365E
$ gpg2 --verify tectonic-1.7.3-tectonic.2-tar-gz.asc tectonic-1.7.3-tectonic.2-tar.gz
# gpg2: Good signature from "CoreOS Application Signing Key <security@coreos.com>"
```

Extract the tarball and navigate to the `tectonic` directory.

```bash
$ tar xzvf tectonic-1.7.3-tectonic.2.tar.gz
$ cd tectonic
```

### Initialize and configure Terraform

Start by setting the `INSTALLER_PATH` to the location of your platform's Tectonic installer. The platform should be `linux` or `darwin`.

```bash
$ export INSTALLER_PATH=$(pwd)/tectonic-installer/linux/installer
$ export PATH=$PATH:$(pwd)/tectonic-installer/linux
```

Next, get the modules that Terraform will use to create the cluster resources:

```bash
$ terraform init ./platforms/metal
```

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

The Tectonic Console should be up and running after the containers have downloaded. You can access it at the DNS name configured in your variables file.

Inside of the `/generated` folder you should find any credentials, including the CA if generated, and a kubeconfig. You can use this to control the cluster with `kubectl`:

```
$ export KUBECONFIG=generated/auth/kubeconfig
$ kubectl cluster-info
```

## Work with the cluster

For more information on working with installed clusters, see [Scaling Tectonic bare metal clusters][scale-metal], and [Uninstalling Tectonic][uninstall].


[conventions]: ../../conventions.md
[generic]: ../../generic-platform.md
[account]: https://account.coreos.com
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
