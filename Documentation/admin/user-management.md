# User Management through Tectonic Identity

Tectonic Identity is an authentication service for both Tectonic Console and `kubectl`, and allows these components to talk to the API server on an end user's behalf. All Tectonic clusters also enable Role-Based Access Control (RBAC) which uses the user information produced by Identity to enforce permissions.

This document describes using the Tectonic Identity config file to
* Edit the Tectonic Identity config file
* Create a static user
* Create a Service Account to manage in-cluster API access

For information on creating other account types, see:
* Creating user roles
* Creating user accounts
* Creating service accounts

## Identity Configuration

Tectonic Identity pulls all its configuration options from a config file stored in a `ConfigMap`, which admins can view and edit using `kubectl`. As a precaution, use the administrative kubeconfig in the  [downloaded `assets.zip`][assets-zip] when editing Identity's config in case of misconfiguration.

First, backup the existing config using `kubectl`:

```
kubectl get configmaps tectonic-identity --namespace=tectonic-system -o yaml > identity-config.yaml.bak
```

Edit the current `ConfigMap` with the desired changes:

```
kubectl edit configmaps tectonic-identity --namespace=tectonic-system
```

Trigger a rolling update using `kubectl`. Identity's deployment is intended to be resilient against invalid config files, but admins should verify the new state and restore the `ConfigMap` backup if Identity enters a crash loop. The following command will cause an update:

```
kubectl patch deployment tectonic-identity \
    --patch "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}" \
    --namespace tectonic-system
```

The update's success can be inspected by watching the pods in the `tectonic-system` namespace.

```
kubectl get pods --namespace=tectonic-system
```

### Add static user

Static users are those defined directly in the Identity `ConfigMap`. Static users are intended to be used for initial setup, and may also be used for troubleshooting and recovery. A static user acts as a stand-in, authenticating users without a connection to a backend Identity provider. To add a new static user, update the tectonic-identity `ConfigMap` with a new `staticPasswords` entry.

```yaml
    staticPasswords:
    # The following fields are required.
    - email: "test1@example.com"
      # bcrypt hash for string "password"
      hash: "$2a$10$2b2cU8CPhOTaGrs1HRQuAueS7JTT5ZHsHSzYiFPm1leZck7Mc8T4W"
      # username to display. NOT used during login.
      username: "test1"
      # randomly generated using uuidgen.
      userID: "1d55c7c4-a76d-4d74-a257-31170f2c4845"
```

A bcrypt encoded hash of the user's password can be generated using the [coreos/bcrypt-tool](https://github.com/coreos/bcrypt-tool/releases/tag/v1.0.0).

When generating Tectonic Console passwords with `bcrypt-tool`, using values higher than the default of `-cost=10` may result in timeouts. bcrypt also imposes a maximum password length of 56 bytes.

To ensure the static user has been added successfully, Log in to Tectonic Console using the static user's username and password.

### Change Password for Static User

To change the password of an existing user, generate a bcrypt hash for the new password (using [coreos/bcrypt-tool](https://github.com/coreos/bcrypt-tool/releases/tag/v1.0.0)) and plug in this value into the tectonic-identity `ConfigMap` for the selected user.

```yaml
    staticPasswords:
    # Existing user entry.
    - email: "test1@example.com"
      # Newly generated bcrypt hash
      hash: "$2a$10$TcWtvcw0N8.xK8nKdBw80uzYij6cJwuQhtfYnEf/hEC9bRTzlWdIq"
      username: "test1"
      userID: "1d55c7c4-a76d-4d74-a257-31170f2c4845"
```

After the config changes are applied, the user can log in to the console using the new password.

### Managing in-cluster API access

Pods use service accounts to authenticate against the Kubernetes API from within the cluster. Service accounts are API credentials stored in the Kubernetes API and mounted into pods at well known paths, giving the pod an identity which can be access-controlled. If an app uses `kubectl` or the official Kubernetes Go client within a pod to talk to the API, these credentials are loaded automatically.

Because RBAC denies all requests unless explicitly allowed, service accounts, and the pods that use them, must be granted access through RBAC rules.

Kubernetes automatically creates a "default" service account in every namespace. If pods don't explicitly request a service account, they're assigned to this "default" account.

```
$ kubectl get serviceaccounts
NAME               SECRETS   AGE
default            1         1h
$ kubectl create deployment nginx --image=nginx
deployment "nginx" created
$ kubectl get pods
NAME                     READY     STATUS    RESTARTS   AGE
nginx-3121059884-x7btf   1/1       Running   0          20s
```

Inspect the `spec` of the pod to see that the pod of the deployment has been assigned the "default" service account:

```
$ kubectl get pod nginx-3121059884-x7btf -o yaml
# ...
spec:
  containers:
  - image: nginx
    imagePullPolicy: Always
    name: nginx
    volumeMounts:
    - mountPath: /var/run/secrets/kubernetes.io/serviceaccount
      name: default-token-twmyd
      readOnly: true
  serviceAccountName: default
  volumes:
  - name: default-token-twmyd
    secret:
      defaultMode: 420
      secretName: default-token-twmyd
# ...
```

To allow the pod to talk to the API server, create a `Role` for the account, then use a RoleBinding to grant the service account the role's powers. For example, if the pod must be able to read `ingress` resources:

```yaml
kind: Role
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: default-service-account
  namespace: default
rules:
  - apiGroups: ["extensions"]
    resources: ["ingress"]
    verbs: ["get", "watch", "list"]
    nonResourceURLs: []
---
kind: RoleBinding
apiVersion: rbac.authorization.k8s.io/v1alpha1
metadata:
  name: default-service-account
  namespace: default
subjects:
  # The subject is the target service account
  - kind: ServiceAccount
    name: default
    namespace: default
roleRef:
  # The roleRef specifies the role to give to the
  # service account.
  kind: Role
  namespace: default
  name: default-service-account # Tectonic also provides "readonly", "user", and "admin" cluster roles.
  apiGroup: rbac.authorization.k8s.io
```

If multiple pods running in the same namespace require different levels of access, create a unique service account for each.

```
$ kubectl create serviceaccount my-robot-account
serviceaccount "my-robot-account" created
```

The newly created service account can be mounted into the pod by specifying the service account's name in the pod spec.

```yaml
apiVersion: extensions/v1beta1
kind: Deployment
metadata:
  name: nginx-deployment
spec:
  replicas: 3
  template:
    metadata:
      labels:
        k8s-app: nginx
    spec:
      containers:
      - name: nginx
        image: nginx:1.7.9
      serviceAccountName: my-robot-account # Specify the custom service account
```

The `RoleBinding` would then reference the custom service account name instead of "default".

Note that because service account credentials are stored in secrets, any clients with the ability to read secrets can extract the bearer token and act on behalf of that service account. Be cautious when giving service accounts powers or clients the ability to read secrets.


[assets-zip]: assets-zip.md
[k8s-rbac]: http://kubernetes.io/docs/admin/authorization/#rbac-mode
