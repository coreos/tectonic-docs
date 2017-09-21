# LDAP integration

Tectonic Identity authenticates clients, such as `kubectl` and Tectonic Console, for access to the Kubernetes API and, through it, to Tectonic cluster services. All Tectonic clusters use [Role-Based Access Control (RBAC)][rbac-config] to govern access to cluster services. Tectonic Identity authenticates a user's identity, and RBAC enforces authorization based on that identity. Tectonic Identity can map cluster RBAC bindings to an existing LDAP (Lightweight Directory Access Protocol) system over a secure channel.

Configure Tectonic Identity to map to your existing LDAP service, to enable user authentication for defined users and groups.

## Configuring Tectonic Identity for LDAP authentication

Tectonic Identity is configured through the Tectonic Console to allow for LDAP user authentication. The following information is required to integrate with an LDAP server:

* The LDAP server host name and port.
* An LDAP service account capable of querying for users and groups.
* An LDAP base distinguished name (dn), from which the search for users and groups will begin.
* Attributes used to describe users and groups. For example: email and username.
* (Optional) The root CA if certificate CA verification is desired.

Use Tectonic Console to enable LDAP authentication in your cluster:

1. Go to *Administration > Cluster Settings*, and click *LDAP*

2. In the LDAP window that opens, enter your host and the (optional) port of the LDAP server in the form `host:port`, and select a verification option:

 * **No SSL:** required if your LDAP host does *not* use SSL.
 * **Skip verification:** select to skip verification of the CA.
 * **Root CA:** if selected, enter the PEM data containing the root CAs.

3. Click *Continue* to enter your LDAP Service Account username and password (obtained from your LDAP admin).

4. Click *Continue* to enter user search criteria:

  * **Base DN**: The root LDAP directory to begin the user search. Translates to the query: `(&(objectClass=person)(uid=<username>))`.
  * **Filter**: (Optional) Filter to apply when searching the directory. For example, when a user search executes, object results could be limited to those with an `objectClass` of [person][person-ldap].
  * **Username Attribute**: The end user's username for login. For use with Kubernetes, the Username Attribute must be set to the user's email address.
  * **User ID Attribute**: String representation of the user's [unique identifier][uid-ldap].
  * **Email Attribute**: Required. Attribute to map to email. For use with Kubernetes, this value must be set to something other than the user's email address.
  * **Name Attribute**: The user's display name.

5. Click *Continue* to enter Group information:

  * **Base DN**: The root LDAP directory to begin the group search. Translates to the query: `(&(objectClass=group)(member=<user uid>))`.
  * **Filter**: (Optional) Filter to apply when searching the directory. For example, when a group search executes, object results could be limited to those with an `objectClass` of [groupOfNames][groupOfNames-ldap].
  * **User Attribute**: The user field that a group uses to identify that a user is part of a group. For example, groups that specify each member as `member: cn=john,dc=example,dc=org` in the LDAP directory, are using the [Distinguished Name (DN)][dn-ldap] attribute.
  * **Member Attribute**: The [member][member-ldap] field associating a user with a group, using the User Attribute mentioned above.
  * **Name Attribute**: The group's display name.

6. Click *Continue* to enter a test *Username* and *Password*, then click *Test Configuration* to verify that users and groups are correctly configured.

7. When confirmed, click *Continue*, and follow the instructions to backup the existing, then apply the new configuration to the cluster.

## Applying LDAP Config changes

First, click **Download Existing Config** to download a backup of the existing configuration.

Then, click **Download New Config** to download the new configuration YAML file.

Run `kubectl apply` to apply the new configuration:

```
kubectl apply -f path/to/config/file/new-tectonic-config.yaml
```

Finally, trigger a rolling update of the Identity pods, which will read the new configuration:

```
kubectl patch deployment tectonic-identity \
    --patch "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}" \
    --namespace tectonic-system
```

If successful the following message is displayed:

```
"tectonic-identity" patched
```

## Using kubectl as an LDAP user

To use `kubectl` as an LDAP user:

1. Log in to Tectonic Console as the desired user.
2. From *My Account*, click *KUBECTL: Download Configuration*.
3. Set the `KUBECONFIG` environment variable to the kubectl configuration file. For example:

`export KUBECONFIG=~/Download/kubectl-config`

Until otherwise modified, use your static account for further administrative setup.

## Example schema

The following examples demonstrate mapping LDAP schema to Tectonic Identity configurations.

### User LDAP schema and Tectonic mapping

User LDAP schema:

```ldap
dn: cn=john,dc=people,dc=example,dc=org
objectClass: person
cn: jane
email: jane.doe@example.com

dn: cn=developers,dc=groups,dc=example,dc=org
objectClass: group
cn: developers
member: jane
```

Corresponding Tectonic Identity config, in which the `cn` of the user is matched with the `member` field of the group:

```
userAttr: cn
memberAttr: member
```

### Group LDAP schema and Tectonic mapping

Group LDAP schema:

```ldap
dn: cn=john,dc=people,dc=example,dc=org
objectClass: person
cn: jane
email: jane.doe@example.com
memberOf: cn=developers,dc=groups,dc=example,dc=org

dn: cn=developers,dc=groups,dc=example,dc=org
objectClass: group
cn: developers
```

Corresponding Tectonic Identity config, in which the `memberOf` attribute of the user is matched with the `dn` field of the group:

```
userAttr: memberOf
memberAttr: dn
```


[person-ldap]: https://tools.ietf.org/html/rfc4519#section-3.12
[uid-ldap]: https://tools.ietf.org/html/rfc4519#section-2.39
[groupOfNames-ldap]: https://tools.ietf.org/html/rfc4519#section-3.5
[dn-ldap]: https://tools.ietf.org/html/rfc4511#section-4.1.3
[member-ldap]: https://tools.ietf.org/html/rfc4519#section-2.17
[rbac-config]: rbac-config.md
