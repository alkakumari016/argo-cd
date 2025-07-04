# Overview

Once installed Argo CD has one built-in `admin` user that has full access to the system. It is recommended to use `admin` user only
for initial configuration and then switch to local users or configure SSO integration.

## Local users/accounts

The local users/accounts feature serves two main use-cases:

* Auth tokens for Argo CD management automation. It is possible to configure an API account with limited permissions and generate an authentication token.
Such token can be used to automatically create applications, projects etc.
* Additional users for a very small team where use of SSO integration might be considered an overkill. The local users don't provide advanced features such as groups,
login history etc. So if you need such features it is strongly recommended to use SSO.

!!! note
    When you create local users, each of those users will need additional [RBAC rules](../rbac.md) set up, otherwise they will fall back to the default policy specified by `policy.default` field of the `argocd-rbac-cm` ConfigMap.

The maximum length of a local account's username is 32.

### Create new user

New users should be defined in `argocd-cm` ConfigMap:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  # add an additional local user with apiKey and login capabilities
  #   apiKey - allows generating API keys
  #   login - allows to login using UI
  accounts.alice: apiKey, login
  # disables user. User is enabled by default
  accounts.alice.enabled: "false"
```

Each user might have two capabilities:

* apiKey - allows generating authentication tokens for API access
* login - allows to login using UI

### Delete user

In order to delete a user, you must remove the corresponding entry defined in the `argocd-cm` ConfigMap:

Example:

```bash
kubectl patch -n argocd cm argocd-cm --type='json' -p='[{"op": "remove", "path": "/data/accounts.alice"}]'
```

It is recommended to also remove the password entry in the `argocd-secret` Secret:

Example:

```bash
kubectl patch -n argocd secrets argocd-secret --type='json' -p='[{"op": "remove", "path": "/data/accounts.alice.password"}]'
```

### Disable admin user

As soon as additional users are created it is recommended to disable `admin` user:

```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  admin.enabled: "false"
```

### Manage users

The Argo CD CLI provides set of commands to set user password and generate tokens.

* Get full users list
```bash
argocd account list
```

* Get specific user details
```bash
argocd account get --account <username>
```

* Set user password
```bash
# if you are managing users as the admin user, <current-user-password> should be the current admin password.
argocd account update-password \
  --account <name> \
  --current-password <current-user-password> \
  --new-password <new-user-password>
```

* Generate auth token
```bash
# if flag --account is omitted then Argo CD generates token for current user
argocd account generate-token --account <username>
```

### Failed logins rate limiting

Argo CD rejects login attempts after too many failed in order to prevent password brute-forcing.
The following environments variables are available to control throttling settings:

* `ARGOCD_SESSION_FAILURE_MAX_FAIL_COUNT`: Maximum number of failed logins before Argo CD starts
rejecting login attempts. Default: 5.

* `ARGOCD_SESSION_FAILURE_WINDOW_SECONDS`: Number of seconds for the failure window.
Default: 300 (5 minutes). If this is set to 0, the failure window is
disabled and the login attempts gets rejected after 10 consecutive logon failures,
regardless of the time frame they happened.

* `ARGOCD_SESSION_MAX_CACHE_SIZE`: Maximum number of entries allowed in the
cache. Default: 1000

* `ARGOCD_MAX_CONCURRENT_LOGIN_REQUESTS_COUNT`: Limits max number of concurrent login requests.
If set to 0 then limit is disabled. Default: 50.

## SSO

There are two ways that SSO can be configured:

* [Bundled Dex OIDC provider](#dex) - use this option if your current provider does not support OIDC (e.g. SAML,
  LDAP) or if you wish to leverage any of Dex's connector features (e.g. the ability to map GitHub
  organizations and teams to OIDC groups claims). Dex also supports OIDC directly and can fetch user
  information from the identity provider when the groups cannot be included in the IDToken.

* [Existing OIDC provider](#existing-oidc-provider) - use this if you already have an OIDC provider which you are using (e.g.
  [Okta](okta.md), [OneLogin](onelogin.md), [Auth0](auth0.md), [Microsoft](microsoft.md), [Keycloak](keycloak.md),
  [Google (G Suite)](google.md)), where you manage your users, groups, and memberships.

## Dex

Argo CD embeds and bundles [Dex](https://github.com/dexidp/dex) as part of its installation, for the
purpose of delegating authentication to an external identity provider. Multiple types of identity
providers are supported (OIDC, SAML, LDAP, GitHub, etc...). SSO configuration of Argo CD requires
editing the `argocd-cm` ConfigMap with
[Dex connector](https://dexidp.io/docs/connectors/) settings.

This document describes how to configure Argo CD SSO using GitHub (OAuth2) as an example, but the
steps should be similar for other identity providers.

### 1. Register the application in the identity provider

In GitHub, register a new application. The callback address should be the `/api/dex/callback`
endpoint of your Argo CD URL (e.g. `https://argocd.example.com/api/dex/callback`).

![Register OAuth App](../../assets/register-app.png "Register OAuth App")

After registering the app, you will receive an OAuth2 client ID and secret. These values will be
inputted into the Argo CD configmap.

![OAuth2 Client Config](../../assets/oauth2-config.png "OAuth2 Client Config")

### 2. Configure Argo CD for SSO

Edit the argocd-cm configmap:

```bash
kubectl edit configmap argocd-cm -n argocd
```

* In the `url` key, input the base URL of Argo CD. In this example, it is `https://argocd.example.com`
* (Optional): If Argo CD should be accessible via multiple base URLs you may
  specify any additional base URLs via the `additionalUrls` key.
* In the `dex.config` key, add the `github` connector to the `connectors` sub field. See Dex's
  [GitHub connector](https://github.com/dexidp/website/blob/main/content/docs/connectors/github.md)
  documentation for explanation of the fields. A minimal config should populate the clientID,
  clientSecret generated in Step 1.
* You will very likely want to restrict logins to one or more GitHub organization. In the
  `connectors.config.orgs` list, add one or more GitHub organizations. Any member of the org will
  then be able to login to Argo CD to perform management tasks.

```yaml
data:
  url: https://argocd.example.com

  dex.config: |
    connectors:
      # GitHub example
      - type: github
        id: github
        name: GitHub
        config:
          clientID: aabbccddeeff00112233
          clientSecret: $dex.github.clientSecret # Alternatively $<some_K8S_secret>:dex.github.clientSecret
          orgs:
          - name: your-github-org

      # GitHub enterprise example
      - type: github
        id: acme-github
        name: Acme GitHub
        config:
          hostName: github.acme.example.com
          clientID: abcdefghijklmnopqrst
          clientSecret: $dex.acme.clientSecret  # Alternatively $<some_K8S_secret>:dex.acme.clientSecret
          orgs:
          - name: your-github-org
```

After saving, the changes should take effect automatically.

NOTES:

* There is no need to set `redirectURI` in the `connectors.config` as shown in the dex documentation.
  Argo CD will automatically use the correct `redirectURI` for any OAuth2 connectors, to match the
  correct external callback URL (e.g. `https://argocd.example.com/api/dex/callback`)
* When using a custom secret (e.g., `some_K8S_secret` above,) it *must* have the label `app.kubernetes.io/part-of: argocd`.

## OIDC Configuration with DEX

Dex can be used for OIDC authentication instead of ArgoCD directly. This provides a separate set of
features such as fetching information from the `UserInfo` endpoint and
[federated tokens](https://dexidp.io/docs/custom-scopes-claims-clients/#cross-client-trust-and-authorized-party)

### Configuration:
* In the `argocd-cm` ConfigMap add the `OIDC` connector to the `connectors` sub field inside `dex.config`.
See Dex's [OIDC connect documentation](https://dexidp.io/docs/connectors/oidc/) to see what other
configuration options might be useful. We're going to be using a minimal configuration here.
* The issuer URL should be where Dex talks to the OIDC provider. There would normally be a
`.well-known/openid-configuration` under this URL which has information about what the provider supports.
e.g. https://accounts.google.com/.well-known/openid-configuration


```yaml
data:
  url: "https://argocd.example.com"
  dex.config: |
    connectors:
      # OIDC
      - type: oidc
        id: oidc
        name: OIDC
        config:
          issuer: https://example-OIDC-provider.example.com
          clientID: aaaabbbbccccddddeee
          clientSecret: $dex.oidc.clientSecret
```

### Requesting additional ID token claims

By default Dex only retrieves the profile and email scopes. In order to retrieve more claims you
can add them under the `scopes` entry in the Dex configuration. To enable group claims through Dex,
`insecureEnableGroups` also needs to be enabled. Group information is currently only refreshed at authentication
time and support to refresh group information more dynamically can be tracked here: [dexidp/dex#1065](https://github.com/dexidp/dex/issues/1065).

```yaml
data:
  url: "https://argocd.example.com"
  dex.config: |
    connectors:
      # OIDC
      - type: oidc
        id: oidc
        name: OIDC
        config:
          issuer: https://example-OIDC-provider.example.com
          clientID: aaaabbbbccccddddeee
          clientSecret: $dex.oidc.clientSecret
          insecureEnableGroups: true
          scopes:
          - profile
          - email
          - groups
```

!!! warning
    Because group information is only refreshed at authentication time just adding or removing an account from a group will not change a user's membership until they reauthenticate. Depending on your organization's needs this could be a security risk and could be mitigated by changing the authentication token's lifetime.

### Retrieving claims that are not in the token

When an Idp does not or cannot support certain claims in an IDToken they can be retrieved separately using
the UserInfo endpoint. Dex supports this functionality using the `getUserInfo` endpoint. One of the most
common claims that is not supported in the IDToken is the `groups` claim and both `getUserInfo` and `insecureEnableGroups`
must be set to true.

```yaml
data:
  url: "https://argocd.example.com"
  dex.config: |
    connectors:
      # OIDC
      - type: oidc
        id: oidc
        name: OIDC
        config:
          issuer: https://example-OIDC-provider.example.com
          clientID: aaaabbbbccccddddeee
          clientSecret: $dex.oidc.clientSecret
          insecureEnableGroups: true
          scopes:
          - profile
          - email
          - groups
          getUserInfo: true
```

## Existing OIDC Provider

To configure Argo CD to delegate authentication to your existing OIDC provider, add the OAuth2
configuration to the `argocd-cm` ConfigMap under the `oidc.config` key:

```yaml
data:
  url: https://argocd.example.com

  oidc.config: |
    name: Okta
    issuer: https://dev-123456.oktapreview.com
    clientID: aaaabbbbccccddddeee
    clientSecret: $oidc.okta.clientSecret
    
    # Optional list of allowed aud claims. If omitted or empty, defaults to the clientID value above (and the 
    # cliClientID, if that is also specified). If you specify a list and want the clientID to be allowed, you must 
    # explicitly include it in the list.
    # Token verification will pass if any of the token's audiences matches any of the audiences in this list.
    allowedAudiences:
    - aaaabbbbccccddddeee
    - qqqqwwwweeeerrrrttt

    # Optional. If false, tokens without an audience will always fail validation. If true, tokens without an audience 
    # will always pass validation.
    # Defaults to true for Argo CD < 2.6.0. Defaults to false for Argo CD >= 2.6.0.
    skipAudienceCheckWhenTokenHasNoAudience: true

    # Optional set of OIDC scopes to request. If omitted, defaults to: ["openid", "profile", "email", "groups"]
    requestedScopes: ["openid", "profile", "email", "groups"]

    # Optional set of OIDC claims to request on the ID token.
    requestedIDTokenClaims: {"groups": {"essential": true}}

    # Some OIDC providers require a separate clientID for different callback URLs.
    # For example, if configuring Argo CD with self-hosted Dex, you will need a separate client ID
    # for the 'localhost' (CLI) client to Dex. This field is optional. If omitted, the CLI will
    # use the same clientID as the Argo CD server
    cliClientID: vvvvwwwwxxxxyyyyzzzz

    # PKCE is an OIDC extension to prevent authorization code interception attacks.
    # Make sure the identity provider supports it and that it is activated for Argo CD OIDC client.
    # Default is false.
    enablePKCEAuthentication: true
```

!!! note
    The callback address should be the /auth/callback endpoint of your Argo CD URL
    (e.g. https://argocd.example.com/auth/callback).

### Requesting additional ID token claims

Not all OIDC providers support a special `groups` scope. E.g. Okta, OneLogin and Microsoft do support a special
`groups` scope and will return group membership with the default `requestedScopes`.

Other OIDC providers might be able to return a claim with group membership if explicitly requested to do so.
Individual claims can be requested with `requestedIDTokenClaims`, see
[OpenID Connect Claims Parameter](https://connect2id.com/products/server/docs/guides/requesting-openid-claims#claims-parameter)
for details. The Argo CD configuration for claims is as follows:

```yaml
  oidc.config: |
    requestedIDTokenClaims:
      email:
        essential: true
      groups:
        essential: true
        value: org:myorg
      acr:
        essential: true
        values:
        - urn:mace:incommon:iap:silver
        - urn:mace:incommon:iap:bronze
```

For a simple case this can be:

```yaml
  oidc.config: |
    requestedIDTokenClaims: {"groups": {"essential": true}}
```

### Retrieving group claims when not in the token

Some OIDC providers don't return the group information for a user in the ID token, even if explicitly requested using the `requestedIDTokenClaims` setting (Okta for example). They instead provide the groups on the user info endpoint. With the following config, Argo CD queries the user info endpoint during login for groups information of a user:

```yaml
oidc.config: |
    enableUserInfoGroups: true
    userInfoPath: /userinfo
    userInfoCacheExpiration: "5m"
```

**Note: If you omit the `userInfoCacheExpiration` setting or if it's greater than the expiration of the ID token, the argocd-server will cache group information as long as the ID token is valid!**

### Configuring a custom logout URL for your OIDC provider

Optionally, if your OIDC provider exposes a logout API and you wish to configure a custom logout URL for the purposes of invalidating 
any active session post logout, you can do so by specifying it as follows:

```yaml
  oidc.config: |
    name: example-OIDC-provider
    issuer: https://example-OIDC-provider.example.com
    clientID: xxxxxxxxx
    clientSecret: xxxxxxxxx
    requestedScopes: ["openid", "profile", "email", "groups"]
    requestedIDTokenClaims: {"groups": {"essential": true}}
    logoutURL: https://example-OIDC-provider.example.com/logout?id_token_hint={{token}}
```
By default, this would take the user to their OIDC provider's login page after logout. If you also wish to redirect the user back to Argo CD after logout, you can specify the logout URL as follows:

```yaml
...
    logoutURL: https://example-OIDC-provider.example.com/logout?id_token_hint={{token}}&post_logout_redirect_uri={{logoutRedirectURL}}
```

You are not required to specify a logoutRedirectURL as this is automatically generated by ArgoCD as your base ArgoCD url + Rootpath

!!! note
   The post logout redirect URI may need to be whitelisted against your OIDC provider's client settings for ArgoCD.

### Configuring a custom root CA certificate for communicating with the OIDC provider

If your OIDC provider is setup with a certificate which is not signed by one of the well known certificate authorities
you can provide a custom certificate which will be used in verifying the OIDC provider's TLS certificate when
communicating with it.  
Add a `rootCA` to your `oidc.config` which contains the PEM encoded root certificate:

```yaml
  oidc.config: |
    ...
    rootCA: |
      -----BEGIN CERTIFICATE-----
      ... encoded certificate data here ...
      -----END CERTIFICATE-----
```


## SSO Further Reading

### Sensitive Data and SSO Client Secrets

`argocd-secret` can be used to store sensitive data which can be referenced by ArgoCD. Values starting with `$` in configmaps are interpreted as follows:

- If value has the form: `$<secret>:a.key.in.k8s.secret`, look for a k8s secret with the name `<secret>` (minus the `$`), and read its value. 
- Otherwise, look for a key in the k8s secret named `argocd-secret`. 

#### Example

SSO `clientSecret` can thus be stored as a Kubernetes secret with the following manifests

`argocd-secret`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: argocd-secret
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-secret
    app.kubernetes.io/part-of: argocd
type: Opaque
data:
  ...
  # The secret value must be base64 encoded **once** 
  # this value corresponds to: `printf "hello-world" | base64`
  oidc.auth0.clientSecret: "aGVsbG8td29ybGQ="
  ...
```

`argocd-cm`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  ...
  oidc.config: |
    name: Auth0
    clientID: aabbccddeeff00112233

    # Reference key in argocd-secret
    clientSecret: $oidc.auth0.clientSecret
  ...
```

#### Alternative

If you want to store sensitive data in **another** Kubernetes `Secret`, instead of `argocd-secret`. ArgoCD knows to check the keys under `data` in your Kubernetes `Secret` for a corresponding key whenever a value in a configmap or secret starts with `$`, then your Kubernetes `Secret` name and `:` (colon).

Syntax: `$<k8s_secret_name>:<a_key_in_that_k8s_secret>`

> NOTE: Secret must have label `app.kubernetes.io/part-of: argocd`

##### Example

`another-secret`:
```yaml
apiVersion: v1
kind: Secret
metadata:
  name: another-secret
  namespace: argocd
  labels:
    app.kubernetes.io/part-of: argocd
type: Opaque
data:
  ...
  # Store client secret like below.
  # Ensure the secret is base64 encoded
  oidc.auth0.clientSecret: <client-secret-base64-encoded>
  ...
```

`argocd-cm`:
```yaml
apiVersion: v1
kind: ConfigMap
metadata:
  name: argocd-cm
  namespace: argocd
  labels:
    app.kubernetes.io/name: argocd-cm
    app.kubernetes.io/part-of: argocd
data:
  ...
  oidc.config: |
    name: Auth0
    clientID: aabbccddeeff00112233
    # Reference key in another-secret (and not argocd-secret)
    clientSecret: $another-secret:oidc.auth0.clientSecret  # Mind the ':'
  ...
```

### Skipping certificate verification on OIDC provider connections

By default, all connections made by the API server to OIDC providers (either external providers or the bundled Dex
instance) must pass certificate validation. These connections occur when getting the OIDC provider's well-known
configuration, when getting the OIDC provider's keys, and  when exchanging an authorization code or verifying an ID 
token as part of an OIDC login flow.

Disabling certificate verification might make sense if:
* You are using the bundled Dex instance **and** your Argo CD instance has TLS configured with a self-signed certificate
  **and** you understand and accept the risks of skipping OIDC provider cert verification.
* You are using an external OIDC provider **and** that provider uses an invalid certificate **and** you cannot solve
  the problem by setting `oidcConfig.rootCA` **and** you understand and accept the risks of skipping OIDC provider cert 
  verification.

If either of those two applies, then you can disable OIDC provider certificate verification by setting
`oidc.tls.insecure.skip.verify` to `"true"` in the `argocd-cm` ConfigMap.
