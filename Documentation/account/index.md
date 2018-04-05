# Getting started with CoreOS Tectonic

When you’re ready to create a production ready cluster, follow these instructions to create an account, and spin up a free 10-node cluster.

## Create a CoreOS account

Go to [https://account.coreos.com/login][account-login], and click *Sign Up*.

Check your inbox for a confirmation email. Click through to accept the terms of the license, activate your account, and be redirected to the account's *Overview* page.

Click "Free for use up to 10 nodes" under Tectonic. Enter your contact information, and click *Get License for 10 nodes*.

Once the update has processed, the *Overview* window will refresh to include links to download the License and Pull Secret.

## Obtain your License and Pull Secret

During installation, you will be asked to provide your Tectonic License and Pull Secret, which are available from your [Account Overview][account-overview] page.

When requested, log in to your [account][account-login], and click the *Overview* tab. Then, click the buttons to *Download CoreOS License* and *Download Pull Secret*.

<div class="row">
  <div class="col-lg-10 col-lg-offset-1 col-md-10 col-md-offset-1 col-sm-10 col-sm-offset-1 col-xs-10 col-xs-offset-1">
    <a href="../img/coreos-account-overview.png" class="co-m-screenshot">
      <img src="../img/coreos-account-overview.png" class="img-responsive">
    </a>
  </div>
</div>

## Download and install Tectonic

Select your platform, and follow the download and installation instructions provided.

* [Install on AWS with a graphical interface][aws-gui]
* [Install on AWS with Terraform][aws-tf]
* [Install on Azure with Terraform][azure-tf]
* [Install on bare metal with Terraform][bare-tf]

Installation requires the CoreOS License and Pull Secret described above. Be certain to create an account and enable the 10 free nodes before launching Tectonic Installer.


[aws-tf]: ../install/aws/aws-terraform.md
[aws-gui]: ../install/aws/index.md
[azure-tf]: ../install/azure/azure-terraform.md
[bare-tf]: ../install/bare-metal/index.md
[account-login]: https://account.coreos.com/login
[account-overview]: create-account.md#coreos-account-license-and-pull-secret
