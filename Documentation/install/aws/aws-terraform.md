# Install Tectonic on AWS with Terraform

Use this guide to manually install a Tectonic cluster on an AWS account. To install Tectonic on AWS with a graphical installer instead, refer to the [AWS graphical installer documentation][aws-gui].

## Prerequsities

* **CoreOS Account**: Register for a [CoreOS Account][account-login], which provides free access for up to 10 nodes on Tectonic. You must provide the account's License and Pull Secret (available from the account Overview page) during installation.
* **Terraform**: Tectonic Installer includes and requires a specific version of Terraform. This is included in the Tectonic Installer tarball. See the [Tectonic Installer release notes][release-notes] for information about which Terraform versions are compatible.
* **DNS**: Ensure that the DNS zone is already created and available in Route 53 for the account. For example if the `tectonic_base_domain` is set to `kube.example.com` a Route 53 zone must exist for this domain and the AWS nameservers must be configured for the domain.


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
$ curl -O https://releases.tectonic.com/releases/tectonic_1.8.4-tectonic.3.zip
$ curl -O https://releases.tectonic.com/releases/tectonic_1.8.4-tectonic.3.zip.sig
```

Verify the release has been signed by the [CoreOS App Signing Key][verification-key].

```bash
$ gpg2 --keyserver pgp.mit.edu --recv-key 18AD5014C99EF7E3BA5F6CE950BDD3E0FC8A365E
$ gpg2 --verify tectonic_1.8.4-tectonic.3.zip.sig tectonic_1.8.4-tectonic.3.zip
# gpg2: Good signature from "CoreOS Application Signing Key <security@coreos.com>"
```

Unzip Tectonic Installer and navigate to the `tectonic` directory.

```bash
$ unzip tectonic_1.8.4-tectonic.3.zip
$ cd tectonic_1.8.4-tectonic.3
```

Add the installer to your path:

```bash
$ export PATH=$(pwd)/tectonic-installer/tectonic:$PATH
```

## Configure cloud credentials

Configure your AWS credentials. See the [AWS docs][env] for details.

```bash
$ export AWS_ACCESS_KEY_ID=
$ export AWS_SECRET_ACCESS_KEY=
```

Set your desired region:

```bash
$ export AWS_REGION=
```

## Create a new cluster

Tectonic is installed with a CLI tool called `tectonic`. Under the hood, it installs clusters by executing two different steps. First, a set of manifests that represents your cluster are generated, based on your configuration file. Second, those generated manifests are used to talk to AWS APIs to start the cluster infrastructure. The shorthand `install`, which we'll use below, executes these steps one after the other.

First, create a new cluster workspace and give it a name:

```bash
$ tectonic cluster new example --template=aws
Created cluster "example"
Created configuration clusters/example/tectonic-install-config.yaml from "aws" template
```

Inside of the clusters directory, there is a new folder called `example` with a minimal AWS configuration.

## Customize the deployment

Open `tectonic-install-config.yaml` and you should a minimal set of AWS configuration options:

```yaml
Version: 1.0
Name: example
Platform: aws
DNS:
  BaseDomain: tectonic.example.com
Nodes:
  Etcd:
    Count: 3
    aws:
      instanceType: m3.medium
      storageType: ssd
      storageSize: 30gb
  Masters:
    count: 2
    aws:
      instanceType: m3.medium
      storageType: ssd
      storageSize: 30gb
  Workers:
    count: 5
    aws:
      instanceType: m3.medium
      storageType: ssd
      storageSize: 30gb
Licensing:
  LicensePath: /path/to/license
  PullSecretPath: /path/to/pullsecret
Credentials:
  AdminEmail: admin@example.com
  AdminPasswordHash: abc123abc123
```

### Set DNS options

By default, Tectonic uses AWS Route53 for DNS. Configure the `BaseDomain` value with a domain that is already configured in Route 53. [Other DNS options (DOCS NEEDED)][other-dns] are also available.

### Set Node options

Tectonic has three different types of Nodes that make up the cluster. All will be provisioned automatically in AWS based on your desired `count` and instance parameters.

| Type | Count | Description |
|:-----|:------|:------------|
| etcd | 1-9 <br/> 3 (default) | Nodes dedicated to running etcd, the multi-master brains of your cluster. |
| masters | At least 1 <br/> 2 (default) | Nodes dedicated to running the cluster's control plane. At least two allows for high availability. API throughput is scaled here. |
| worker | At least 1 <br/> 2 (default) | These nodes run your workloads. Tectonic will automatically spread out your apps over the available set of Nodes and automatically handle failover. |

**COMING IN FUTURE**
* Ability to set ignition profiles per node
* Ability to label these nodes

### Set Console login secrets

**THESE NEED TO BE UPDATED**

Set these sensitive values in the environment. The `tectonic_admin_password` will be encrypted before storage or transport:

* `TF_VAR_tectonic_admin_email` - String giving the email address used as user name for the initial Console login
* `TF_VAR_tectonic_admin_password` - Plaintext password string for initial Console login

For example, in the `bash(1)` shell, replace the quoted values with those for the cluster being deployed and run the following commands:

```bash
$ export TF_VAR_tectonic_admin_email="admin@example.com"
$ export TF_VAR_tectonic_admin_password="pl41nT3xt"
```

## Deploy the cluster

The cluster is now ready to be installed. Your configuration values will be checked prior to creating the infrastructure.

This will run for a little bit. When complete, your Tectonic cluster will be ready.

```bash
$ tectonic install
Cluster workspace is "example"
Checking cluster configuration from clusters/example/tectonic-install-config.yaml
Generating manifests
  Operator manifests
  Cluster certificate authority
  TLS certificates for etcd
  TLS certificates for control plane
  ...
Creating infrastructure
  Running terraform init
  Running terraform plan
  ...
Successfully booted Tectonic cluster "example". Access your cluster at https://tectonic.example.com. All configuration for your cluster was saved in clusters/example/. This directory contains sensitive credentials and should be protected. You can use the kubeconfig file there for root access to the cluster using kubectl.
```

## Access the cluster

The Tectonic Console will be up and running after the containers have downloaded. Access it at the DNS name `https://tectonic.example.com`.

Inside of the `example` cluster folder you should find any credentials, including the CA if generated, and a `kubeconfig`. Use these credentials to control the cluster with `kubectl`:

```bash
$ export KUBECONFIG=clusters/example/auth/kubeconfig
$ kubectl cluster-info
```

## Work with the cluster

For more information on working with installed clusters, see [Scaling Tectonic AWS clusters][scale-aws], and [Uninstalling Tectonic][uninstall].

## Optional customizations

### Swap modules

Needs docs for PowerDNS, etc

### Making manifest changes before installation

Needs docs to change something

## Known issues and workarounds

See the [troubleshooting][troubleshooting] document for workarounds for bugs that are being tracked.


[conventions]: ../../conventions.md
[generic]: ../../generic-platform.md
[env]: http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#cli-environment
[vars]: https://github.com/coreos/tectonic-installer/tree/master/Documentation/variables/config.md
[troubleshooting]: ../../troubleshooting/faq.md
[aws-vars]: https://github.com/coreos/tectonic-installer/tree/master/Documentation/variables/aws.md
[aws-gui]: https://coreos.com/tectonic/docs/latest/install/aws/index.html
[terraform]: https://www.terraform.io/downloads.html
[uninstall]: uninstall.md
[scale-aws]: ../../admin/aws-scale.md
[release-notes]: https://coreos.com/tectonic/releases/
[verification-key]: https://coreos.com/security/app-signing-key/
[account-login]: https://account.coreos.com/login
