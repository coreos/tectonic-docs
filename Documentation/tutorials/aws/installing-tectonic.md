# Download and install Tectonic on AWS

Once your AWS account is activated, create a CoreOS Tectonic account, and prepare your AWS account for installation. This tutorial will cover:

* Creating your CoreOS account
* Preparing your AWS account for installation
* Installing Tectonic using AWS
* Creating a new Kubernetes cluster

## Create a CoreOS account

First, sign up for a CoreOS account, which provides up to 10 free nodes of production quality Tectonic. Once completed, log in to the account to obtain the License and Pull Secret required for installation.

1. Go to [https://account.coreos.com/login][account-login].
2. Click *Sign Up* and create an account using either your Google account or another email address.
3. Enter your contact information, and click *Get License* for 10 nodes.
4. Agree to the license terms.

Check your inbox for a confirmation email. Once confirmed, log in to display the account's *Overview* page. Click "Free for use for up to 10 nodes" under Tectonic, and add your contact information. Once the update has processed, the *Overview* window will refresh to display the License and Pull Secret required for installation.

## Prepare your AWS account for installation

After activating your Tectonic account, review and complete [Creating an AWS account][creating-aws] before downloading Tectonic Installer.

Installing Tectonic requires:
* The License and Pull Secret associated with your CoreOS account
* An Identity Access Management (IAM) account
* An associated Access Key
* A domain or subdomain with DNS name service at AWS Route 53
* An AWS Route 53 EC2 SSH key pair

Tectonic will create a new AWS Virtual Private Cloud (VPC), or you can select an existing VPC. To use an existing VPC, see the [existing VPC requirements][vpc-req].

## Download and run Tectonic Installer

Having completed the AWS installation requirements, you are now ready to download and run the Tectonic Installer.

Make sure a current version of either Google Chrome or Mozilla Firefox is set as the default browser on the workstation where the installer will run.

1. Download and run Tectonic Installer by opening a new terminal and running the following command:

```
$ curl -O https://releases.tectonic.com/releases/tectonic_1.7.9-tectonic.1.tar.gz
$ curl -O https://releases.tectonic.com/releases/tectonic_1.7.9-tectonic.1.tar.gz.sig
```

2. Verify the release has been signed by the [CoreOS App Signing Key][verification-key].

```bash
$ gpg2 --keyserver pgp.mit.edu --recv-key 18AD5014C99EF7E3BA5F6CE950BDD3E0FC8A365E
$ gpg2 --verify tectonic_1.7.9-tectonic.1.tar.gz.sig tectonic_1.7.9-tectonic.1.tar.gz
# gpg2: Good signature from "CoreOS Application Signing Key <security@coreos.com>"
```

3. Double click the tarball to extract it, then navigate to the tectonic/tectonic-installer folder.
4. Double click the folder for your operating system.
5. Double click the installer to launch it.

A browser window will open Tectonic Installer to walk you through the setup process and provision your cluster.

When prompted, log in to your CoreOS account to obtain the License and Pull Secret, as described in [Create a CoreOS account](#create-a-coreos-account) above.

If you prefer to work within the terminal, extract and launch the Installer using:
```bash
tar xzvf tectonic_1.7.9-tectonic.1.tar.gz # to extract the tarball
$ cd tectonic/tectonic-installer # to change to the previously untarred directory
$ ./$PLATFORM/installer # to run Tectonic Installer
```
Where `$PLATFORM` is `linux` or `darwin`.

Use the `./$PLATFORM/installer` command to relaunch the Installer at any time. When launched, you will be given the option to *Start Over*, or to *Continue* where you left off.

![Installer Pop-up](https://coreos.com/tectonic/docs/latest/img/installer-aws.png)

Setup should take about 10-15 minutes. If you encounter any errors, please see the [AWS: Troubleshooting Installation][aws-troubleshooting] guide.

Once installation is complete, access Tectonic Console through a browser window.

## Create a new Kubernetes cluster

Step through Tectonic Installer to deploy the Tectonic Kubernetes distribution on a new cluster.

### Choose Cluster Type

**Platform**

Use the pulldown menu to select the platform on which the cluster will be installed.

(This example uses use Amazon Web Services as its Platform.)

### Define Cluster

Define AWS credentials and configuration options for the cluster.

**AWS Credentials**

To allow Tectonic to communicate with an AWS account, provide the AWS credentials.

Select *Use a normal access key*, or *Use a temporary session token*. 	

* *Use a normal access key:* Copy and paste the Access Key ID and Secret Access Keys created earlier in the AWS setup process.
* *Use a temporary session token:* CoreOS recommends that you use a temporary session token to generate temporary credentials, and protect the integrity of your main AWS credentials. Enter the Access Key Id and Secret Access Keys created during the AWS setup process.
* *Region:* Enter the EC2 region selected during your AWS setup.

Your Access Key ID is available from the AWS console. Your Secret Access Key is available from the CSV file downloaded when creating the Access Key. See [Creating an AWS account][creating-aws] for more information.

**Cluster Info**

Next, define the following attributes for the cluster:

* *Name:* Name your cluster. This name will appear in Tectonic Console.
* *Container Linux Update Channel:* Select the channel for your update mechanism for Container Linux (Stable, Beta or Alpha).
* *Experimental Features:* Check this box to enable operators to perform automatic updates.
* *CoreOS License and Pull Secret:* Copy and paste your License, and Pull Secret from your CoreOS account page. (https://account.tectonic.com)
* *Tags:* Enter any key-value pairs you wish to add to the Cluster as tags.

**Certificate Authority**

Select the option to allow Tectonic to generate a Certificate Authority and key for you.

Provide a CA certificate and key in PEM format if you are managing your own PKI.

**Submit Keys**

Select your SSH Key from the pulldown list.

Be certain to select the SSH key you submitted while setting up your AWS EC2 Network and Security keys.

**Define Nodes**

Enter Node parameters specific to your cluster.

**Networking**

Define your networking parameters:

* Select a Route 53 hosted zone domain for your cluster, and enter a subdomain.
* Select your VPC type.
* Click *Show Advanced Settings* if you wish to check or change your CIDR ranges.

**Console Login**

Enter the email address and password that will be used to log in to Tectonic Console.

**Submit**

Click *Submit* to submit your assets and create your Kubernetes cluster. (Cluster creation may take up to 20 minutes.)

Click *Advanced mode: Manually boot* to validate configuration and generate assets, but not create the cluster.

If you hit permissions errors during the creation process it is likely that your IAM account does not have sufficient privileges. Review the privileges section of our AWS: Installation Requirements to get your IAM account configured correctly.

### Boot Cluster

The final step in creating your Kubernetes cluster is to boot your cluster.

**Start Installation**

The *Start Installation* screen displays cluster creation in process.

* Terraform apply
* Resolving subdomain DNS
* Starting Tectonic console

When Terraform apply and Resolving subdomain DNS are complete, click *Download Assets* to save your cluster assets locally. (These assets will be required if you wish to destroy your cluster in the future.)

Click *Show* or *Save log* to view or save the log generated during Terraform apply.

When *Starting Tectonic Console* is complete, click *Next Step* to continue.

**Installation Complete**

Click *Go to my Tectonic Console* to open the console and begin using Tectonic. Use the email address and password used to create your Tectonic account to log in to the Console.

Click *Configure kubectl* or *Deploy Application* to open CoreOS tutorials for these subjects.

[**NEXT:** Deploying an application on Tectonic][first-app]


[install-req]: ../../install/aws/requirements.md
[ssh-key]: ../../install/aws/requirements.md#ssh-key
[vpc-req]: ../../install/aws/requirements.md#using-an-existing-vpc
[trouble-shoot]: ../../install/aws/troubleshooting.md
[privileges]: ../../install/aws/requirements.md#privileges
[first-app]: first-app.md
[creating-aws]: creating-aws.md
[aws-troubleshooting]: ../../install/aws/troubleshooting.md
[verification-key]: https://coreos.com/security/app-signing-key/
[account-login]: https://account.coreos.com/login
