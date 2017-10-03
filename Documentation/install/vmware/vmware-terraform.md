# Install Tectonic on VMware with Terraform

Following this guide will deploy a Tectonic cluster within a VMware vSphere infrastructure .

Generally, the VMware platform templates adhere to the standards defined by the project [conventions][conventions] and [generic platform requirements][generic]. This document aims to document the implementation details specific to the VMware platform.

<p style="background:#d9edf7; padding: 10px;" class="text-info"><strong>Pre-Alpha:</strong> These modules and instructions are currently considered pre-alpha. See the <a href="../../platform-lifecycle.md">platform life cycle</a> for more details.</p>

## Prerequsities

1. Download the latest Container Linux Stable OVA from  [https://coreos.com/os/docs/latest/booting-on-vmware.html][boot-vm].
2. Import `coreos_production_vmware_ova.ova` into vCenter. Most settings may be left at their default. Consider "thin" provisioning and naming the template with the CoreOS Container Linux version number.
3. Resize the virtual machine disk size to 30GB or larger.
4. In the *Virtual Machine Configuration* view select "vApp Options" tab and un-check "Enable vApp Options".
5. Convert the Container Linux image into a Virtual Machine template.
6. Pre-allocate IP addresses for the cluster and create DNS records.
7. For production clusters, configure an existing Load Balancer for Tectonic. For an example setup, see [Using F5 BIG-IP LTM with Tectonic][using-f5].

### DNS and IP address allocation

Create required DNS records before beginning setup. The following table lists 3 etcd nodes, 2 master nodes and 2 worker nodes.


| Record | Type | Value |
|------|-------------|:-----:|
|mycluster.mycompany.com | A | 192.168.246.30 |
|mycluster.mycompany.com | A | 192.168.246.31 |
|mycluster-k8s.mycompany.com | A | 192.168.246.20 |
|mycluster-k8s.mycompany.com | A | 192.168.246.21 |
|mycluster-worker-0.mycompany.com | A | 192.168.246.30 |
|mycluster-worker-1.mycompany.com | A | 192.168.246.31 |
|mycluster-master-0.mycompany.com | A | 192.168.246.20 |
|mycluster-master-1.mycompany.com | A | 192.168.246.21 |
|mycluster-etcd-0.mycompany.com | A | 192.168.246.10 |
|mycluster-etcd-1.mycompany.com | A | 192.168.246.11 |
|mycluster-etcd-2.mycompany.com | A | 192.168.246.12 |

See [Tectonic on Baremetal DNS documentation][baremetaldns] for general DNS Requirements.

### Tectonic Account

Register for a [Tectonic Account][register], which is free for up to 10 nodes. You must provide the cluster license and pull secret during installation.

### ssh-agent

Ensure `ssh-agent` is running:
```
$ eval $(ssh-agent)
```

Add the SSH key that will be used for the Tectonic installation to `ssh-agent`:
```
$ ssh-add <path-to-ssh-key>
```

Verify that the SSH key identity is available to the ssh-agent:
```
$ ssh-add -L
```

Reference the absolute path of the **_public_** component of the SSH key in `tectonic_vmware_ssh_authorized_key`.

Without this, terraform is not able to SSH copy the assets and start bootkube.
Ensure the SSH known_hosts file does not contain old records for the API DNS name to avoid a key fingerprint mismatch.

## Getting Started

The following steps must be executed on a machine that has network connectivity to VMware vCenter API and SSH access to Tectonic Master Server(s).

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

## Customize the deployment

Customizations to the base installation live in `examples/terraform.tfvars.<flavor>`. Export a variable that will be the cluster identifier:

```
$ export CLUSTER=my-cluster
```

Create a build directory to hold customizations and copy the example file into it:

```
$ mkdir -p build/${CLUSTER}
$ cp examples/terraform.tfvars.vmware build/${CLUSTER}/terraform.tfvars
$ cd build/${CLUSTER}/
```

Edit the parameters with details of the VMware infrastructure. View all of the [VMware][vmware] specific options and [the common Tectonic variables][vars].

## Deploy the cluster

Get the modules that Terraform will use to create the cluster resources:

```
$ terraform get ../../platforms/vmware
```

Test out the plan before deploying everything:

```
$ terraform plan ../../platforms/vmware
```

Terraform will prompt for vSphere credentials:

```
provider.vsphere.password
  The user password for vSphere API operations.

  Enter a value:

provider.vsphere.user
  The user name for vSphere API operations.

  Enter a value:
```

Next, deploy the cluster:

```
$ terraform apply ../../platforms/vmware
```

Wait for `terraform apply` to complete all tasks. The Tectonic cluster should be ready upon completion of the `apply` command. See the [troubleshooting][troubleshooting] guide if problems occur.

## Access the cluster

Tectonic Console will be up and running after the containers have downloaded. Console can be accessed by the DNS name configured as `tectonic_vmware_ingress_domain` in the `terraform.tfvars` variables file.

Credentials and secrets for Tectonic can be found in the `/generated` folder, including the CA (if generated) and a kubeconfig. Use the kubeconfig file to control the cluster with `kubectl`:

```
$ export KUBECONFIG=generated/auth/kubeconfig
$ kubectl cluster-info
```

## Scaling Tectonic VMware clusters

Both master and worker nodes may be scaled on VMware using terraform.

### Scaling worker nodes

To scale worker nodes, adjust `tectonic_worker_count`, `tectonic_vmware_worker_hostnames` and `tectonic_vmware_worker_ip` variables in `terraform.tfvars` and run:

```
$ terraform plan \
  ../../platforms/vmware
$ terraform apply \
  ../../platforms/vmware
```
After running `terraform apply` new worker machines will appear in Tectonic Console. This change may take several minutes.

### Scaling master nodes

To scale master nodes, adjust `tectonic_master_count`, `tectonic_vmware_master_hostnames` and `tectonic_vmware_master_ip` variables in `terraform.tfvars` and run:

```
$ terraform plan \
  ../../platforms/vmware
$ terraform apply \
  ../../platforms/vmware
```
After running `terraform apply` master machines will appear in Tectonic Console. This change may take several minutes.  

Add the new controller nodes' IP address to the DNS record for the name set in the tectonic_vmware_controller_domain variable, or update the load balancer configuration with the new controller nodes.

## Known issues and workarounds

See the [troubleshooting][troubleshooting] document for known issues and workarounds.

## Delete the cluster

To delete Tectonic cluster, run:

```
$ terraform destroy ../../platforms/vmware
```

[boot-vm]: https://coreos.com/os/docs/latest/booting-on-vmware.html
[register]: https://account.coreos.com
[baremetaldns]: https://coreos.com/tectonic/docs/latest/install/bare-metal/#dns
[conventions]: ../../conventions.md
[generic]: ../../generic-platform.md
[downloadterraform]: https://www.terraform.io/downloads.html
[vmware]: https://github.com/coreos/tectonic-installer/tree/master/Documentation/variables/vmware.md
[vars]: https://github.com/coreos/tectonic-installer/tree/master/Documentation/variables/config.md
[troubleshooting]: ../../troubleshooting/faq.md
[using-f5]: ../../reference/f5-ltm-lb.md
[verification-key]: https://coreos.com/security/app-signing-key/
