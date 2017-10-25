# Troubleshooting user authentication


### Authorizing users

Tectonic Identity uses email address for authorization across all identity provider platforms. For example, if *john.doe@example.org* is marked as an admin, admin access is given only if John Doe logs in with the same email address. If users attempt to log in with anything other than email, role binding policies are not applied.

The following outlines an example LDAP configuration for the user *john.doe* and an associated ClusterRoleBinding.

```
john.doe, Users, 5866a86d3187bc712e435b35, example.org
dn: uid=john.doe,ou=Users,o=5866a86d3187bc712e435b35,dc=example,dc=org
givenName: john
jcLdapAdmin: TRUE
uid: john.doe
uidNumber: 5006
loginShell: /bin/bash
homeDirectory: /home/John.doe
sn: Doe
cn: John Doe
objectClass: exampleUser
gidNumber: 5006
mail: john.doe@example.org
memberOf: cn=tectonic_users,ou=Users,o=5866a86d3187bc712e435b35,dc=example,dc=org
```

Use the email address as the `name` attribute for the RoleBinding definition for the account:

```
apiVersion: rbac.authorization.k8s.io/v1beta1
kind: RoleBinding
metadata:
  name: admin-john.doe-namespace-all-access-binding
  namespace: john.doe-namespace
roleRef:
  apiGroup: rbac.authorization.k8s.io
  kind: Role
  name: john.doe@example.org
subjects:
- kind: User
  name: john.doe
  namespace: john.doe-namespace
```

### Troubleshooting users and groups configuration

To troubleshoot users and groups configuration, check the Tectonic Console's logs to see the LDAP query being sent.

For more information, see [Aggregated Logging in Tectonic][logging-tectonic].

Locate the Console's pod name:

```bash
$ kubectl --namespace=tectonic-system get pods

NAME                                         READY     STATUS    RESTARTS   AGE
tectonic-console-3824021701-50j0x            1/1       Running   0          2h
tectonic-identity-3193269714-bg19p           1/1       Running   0          24m
tectonic-ingress-controller-99581103-rd0cj   1/1       Running   0          2h
```

Use `kubectl` to tail the logs for the console pod:

```bash
$ kubectl --namespace=tectonic-system logs -f tectonic-console-3824021701-50j0x

2017/01/29 22:29:47 ldap: no results returned for filter: "(&(objectClass=person)(mail=jane2.doe@example.org))"
2017/01/29 22:30:16 ldap: groups search with filter "(&(objectClass=groupOfNames)(member=john.doe@example.org))" returned no groups
```

The return lists two issues:
* `jane2.doe@example.org` was not found in the LDAP directory.
* A group search is attempting to check if the member field contains `john.doe@example.org`.


[logging-tectonic]: ../admin/logging.md
