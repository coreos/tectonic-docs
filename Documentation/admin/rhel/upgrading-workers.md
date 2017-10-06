# Upgrading Tectonic workers on Red Hat Enterprise Linux

Tectonic worker versions on RHEL are not managed from the Tectonic Console. Upgrades are performed using standard Red Hat package management tools and practices, specifically the YUM program.

After installing the `tectonic-release` package from the [installation guide][install], the RHEL system will be configured to upgrade the worker software to the latest available version with the `yum upgrade` command. Depending on your site's update practices, additional care should be taken to ensure the RHEL workers continue to run the same version as the rest of the Tectonic cluster.

The following sections describe alternative configurations for restricting Tectonic updates on RHEL. Note that for any of these options, updating the package will not affect the running service; either run `systemctl restart kubelet` or reboot the system to begin using the new version.

## Whitelisting versions of the Tectonic update channel

Each Tectonic cluster is configured to use a specific update channel so that clusters will not be upgraded past a certain version milestone. The YUM repository option `includepkgs` can achieve the same effect on RHEL workers by whitelisting acceptable packages. This method is well-suited for sites that apply updates automatically.

For example, if the Tectonic cluster is on the `1.6` update channel, the repository can be configured to only acknowledge versions starting with `1.6` by writing the following line under the `[tectonic]` section in `/etc/yum.repos.d/tectonic.conf`:

```ini
includepkgs=rkt tectonic-release tectonic-worker-1.6.*
```

Tectonic worker packages have three numerical components, so this pattern supports updating workers through `1.6.9`, `1.6.10`, etc. Since `includepkgs` defines a whitelist, this line also accepts all versions of the `rkt` and `tectonic-release` packages, which can be updated independently of the Tectonic worker version.

New packages may also be released for the same Tectonic worker version to apply bug fixes or security updates. For example, if a cluster version is fixed to `1.7.3`, RHEL workers may continue to receive updates by using the following line:

```ini
includepkgs=rkt tectonic-release tectonic-worker-1.7.3
```

If the cluster's update channel changes, these repo configurations will need to be updated appropriately.

## Disabling the repository by default

The safest way to prevent unexpected updates is to disable the `tectonic` YUM repository. This can be done by setting `enabled=0` under the `[tectonic]` section in `/etc/yum.repos.d/tectonic.repo`. When using this option, individual YUM commands can update Tectonic packages by using the `--enablerepo=tectonic` argument. This method is well-suited for sites that apply updates manually.

If not whitelisting versions as described above, find [an available worker version][worker-packages] that matches the rest of the Tectonic cluster. For example, the following command will upgrade to the latest `1.7.5` package, and then the worker will not be updated until the repo is explicitly enabled again:

```sh
yum --enablerepo=tectonic upgrade tectonic-worker-1.7.5
```

[install]: ../../install/rhel/installing-workers.md
[worker-packages]: https://yum.prod.coreos.systems/repo/tectonic-rhel/7Server/x86_64/repoview/tectonic-worker.html
