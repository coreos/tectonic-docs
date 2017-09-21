# Troubleshooting user authentication


### Authorizing users

Regardless of the identity provider being used, authorization is performed by using the email address of the user attempting to be authorized. For example, if *john.doe@example.org* is marked as an admin, admin access is given only if the user logs in by providing the same email address. If the user attempts to log in with anything other than email, role binding policies are not applied. This issue is observed due to a Kubernetes limitation when Dex is used as the OpenID Connect provider.

If a subject other than the email attribute is used to log in, Kubernetes will prefix the field with the issuer URL and considers the new string as the username. For example, logging in with the uid, *john.doe*, makes it intuitive that it's the subject in the role binding that is matched on. Kubernetes adds extra strings to *john.doe* and provide `https://john.doe-example.org/identity#(UID)`. As a result role binding fails because there is no match between the username and the value of the subject `name` in ClusterRoleBinding.

To work around, use the email attribute when mapping users to roles.

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

In the following example, email address is provided as the name attribute to work around the Kubernetes limitation.

```
apiVersion: rbac.authorization.k8s.io/v1alpha1
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

To troubleshoot users and groups configuration, check the `tectonic-console`'s logs to see what LDAP query is being sent.

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

The return lists two issues. First, `jane2.doe@example.org` was not found in the LDAP directory. Second, a group search is attempting to check if the member field contains `john.doe@example.org`. However, the original example and LDAP server output, shown earlier in this document, qualify members based on `DN` rather than `mail`.
