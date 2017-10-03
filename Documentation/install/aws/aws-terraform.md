# Install Tectonic on AWS with Terraform

Use this guide to manually install a Tectonic cluster on an AWS account. To install Tectonic on AWS with a graphical installer instead, refer to the [AWS graphical installer documentation][aws-gui].

Generally, the AWS platform templates adhere to the standards defined by the project [conventions][conventions] and [generic platform requirements][generic]. This document aims to document the implementation details specific to the AWS platform.

## Prerequsities

* **Terraform**: Tectonic Installer includes and requires a specific version of Terraform. This is included in the Tectonic Installer tarball. See the [Tectonic Installer release notes][release-notes] for information about which Terraform versions are compatible.
* **DNS**: Ensure that the DNS zone is already created and available in Route 53 for the account. For example if the `tectonic_base_domain` is set to `kube.example.com` a Route 53 zone must exist for this domain and the AWS nameservers must be configured for the domain.
* **Tectonic Account**: Register for a [Tectonic Account][register], which is free for up to 10 nodes. You must provide the cluster license and pull secret during installation.

## Getting Started

### Download and extract Tectonic Installer

Open a new terminal and run the following command to download Tectonic Installer.

```bash
$ curl -O https://releases.tectonic.com/releases/tectonic_1.7.3-tectonic.3.tar.gz # download
```

Verify the release has been signed by the [CoreOS App Signing Key][verification-key].

```bash
$ gpg2 --keyserver pgp.mit.edu --recv-key 18AD5014C99EF7E3BA5F6CE950BDD3E0FC8A365E
$ gpg2 --verify tectonic-1.7.3-tectonic.2.tar.gz.asc tectonic-1.7.3-tectonic.2.tar.gz
# gpg2: Good signature from "CoreOS Application Signing Key <security@coreos.com>"
```

Extract the tarball and navigate to the `tectonic` directory.

```bash
$ tar xzvf tectonic-1.7.3-tectonic.2.tar.gz
$ cd tectonic
```

### Initialize and configure Terraform

We need to add the `terraform` binary to our `PATH`. The platform should be `darwin` or `linux`.

```bash
$ export PATH=$PATH:$(pwd)/tectonic-installer/darwin # Put the `terraform` binary in the PATH
```

Download the Tectonic Terraform modules.

```bash
$ terraform init platforms/aws
```

Configure your AWS credentials. See the [AWS docs][env] for details.

```bash
$ export AWS_ACCESS_KEY_ID=
$ export AWS_SECRET_ACCESS_KEY=
```

Set your desired region:

```bash
$ export AWS_REGION=
```

Next, specify the cluster configuration.

## Customize the deployment

Customizations to the base installation live in `examples/terraform.tfvars.aws`. Export a variable that will be your cluster identifier:

```bash
$ export CLUSTER=my-cluster
```

Create a build directory to hold your customizations and copy the example file into it:

```bash
$ mkdir -p build/${CLUSTER}
$ cp examples/terraform.tfvars.aws build/${CLUSTER}/terraform.tfvars
```

Edit the parameters with your AWS details, domain name, license, etc. [View all of the AWS specific options][aws-vars] and [the common Tectonic variables][vars].

## Deploy the cluster

Test out the plan before deploying everything:

```bash
$ terraform plan -var-file=build/${CLUSTER}/terraform.tfvars platforms/aws
```

Next, deploy the cluster:

```bash
$ terraform apply -var-file=build/${CLUSTER}/terraform.tfvars platforms/aws
```

This should run for a little bit, and when complete, your Tectonic cluster should be ready.

## Access the cluster

The Tectonic Console should be up and running after the containers have downloaded. You can access it at the DNS name configured in your variables file.

Inside of the `/generated` folder you should find any credentials, including the CA if generated, and a `kubeconfig`. You can use this to control the cluster with `kubectl`:

```bash
$ export KUBECONFIG=generated/auth/kubeconfig
$ kubectl cluster-info
```

## Work with the cluster

For more information on working with installed clusters, see [Scaling Tectonic AWS clusters][scale-aws], and [Uninstalling Tectonic][uninstall].

## Known issues and workarounds

See the [troubleshooting][troubleshooting] document for workarounds for bugs that are being tracked.


[conventions]: ../../conventions.md
[generic]: ../../generic-platform.md
[env]: http://docs.aws.amazon.com/cli/latest/userguide/cli-chap-getting-started.html#cli-environment
[register]: https://account.coreos.com/signup/summary/tectonic-2016-12
[account]: https://account.coreos.com
[vars]: https://github.com/coreos/tectonic-installer/tree/master/Documentation/variables/config.md
[troubleshooting]: ../../troubleshooting/faq.md
[aws-vars]: https://github.com/coreos/tectonic-installer/tree/master/Documentation/variables/aws.md
[aws-gui]: https://coreos.com/tectonic/docs/latest/install/aws/index.html
[terraform]: https://www.terraform.io/downloads.html
[uninstall]: uninstall.md
[scale-aws]: ../../admin/aws-scale.md
[release-notes]: https://coreos.com/tectonic/releases/
[verification-key]: https://coreos.com/security/app-signing-key/
