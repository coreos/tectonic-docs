# Tectonic Identity and user management

User management in Tectonic is performed in two stages. The first stage authenticates a user, and the second authorizes the user to perform a given set of tasks associated with a role. Authenticating users in Tectonic is managed by Tectonic Identity, authorizing users is controlled by Kubernetes [Role-Based Access Control (RBAC)][k8s-rbac]. All Tectonic clusters enable RBAC which uses the user information produced by Tectonic Identity to enforce permissions.

Tectonic Identity is an authentication service for both Tectonic Console and `kubectl` and allows these components to talk to the API server on an end user's behalf. Users are either defined in the Tectonic `ConfigMap` or integrated with Tectonic through an external Identity Provider (IdP).

This document provides an overview of Tectonic Identity, and its use in user authentication and access control.

For more information see:
* [Tectonic Identity configuration][identity-config]
* [LDAP integration][ldap-integration]
* [SAML integration][saml-integration]

## Components of Tectonic Identity

The three major components of Tectonic Identity are the API server, Tectonic Console, and kubectl.

* Tectonic Identity: logs users into corporate identity systems (LDAP, SAML, etc.) and issues credentials.
* Kubernetes API server: authenticates user credentials against Tectonic Identity.
* Tectonic Console: requests credentials on the user's behalf from Tectonic Identity.
* kubectl: Tectonic Console provides credentials to authenticated users which may be used with kubectl when issuing commands to the Kubernetes API server.

### Tectonic API server

The Tectonic API Server is expected to enable its OpenID Connect plugin, deferring to Dex for authentication. The API server is not a Dex client.

### Kubectl

For Dex, kubectl is a public client. All kubectl instances share a `client_id` and `client_secret`, and the `client_secret` isn't considered private. kubectl communicates only with the API server.

### Tectonic Console

Tectonic Console communicates with both Dex and the API server. Therefore, Tectonic Console is considered to be an admin client for Dex. To be trusted by both Kubernetes and Dex, ID Tokens are issued to both Console and kubectl. When a user logs in to a Tectonic Console, Dex creates an ID Token for both Console and kubectl allowing Console to both authenticate with Kubernetes and the Dex APIs.

## RBAC in Tectonic

Authorization in Tectonic is controlled by a set of permissions called Roles. Role Bindings grant the permissions associated with a Role to a subject. Subject is defined as a type of account used to access the Tectonic clusters.

For more information, see [rbac-config][rbac-config].

## Tectonic authentication through Dex

Tectonic Identity is built on top of [Dex][dex], an open-source OpenID Connect server.

Dex runs natively on top of Tectonic clusters using [custom resource definitions][crds] ([since Tectonic 1.8.x][release-note-1.8.4], before: [third-party resources][third-party]), and drives API server authentication through the OpenID Connect plugin. Clients, such as Tectonic Console and kubectl, act on behalf users who can log in to Tectonic cluster through an identity provider, such as LDAP, that both Tectonic and Dex support.

Dex server issues short-lived, signed tokens on behalf of users. This token response, called ID Token, is a signed JSON web token. ID Token contains names, emails, unique identifiers, and a set of groups that can be used to identify a user. Dex publishes public keys, and Tectonic API server uses these to verify ID Tokens. The username and group information of a user is used in conjunction with RBAC to enforce authorization policy.

Dex does not support hashing and instead strongly recommends that all administrators use TLS. This is achieved by configuring port 636 instead of 389 in the Tectonic Identity `configMap`.


[identity-config]: tectonic-identity-config.md
[ldap-integration]: ldap-integration.md
[saml-integration]: saml-integration.md
[dex]: https://github.com/coreos/dex/
[rbac-config]: rbac-config.md
[crds]:        https://github.com/coreos/dex/blob/master/Documentation/storage.md#kubernetes-custom-resource-definitions-crds
[release-note-1.8.4]: https://coreos.com/tectonic/releases/#1.8.4-tectonic.1
[third-party]: https://github.com/coreos/dex/blob/master/Documentation/storage.md#Kubernetes-third-party-resources
[k8s-rbac]: https://kubernetes.io/docs/admin/authorization/rbac/
