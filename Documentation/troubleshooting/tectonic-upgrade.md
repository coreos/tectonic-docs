# Troubleshooting Tectonic upgrades

This document describes how to troubleshoot issues encountered when upgrading to specific Tectonic versions.

## Upgrading to 1.8.4-tectonic.1

To update to 1.8.4-tectonic.1, first update to 1.7.9-tectonic.3. Ensure that all Third Party Resources (TPRs) have been removed from the cluster, as compatibility was removed in Kubernetes 1.8..

If you participated in the Calico Network Security Policy alpha program, please follow [the manual steps][calico-upgrade] to migrate your TPRs to CRDs. After the alpha program concludes, migrations like this will be handled via Automated Operations without any user intervention.

Once you are running `1.7.9-tectonic.3`, simply change the update channel to `Tectonic-1.8-preproduction` or `Tectonic-1.8-production` channel and click "Check for Update".

## Upgrading StatefulSets

StatefulSet rolling updates may result in the following errors after upgrading to Kubernetes v1.7.x:

* Calling `kubectl describe` on StatefulSets returns errors containing "Forbidden: pod updates may not change fields other than...".
* StatefulSet Pod DNS entries stop resolving.

To resolve these issues, delete each affected Pod and allow the StatefulSet to recreate it.

## Upgrading to 1.7.1-tectonic.1

To update to 1.7.1-tectonic.1, first update to 1.6.7-tectonic.2. Updates to 1.7.1-tectonic.1 from versions previous to 1.6.7-tectonic.2 will fail.

### Switching to 1.7 channel before updating to v1.6.7-tectonic.2.

If Tectonic Console was used to switch to the `Tectonic-1.7-preproduction` or `Tectonic-1.7-production` channel from v1.6.7-tectonic.1 or previous, first revert to the channel listed before update. Then wait for the next update check. When Tectonic Console lists the option, switch to `Tectonic-1.6.7`. Once that update is complete, use the Console to update to `Tectonic-1.7`.

### Updating to 1.7 before updating to v1.6.7_tectonic.2.

Updating Tectonic to 1.7.1-tectonic.1 before updating to 1.6.7_tectonic.2 will issue the following error:

```
Updates are not possible : Upgrade is not supported: minor version upgrade is not supported, desired: "1.7.2-tectonic.1", current: "1.6.7-tectonic.1"
```

To clear the error and proceed with the update, reset the ThirdPartyResource which stores update status.

First, use `kubectl replace` to reset to the desired version:

```sh
kubectl replace -f - <<EOF
apiVersion: coreos.com/v1
kind: AppVersion
metadata:
  name: tectonic-cluster
  namespace: tectonic-system
  labels:
    managed-by-channel-operator: "true"
status:
  currentVersion: 1.6.7-tectonic.1
  paused: false
spec:
  desiredVersion: 1.6.7-tectonic.1
  paused: false
EOF
```
Then, use Tectonic Console to switch the channel back to `Tectonic-1.6`. Click `Check for Updates`, then click `Start Upgrade`.

After upgrading to `1.6.7-tectonic.2`, switch to the `Tectonic-1.7` channel and upgrade from there.

[calico-upgrade]: upgrade-calico.md