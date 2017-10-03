# Creating a CoreOS account

Creating a CoreOS account activates a License for up to 10 free nodes on a Tectonic cluster.

First, go to [https://account.coreos.com/login][account-login] to sign up for a CoreOS account. Sign up using an existing Google+ account, or enter an email address and password, and click *Create Account*.

A confirmation email will be sent to the listed account. Check your inbox for an email from CoreOS Support, and click *Verify Email*.

Log in to complete registration and access the account *Overview* page, which lists available CoreOS products.

<div class="row">
  <div class="col-lg-10 col-lg-offset-1 col-md-10 col-md-offset-1 col-sm-10 col-sm-offset-1 col-xs-10 col-xs-offset-1">
    <a href="../img/coreos-account-new.png" class="co-m-screenshot">
      <img src="../img/coreos-account-new.png" class="img-responsive">
    </a>
  </div>
</div>

Click *Free for use up to 10 nodes* below Tectonic, enter your contact information, and click *Get License for 10 nodes* to complete the registration process. Once complete, the *Overview* page will list active subscriptions, and provide access to your CoreOS License and Pull Secret.

<div class="row">
  <div class="col-lg-10 col-lg-offset-1 col-md-10 col-md-offset-1 col-sm-10 col-sm-offset-1 col-xs-10 col-xs-offset-1">
    <a href="../img/coreos-account-overview.png" class="co-m-screenshot">
      <img src="../img/coreos-account-overview.png" class="img-responsive">
    </a>
  </div>
</div>

## CoreOS License and Pull Secret

The *CoreOS License* and *Pull Secret* are required to install Tectonic and access container images for CoreOS products.

When requested during Tectonic Installation, [log in][account-login] to your CoreOS account, and click the *copy and paste* link to open a window containing these strings. Then, copy and paste them into the appropriate fields during the installation process.

<div class="row">
  <div class="col-lg-10 col-lg-offset-1 col-md-10 col-md-offset-1 col-sm-10 col-sm-offset-1 col-xs-10 col-xs-offset-1">
    <a href="../img/coreos-account-license-secret.png" class="co-m-screenshot">
      <img src="../img/coreos-account-license-secret.png" class="img-responsive">
    </a>
  </div>
</div>

The License and Pull Secret may also be downloaded for later use. The Pull Secret will be downloaded as `config.json` which may be added to the Docker configuration directory to grant access to CoreOS container registries.


[account-login]: https://account.coreos.com/login
