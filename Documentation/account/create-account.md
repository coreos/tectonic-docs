# Creating a CoreOS account

Creating a CoreOS account activates a CoreOS License for up to 10 free nodes.

First, go to [https://account.coreos.com/login][sign-up] to sign up for a CoreOS account. Sign up using an existing +Google account, or enter an email address and password, and click *Create Account*.

A confirmation email will be sent to the listed account. Check your inbox for an email from CoreOS Support, and click *Verify Email*.

Log in to complete registration and access the account *Overview* page.

<div class="row">
  <div class="col-lg-10 col-lg-offset-1 col-md-10 col-md-offset-1 col-sm-10 col-sm-offset-1 col-xs-10 col-xs-offset-1">
    <a href="../img/coreos-account-overview.png" class="co-m-screenshot">
      <img src="../img/coreos-account-overview.png" class="img-responsive">
    </a>
  </div>
</div>

This page lists your current CoreOS subscriptions and other available products, and provides access to your CoreOS License and Pull Secret.

## CoreOS License and Pull Secret

The *CoreOS License* and *Pull Secret* are required to access container images for CoreOS products.

Provide the License and Pull Secret during Tectonic Installation to enable cluster access. When requested, click the *copy and paste* link to open a window containing these strings. Then, copy and paste them into the appropriate fields during the installation process.

<div class="row">
  <div class="col-lg-10 col-lg-offset-1 col-md-10 col-md-offset-1 col-sm-10 col-sm-offset-1 col-xs-10 col-xs-offset-1">
    <a href="../img/coreos-account-license-secret.png" class="co-m-screenshot">
      <img src="../img/coreos-account-license-secret.png" class="img-responsive">
    </a>
  </div>
</div>

The License and Pull secret may also be entered manually. The `config.json` file contains a pull secret granting access to CoreOS container registries.

Download `config.json` from the account *Overview* screen and write it to the Docker configuration directory. On CoreOS Container Linux, copy the file to `/home/core/.docker/config.json`. On most other Linux distributions, copy the file to `/root/.docker/config.json` or the configured Docker configuration directory.

Docker will use the credentials in `config.json` when fetching Tectonic software.


[sign-up]: https://account.coreos.com/login
