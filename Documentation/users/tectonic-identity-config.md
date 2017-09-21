# Tectonic Identity Configuration

Tectonic Identity is an authentication service for both Tectonic Console and `kubectl`, and allows these components to talk to the Kubernetes API server on an end user's behalf. All Tectonic clusters also enable Role-Based Access Control (RBAC) which uses the user information produced by Identity to enforce permissions.

Tectonic Identity is configured through a YAML ConfigMap, which provides for the configuration of static users. Static users are not tied to any authentication provider, and may be used to set up or troubleshoot your Tectonic installation.

This document provides an overview of the Tectonic Identity ConfigMap, and describes how to create a Static User.

For information on creating other account types, see:
* [Defining Tectonic user roles][creating-roles]
* [Creating Tectonic accounts][creating-accounts]
* [Adding a service account to a Tectonic cluster][creating-service-accounts]

## Identity Configuration

Tectonic Identity pulls its configuration options from a config file stored in a `ConfigMap`, which admins can view and edit using `kubectl`. As a precaution in case of misconfiguration, use the administrative kubeconfig in the [downloaded `assets.zip`][assets-zip] when editing Identity's config.

First, use `kubectl` to backup the existing config:

```
kubectl get configmaps tectonic-identity --namespace=tectonic-system -o yaml > identity-config.yaml.bak
```

Then, edit the current `ConfigMap` with the desired changes:

```
kubectl edit configmaps tectonic-identity --namespace=tectonic-system
```

Finally, use `kubectl` to trigger  a rolling update:

```
kubectl patch deployment tectonic-identity \
    --patch "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}" \
    --namespace tectonic-system
```

Identity's deployment is intended to be resilient against invalid config files. Verify the new state to confirm the update, and restore the `ConfigMap` backup if problems are encountered.

The update's success can be inspected by watching the pods in the `tectonic-system` namespace.

```
kubectl get pods --namespace=tectonic-system
```

## Add a static user

Static users are those defined directly in the Tectonic Identity `ConfigMap`. Static users are intended to be used for initial setup, and may also be used for troubleshooting and recovery. A static user acts as a stand-in, authenticating users without a connection to a backend identity provider.

To add a static user, update the Tectonic Identity `ConfigMap` with a new `staticPasswords` entry.

```yaml
staticPasswords:
# The following fields are required.
- email: "test1@example.com"
  hash: "$2a$10$2b2cU8CPhOTaGrs1HRQuAueS7JTT5ZHsHSzYiFPm1leZck7Mc8T4W"
    # bcrypt hash for string "password"
  username: "test1" # username to display. NOT used during login.
  userID: "1d55c7c4-a76d-4d74-a257-31170f2c4845"
    # randomly generated using uuidgen.
```

A bcrypt encoded hash of the user's password can be generated using the [coreos/bcrypt-tool](https://github.com/coreos/bcrypt-tool/releases/tag/v1.0.0).

When generating Tectonic Console passwords with `bcrypt-tool`, using values higher than the default of `-cost=10` may result in timeouts. bcrypt also imposes a maximum password length of 56 bytes.

To ensure the static user has been added successfully, log in to Tectonic Console using the static user's username and password.

## Change password for static user

To change the password of an existing user, generate a bcrypt hash for the new password (using [coreos/bcrypt-tool](https://github.com/coreos/bcrypt-tool/releases/tag/v1.0.0)) and add this value to the Tectonic Identity `ConfigMap` for the selected user.

```yaml
staticPasswords:
# Existing user entry.
- email: "test1@example.com"
  hash: "$2a$10$TcWtvcw0N8.xK8nKdBw80uzYij6cJwuQhtfYnEf/hEC9bRTzlWdIq"
    # bcrypt hash for new password
  username: "test1"
  userID: "1d55c7c4-a76d-4d74-a257-31170f2c4845"
```

Apply ConfigMap changes, then bring up the Tectonic Identity Pod with the new changes to enable users to log in to Tectonic Console using the new password.


[assets-zip]: ../admin/assets-zip.md
[k8s-rbac]: http://kubernetes.io/docs/admin/authorization/#rbac-mode
[creating-roles]: creating-roles.md
[creating-accounts]: creating-accounts.md
[creating-service-accounts]: creating-service-accounts.md
