# Install Tectonic on OpenStack with Terraform

Following this guide will deploy a Tectonic cluster within your OpenStack account.

Generally, the OpenStack platform templates adhere to the standards defined by the project [conventions][conventions] and [generic platform requirements][generic]. This document aims to document the implementation details specific to the OpenStack platform.

<p style="background:#d9edf7; padding: 10px;" class="text-info"><strong>Pre-Alpha:</strong> These modules and instructions are currently considered pre-alpha. See the <a href="../../platform-lifecycle.md">platform life cycle</a> for more details.</p>

## Prerequsities

* **Terraform**: Tectonic Installer includes and requires a specific version of Terraform. This is included in the Tectonic Installer tarball. See the [Tectonic Installer release notes][release-notes] for information about which Terraform versions are compatible.
* **CoreOS Container Linux**: The latest Container Linux Beta (1353.2.0 or later) [uploaded into Glance](https://coreos.com/os/docs/latest/booting-on-openstack.html) and its OpenStack image ID.
* **CoreOS Tectonic account**: A [CoreOS Tectonic account][account-login]. You must provide the account's License and Pull Secret during installation.

## Getting Started
OpenStack is a highly customizable environment where different components can be enabled/disabled. The installer currently supports only one flavor:

- `neutron`: A private Neutron network is being created with master/worker nodes exposed via floating IPs connected to an etcd instance via an internal network.

Replace `<flavor>` with either option in the following commands. Now we're ready to specify our cluster configuration.

### Sign up for a CoreOS account

First, sign up for a CoreOS account, which provides up to 10 free nodes of production quality Tectonic. Once completed, log in to the account to obtain the License and Pull Secret required for installation.

1. Go to [https://account.coreos.com/login][account-login].
2. Click *Sign Up* and create an account using either your Google account or another email address.
3. Enter your contact information, and click *Get License* for 10 nodes.
4. Agree to the license terms.

Check your inbox for a confirmation email. Once confirmed, log in to display the account's *Overview* page. Click "Free for use for up to 10 nodes" under Tectonic, and add your contact information. Once the update has processed, the *Overview* window will refresh to display the License and Pull Secret required for installation.

### Download and extract Tectonic Installer

Open a new terminal and run the following command to download Tectonic Installer.

```bash
$ curl -O https://releases.tectonic.com/releases/tectonic_1.7.3-tectonic.3.tar.gz # download
```

Verify the release has been signed by the [CoreOS App Signing Key][verification-key].

```bash
$ gpg2 --keyserver pgp.mit.edu --recv-key 18AD5014C99EF7E3BA5F6CE950BDD3E0FC8A365E
$ gpg2 --verify tectonic_1.7.3-tectonic.3-tar-gz.asc tectonic_1.7.3-tectonic.3-tar.gz
# gpg2: Good signature from "CoreOS Application Signing Key <security@coreos.com>"
```

Extract the tarball and navigate to the `tectonic` directory.

```bash
$ tar xzvf tectonic_1.7.3-tectonic.3.tar.gz
$ cd tectonic
```

### Initialize and configure Terraform

We need to add the `terraform` binary to our `PATH`. The platform should be `darwin` or `linux`.

```bash
$ export PATH=$PATH:$(pwd)/tectonic-installer/linux # Put the `terraform` binary in our PATH
```

Download the Tectonic Terraform modules.

```bash
$ terraform init platforms/openstack/<flavor>
```

Configure your AWS credentials for setting up Route 53 DNS record entries. See the [AWS docs][env] for details.

```
$ export AWS_ACCESS_KEY_ID=
$ export AWS_SECRET_ACCESS_KEY=
```

Set your desired region:

```
$ export AWS_REGION=
```

Configure your OpenStack credentials.

```
$ export OS_TENANT_NAME=
$ export OS_USERNAME=
$ export OS_PASSWORD=
$ export OS_AUTH_URL=
$ export OS_REGION_NAME=
```

## Customize the deployment

Customizations to the base installation live in `examples/terraform.tfvars.<flavor>`. Export a variable that will be your cluster identifier:

```
$ export CLUSTER=my-cluster
```

Create a build directory to hold your customizations and copy the example file into it:

```
$ mkdir -p build/${CLUSTER}
$ cp examples/terraform.tfvars.openstack-neutron build/${CLUSTER}/terraform.tfvars
```

Edit the parameters with your OpenStack details. View all of the [OpenStack Neutron][openstack-neutron-vars] specific options and [the common Tectonic variables][vars].

## Deploy the cluster

Test out the plan before deploying everything:

```
$ terraform plan -var-file=build/${CLUSTER}/terraform.tfvars platforms/openstack/<flavor>
```

Next, deploy the cluster:

```
$ terraform apply -var-file=build/${CLUSTER}/terraform.tfvars platforms/openstack/<flavor>
```

This should run for a little bit, and when complete, your Tectonic cluster should be ready.

If you encounter any issues, check the known issues and workarounds below.

## Access the cluster

The Tectonic Console should be up and running after the containers have downloaded. You can access it at the DNS name configured in your variables file.

Inside of the `/generated` folder you should find any credentials, including the CA if generated, and a kubeconfig. You can use this to control the cluster with `kubectl`:

```
$ KUBECONFIG=generated/auth/kubeconfig
$ kubectl cluster-info
```

## Scale the cluster

To scale worker nodes, adjust `tectonic_worker_count` in `terraform.tfvars`.

Use the `plan` command to check your syntax:

```
$ terraform plan \
  -var-file=build/${CLUSTER}/terraform.tfvars \
  -target module.workers \
  platforms/openstack/<flavor>
```

Once you are ready to make the changes live, use `apply`:

```
$ terraform apply \
  -var-file=build/${CLUSTER}/terraform.tfvars \
  -target module.workers \
  platforms/openstack/<flavor>
```

The new nodes should automatically show up in the Tectonic Console shortly after they boot.

## Delete the cluster

Deleting your cluster will remove only the infrastructure elements created by Terraform. If you selected an existing VPC and subnets, these items are not touched. To delete, run:

```
$ terraform destroy -var-file=build/${CLUSTER}/terraform.tfvars platforms/openstack/<flavor>
```

### Known issues and workarounds

If you experience pod-to-pod networking issues, try lowering the MTU setting of the CNI bridge.
Change the contents of `modules/bootkube/resources/manifests/kube-flannel.yaml` and configure the following settings:

```
apiVersion: v1
kind: ConfigMap
metadata:
  name: kube-flannel-cfg
  namespace: kube-system
  labels:
    tier: node
    k8s-app: flannel
data:
  cni-conf.json: |
    {
      "name": "cbr0",
      "type": "flannel",
      "delegate": {
        "mtu": 1400,
        "isDefaultGateway": true
      }
    }
  net-conf.json: |
    {
      "Network": "${cluster_cidr}",
      "Backend": {
        "Type": "vxlan",
        "Port": 4789
      }
    }
```

Setting the IANA standard port `4789` can help debugging when using `tcpdump -vv -i eth0` on the worker/master nodes as encapsulated VXLAN packets will be shown.

See the [troubleshooting][troubleshooting] document for workarounds for bugs that are being tracked.

[conventions]: ../../conventions.md
[generic]: ../../generic-platform.md
[env]: http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#cli-environment
[vars]: https://github.com/coreos/tectonic-installer/tree/master/Documentation/variables/config.md
[troubleshooting]: ../../troubleshooting/faq.md
[openstack-neutron-vars]: https://github.com/coreos/tectonic-installer/tree/master/Documentation/variables/openstack-neutron.md
[release-notes]: https://coreos.com/tectonic/releases/
[verification-key]: https://coreos.com/security/app-signing-key/
[account-login]: https://account.coreos.com/login
