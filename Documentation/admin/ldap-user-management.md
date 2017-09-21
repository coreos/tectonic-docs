# LDAP User Management

Tectonic Identity is an authentication service for both Tectonic Console and `kubectl`. It facilitates communication to the API server on the end user's behalf. All Tectonic clusters enable Role Based Access Control (RBAC). Tectonic Identity can be mapped to an LDAP service ensuring RBAC role bindings for groups and users map to your exiting LDAP system.

This document describes managing users and access control in Tectonic and Kubernetes using LDAP.

## Configuring Tectonic Identity for LDAP authentication

Tectonic Identity is configured through the Tectonic Console to allow for LDAP user authentication. The following information is required to integrate with an LDAP server:

* Your LDAP server host name and port
* An LDAP service account capable of querying for users and groups
* An LDAP base distinguished name (dn) representing where to start the search from for users and groups
* The attributes used to describe users and groups. For example, mail, username, and so on
* (Optional) The root CA if certificate CA verification is desired

Use Tectonic Console to enable LDAP authentication in your cluster:

1. Go to *Administration > Cluster Settings*, and click *LDAP*

2. In the LDAP window that opens, enter your Host and optional port of the LDAP server in the form "host:port", and select a verification option:
 * No SSL: required if your LDAP host does not use SSL.
 * Skip verification: select to skip verification of the CA.
 * Root CA: if selected, enter the PEM data containing the root CAs.

3. Click Continue to enter your LDAP Service Account username and password.

4. Click *Continue* to enter user search criteria:
  * **Base DN**: The root LDAP directory to begin the user search. Translates to the query: `(&(objectClass=person)(uid=<username>))`.
  * **Filter**: Filter(s) applied to every user search to limit the results. For example, when a user search executes, object results could be limited to those with an `objectClass` of [person][person-ldap-rfc].
  * **Username Attribute**: the end user's username for login. The field used when searching for users. This is the field users will use to login. For example, a commonName (assuming it's unique) or an email address.
  * **User ID Attribute**: string representation of the user's [unique identifier][uid-ldap-rfc].
  * **Email Attribute**: maps to user's email address.
  * **Name Attribute**: maps to user's display name.

  The example configuration above translates to the following diagram:

  do we need or want this? - beth

  <div class="row">
    <div class="col-lg-10 col-lg-offset-1 col-md-10 col-md-offset-1 col-sm-10 col-sm-offset-1 col-xs-12 col-xs-offset-1">
      <a href="../img/ldap-server-user-search-diagram.png" class="co-m-screenshot">
        <img src="../img/ldap-server-user-search-diagram.png">
      </a>
    </div>
  </div>

5. Click Continue to enter Group information:

  * **Base DN**: The root LDAP directory to begin the group search. Translates to the query: `(&(objectClass=group)(member=<user uid>))`.
  * **Filter**: Optional filter to apply when searching the directory. For example, when a group search executes, object results could be limited to those with an `objectClass` of [groupOfNames][groupOfNames-ldap-rfc].
  * **User Attribute**: The user field that a group uses to identify a user is part of a group. For example, groups that specify each member as `member: cn=john,dc=example,dc=org` in the LDAP directory, are using the [Distinguished Name (DN)][dn-ldap-rfc] attribute.
  * **Member Attribute**: matches a user to a group using the User Attribute listed above. (what? - beth) The [member][member-ldap-rfc] field associating a user, using the User Attribute mentioned above, with the group.
  * **Name Attribute**: mapps to a group's display name.


  The example configuration above translates to the following diagram:

  <div class="row">
    <div class="col-lg-10 col-lg-offset-1 col-md-10 col-md-offset-1 col-sm-10 col-sm-offset-1 col-xs-12 col-xs-offset-1">
      <a href="../img/ldap-server-group-search-diagram.png" class="co-m-screenshot">
        <img src="../img/ldap-server-group-search-diagram.png">
      </a>
    </div>
  </div>

8. Click Continue to enter a test *Username* and *Password*, then click *Test Configuration* to verify that users and groups are correctly configured.
If the query does not return expected results, see the [Troubleshooting](#troubleshooting) section.

  The following is a sample LDAP directory used in the test steps.

  ```
  # john, example.org
  dn: cn=john,dc=example,dc=org
  objectClass: person
  objectClass: inetOrgPerson
  uid: john.doe
  mail: john.doe@example.org
  cn: john
  sn: doe
  userPassword:: e1NTSEF9dkltSFZkNTgzN3JBaVdEZ2xyVXFyeE9nM1FETHBkM04=

  # jane, example.org
  dn: cn=jane,dc=example,dc=org
  objectClass: person
  objectClass: inetOrgPerson
  uid: jane.doe
  mail: jane.doe@example.org
  cn: jane
  sn: doe
  userPassword:: e1NTSEF9dkltSFZkNTgzN3JBaVdEZ2xyVXFyeE9nM1FETHBkM04=

  # tstgrp, groups, example.org
  dn: cn=tstgrp,ou=groups,dc=example,dc=org
  objectClass: top
  objectClass: groupOfNames
  member: cn=john,dc=example,dc=org
  cn: tstgrp
  ```

  In this example the user `john.doe`'s `dn` is in `tstgrp`, but `jane.doe` is in no groups. You can query for these users in the *Test Configuration* page to verify this.

9. From the *Test Configuration* page, click *Continue* to see instructions on updating the given Tectonic Identity.
10. Click *My Account*.
11. From the *Profile* page, download the new configuration file.

 Keep a backup of the existing config in case something goes wrong during the update.

### Applying a new Tectonic Identity configuration

To apply a new Tectonic Identity configuration to your Kubernetes cluster:

1. On your terminal, run `kubectl apply`:

```bash
$ kubectl apply -f ~/Downloads/new-tectonic-config.yaml
```
If successful the following message is displayed:

```
configmap "tectonic-identity" configured
```

2. Restart the Tectonic Identity pods for the changes to take effect. Run the following command to trigger a rolling update and attach the current date as an annotation.

```bash
$ kubectl patch deployment tectonic-identity \
    --patch "{\"spec\":{\"template\":{\"metadata\":{\"annotations\":{\"date\":\"`date +'%s'`\"}}}}}" \
    --namespace tectonic-system
```
If successful the following message is displayed:

```
"tectonic-identity" patched
```

3. Log out of Tectonic Console and log in again by using the LDAP authentication credentials.

> In order for an LDAP user to have any access to Kubernetes resources, both from the console and kubectl, you must setup role bindings. See [Configuring Access](#configuring-access) for more details.

### Using kubectl as an LDAP user

To use `kubectl` as an LDAP user:

1. Log in to your Tectonic Console as the desired user.
2. Navigate to *My Account*
3. Verify your identity and download `kubectl` configuration.
4. Set the KUBECONFIG environment variable to the `kubectl` configuration file. For example:

```
export KUBECONFIG=~/Download/kubectl-config
```

Until otherwise modified, you can still use your static account for further administrative setup.


[onboard-user]: onboard-user.md
[onboard-admin]: onboard-admin.md
[onboard-team]: onboard-team.md
[onboard-service-account]: onboard-service-account.md
[person-ldap-rfc]: https://tools.ietf.org/html/rfc4519#section-3.12
[uid-ldap-rfc]: https://tools.ietf.org/html/rfc4519#section-2.39
[groupOfNames-ldap-rfc]: https://tools.ietf.org/html/rfc4519#section-3.5
[dn-ldap-rfc]: https://tools.ietf.org/html/rfc4511#section-4.1.3
[member-ldap-rfc]: https://tools.ietf.org/html/rfc4519#section-2.17
[k8s-auth]: https://kubernetes.io/docs/admin/authorization/#roles-rolesbindings-clusterroles-and-clusterrolebindings
