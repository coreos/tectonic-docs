# SAML integration

Tectonic Identity authenticates clients, such as `kubectl` and Tectonic Console, for access to the Kubernetes API and, through it, to Tectonic cluster services. All Tectonic clusters use [Role-Based Access Control (RBAC)][rbac-config] to govern access to cluster services. Tectonic Identity authenticates a user's identity, and RBAC enforces authorization based on that identity. Tectonic Identity can map cluster RBAC bindings to an existing Security Assertion Markup Language Identity Provider (SAML IdP) over a secure channel.

Configure Tectonic Identity to map to your existing SAML IdP, to enable user authentication for defined users and groups.

## Integrating Tectonic Identity with SAML

### Authentication workflow

When a user logs in to Tectonic Console, Tectonic Identity constructs a SAML `AuthnRequest` and  redirects the user to the SAML server. Once the user is verified, the SAML server returns an `AuthnResponse` to Tectonic Identity, which then verifies the response, and retrieves user data, such as username and groups.

<div class="row">
  <div class="col-lg-10 col-lg-offset-1 col-md-10 col-md-offset-1 col-sm-10 col-sm-offset-1 col-xs-10 col-xs-offset-1">
    <a href="../img/tectonic-saml.png" class="co-m-screenshot">
      <img src="../img/tectonic-saml.png" class="img-responsive">
    </a>
  </div>
</div>

1. A user logs in to Tectonic Console.

2. Tectonic Identity identifies the user’s origin and redirects the user to the SAML server,  requesting authentication (`AuthnRequest`).

3. SAML then builds the authentication response in the form of an XML file containing the user’s username or email address (`NameID`), signs it using the private key of an X.509 certificate, and posts this information to Tectonic Identity as an `AuthnResponse`.

4. Tectonic Identity retrieves the authentication response and validates it using the public key of the X.509 certificate.

5. If the identity is established, the user is provided access to Tectonic cluster.

## Configuring Tectonic Identity for SAML authentication

To configure Tectonic Identity, first:
* Configure SAML for access by Tectonic.
* Add Tectonic users to the SAML database.
* Ensure that Tectonic clusters are up and running.

Tectonic Identity pulls its configuration options from a `ConfigMap`, which admins can view and edit using `kubectl`. To prevent misconfiguration, use the [kubectl downloaded from Tectonic Console][kubeconfig-download] to edit Tectonic Identity's ConfigMap.

1. Use kubectl to back up the existing config:

```
kubectl get configmaps tectonic-identity --namespace=tectonic-system -o yaml > tectonic-config.yaml.bak
```

2. Edit the current `ConfigMap` with the desired changes:

```
kubectl edit configmaps tectonic-identity --namespace=tectonic-system
```

3. Add the `connectors` configuration.

Replace `example.com` with your IdP domain name, and `tectonic-domain` with your Tectonic domain name.

```yaml
connectors:
- type: saml
  id: saml
  name: SAML
  config:
    ssoURL: https://example.com/sso/saml
    redirectURI: https://tectonic-domain/identity/callback
    usernameAttr: name
    emailAttr: email
    groupsAttr: groups # optional
    caData: /path/to/ca.pem
    entityIssuer: https://tectonic-domain/identity/callback
    ssoIssuer: https://example.com/sso/saml
    nameIDPolicyFormat: persistent
```

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

Create role bindings to allow SAML users access to Kubernetes resources through both Tectonic Console and kubectl. See [Configuring RBAC](#configuring-rbac) for more details.

### Using kubectl as a SAML user

To use kubectl as a SAML user, log in to Tectonic Console as the desired user. Then follow the standard process to [download kubectl and configure credentials][kubeconfig-download] for the cluster.

### Handling SAML authentication expiration

Every twenty four hours, access tokens for the session will expire. Users must log in again to regain access to the cluster.

## Tectonic Identity configuration parameters

Parameters must be configured on both Tectonic Identity and the SAML IdP to enable them to exchange user data. These parameters are defined for Tectonic Identity in the `config-dev.yaml` file, which must be mapped directly to those configured on the IdP side.

Once configured, SAML IdP exchanges security tokens with Tectonic Identity, acting as a SAML consumer  to exchange user authentication data.

See your provider's IdP documentation for more information on their specific SAML configuration options.

The table below describes the parameters that configure Tectonic Identity federation with SAML:

|Tectonic Identity (`config-dev.yaml`) |example| Description|
|:------------- |:-------------|:-----|
| `ssoURL`      | `https://example.com/sso/saml`    |Single sign-on URL received from SAML. The URL carries the SAML2 security token with user information.|
| `redirectURI`      | `https://tectonic-domain/identity/callback`    |Callback URL from Tectonic Identity. SAML sends the security token containing user information with SAML assertion to this location. |
| `usernameAttr` | `name`    |Username attribute set in SAML. This maps to the ID token and claims from the callback URL in the returned assertion.|
| `emailAttr` | `email`    |Email attribute in the returned assertion. This maps to the ID token and claims from the callback URL.|
| `groupsAttr` | `groups`    |(Optional.) Group attribute in the returned assertion. This maps to the ID token and claims from the callback URL.|
| `caData` | `/path/to/ca.pem`    |Path to your Certificate Authority Data, or the base64 value of the X.509 Certificate. This is used to validate the signature of the SAML response. |
| Name ID Format | Name ID Format    |Unique identifier string for the user’s linked account. Tectonic Identity assumes that this value is both unique and constant. Therefore, your SAML IdP and Tectonic Identity should be in agreement with this choice. The identifier specification, called `NameIDPolicy`, determines what format should be requested in the SAML assertion. The `NameID` format indicated in the `NameIDPolicy` is included in the SAML assertion. If Tectonic Identity requests a `NameID` format unknown to the IdP or for which the IdP is not configured, the authentication flow will fail. Select the default value, `unspecified` unless you require a specific format. |
| `entityIssuer`      | `https://tectonic-domain/identity/callback`    |(Optional.) Custom value for Tectonic Identity's Issuer value. When provided, Tectonic Identity will include this as the Issuer value during AuthnRequest. It will also override the redirectURI as the required audience when evaluating AudienceRestriction elements in the response.|
| `ssoIssuer`      | `https://example.com/sso/saml`    |(Optional.) Issuer value expected in the SAML response.|
| `nameIDPolicyFormat`      | `persistent`    |(Optional.) Requested format of the NameID. The NameID value is is mapped to the user ID of the user. This can be an abbreviated form of the full URI with just the last component. For example, if this value is set to "emailAddress" the format will resolve to: `urn:oasis:names:tc:SAML:1.1:nameid-format:emailAddress`. If no value is specified, defaults to: `urn:oasis:names:tc:SAML:2.0:nameid-format:persistent`.|


[k8s-auth]: https://kubernetes.io/docs/admin/authorization/#roles-rolesbindings-clusterroles-and-clusterrolebindings
[kubeconfig-download]: https://coreos.com/tectonic/docs/latest/tutorials/aws/first-app.html#configuring-credentials
[rbac-config]: rbac-config.md
