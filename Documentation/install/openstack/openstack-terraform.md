# Install Tectonic on OpenStack with Terraform

Following this guide will deploy a Tectonic cluster within your OpenStack account.

Generally, the OpenStack platform templates adhere to the standards defined by the project [conventions][conventions] and [generic platform requirements][generic]. This document aims to document the implementation details specific to the OpenStack platform.

<p style="background:#d9edf7; padding: 10px;" class="text-info"><strong>Pre-Alpha:</strong> These modules and instructions are currently considered pre-alpha. See the <a href="../../platform-lifecycle.md">platform life cycle</a> for more details.</p>

## Prerequsities

* **CoreOS Tectonic account**: A [CoreOS Tectonic account][account-login]. You must provide the account's License and Pull Secret during installation.
* **Terraform**: Tectonic Installer includes and requires a specific version of Terraform. This is included in the Tectonic Installer tarball. See the [Tectonic Installer release notes][release-notes] for information about which Terraform versions are compatible.
* **CoreOS Container Linux**: The latest Container Linux Beta (1353.2.0 or later) [uploaded into Glance](https://coreos.com/os/docs/latest/booting-on-openstack.html) and its OpenStack image ID.

## Getting Started
OpenStack is a highly customizable environment where different components can be enabled/disabled. The installer currently supports only one flavor:

- `neutron`: A private Neutron network is being created with master/worker nodes exposed via floating IPs connected to an etcd instance via an internal network.

Replace `<flavor>` with either option in the following commands. Now we're ready to specify our cluster configuration.

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

We need to add the `terraform` binary to our `PATH`. The platform should be `darwin` or `linux`.

```bash
$ export PATH=$(pwd)/tectonic-installer/linux:$PATH # Put the `terraform` binary in our PATH
```

Download the Tectonic Terraform modules.

```bash
$ terraform init platforms/openstack/<flavor>
Downloading modules...
Get: modules/tls/kube/self-signed
Get: modules/tls/etcd
Get: modules/tls/ingress/self-signed
Get: modules/tls/identity/self-signed
Get: modules/bootkube
Get: modules/tectonic
Get: modules/openstack/etcd
Get: modules/ignition
Get: modules/openstack/nodes
Get: modules/ignition
Get: modules/openstack/nodes
Get: modules/openstack/secrets
Get: modules/openstack/secgroups
Get: modules/dns/designate
Get: modules/net/flannel-vxlan
Get: modules/net/calico-network-policy
Get: modules/openstack/secgroups/rules/default
Get: modules/openstack/secgroups/rules/k8s
Get: modules/openstack/secgroups/rules/k8s_nodes
Get: modules/openstack/secgroups/rules/etcd

Initializing provider plugins...
   Checking for available provider plugins on https://releases.hashicorp.com...
```

Terraform will download any available plugins, and report when initialization is complete.

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
$ terraform plan -var-file=build/${CLUSTER}/terraform.tfvars platforms/openstack/<flavor>
```

Next, deploy the cluster:

```
$ terraform apply -var-file=build/${CLUSTER}/terraform.tfvars platforms/openstack/<flavor>
```

This should run for a little bit, and when complete, your Tectonic cluster should be ready.

If you encounter any issues, check the known issues and workarounds below.

## Access the cluster

The Tectonic Console should be up and running after the containers have downloaded. Access it at the DNS name `https://<tectonic_cluster_name>.<tectonic_base_domain>`, configured in the `terraform.tfvars` variables file.

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
