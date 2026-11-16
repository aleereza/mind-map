# Authentication Methods {#authentication-methods}

> type: category

How does a service know who is calling, and how do OAuth, Auth0, and the other methods differ? Authentication means proving who you are. A caller proves it with a credential: a password, a key, or a certificate. A standard such as [OAuth 2.0](https://www.rfc-editor.org/rfc/rfc6749.html) (Open Authorization) defines the messages that carry the proof. A product such as [Auth0](https://auth0.com/docs/get-started/auth0-overview) runs the standard for you. A program has no keyboard and no phone. It holds a secret or a private key, or its platform gives it an identity.

The map has four parts. Core Ideas defines the words. Standards compares the methods. Products compares the services that run them. Programs and Service Accounts shows how a program signs in, with Snowflake and Iceberg examples.

## Core Ideas {#core-ideas}

> type: category

Six ideas explain every method in this map. They come in dependency order: each idea uses only the ones before it.

### Authentication and Authorization {#authentication-and-authorization}

> type: concept

> edge.related: oauth-2-0

Authentication checks who the caller is. [NIST](https://pages.nist.gov/800-63-4/sp800-63b.html) defines it as checking the authenticators that a subject uses to claim a digital identity. Authorization decides what the caller may do. In Snowflake, a sign-in method such as a key pair proves who the user is. The roles that the user holds carry the privileges ([access control overview](https://docs.snowflake.com/en/user-guide/security-access-control-overview)).

HTTP shows the same split with two status codes. A 401 means that the request lacks valid authentication credentials. A 403 means that the server understood the request and refuses to fulfill it. Both codes come from RFC 9110, a standards document of the Internet Engineering Task Force (IETF) ([401](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.5.2), [403](https://www.rfc-editor.org/rfc/rfc9110.html#section-15.5.4)). On 2026-11-15, `curl -i https://api.github.com/user` sent no credential and got `HTTP/2 401` with the body `{"message": "Requires authentication", "documentation_url": "https://docs.github.com/rest", "status": "401"}`.

### Principal {#principal}

> type: concept

A principal is an identity that can prove itself to a service. It can stand for a person or for a program. The Kerberos specification uses the word for the users and servers in its database ([RFC 4120](https://www.rfc-editor.org/rfc/rfc4120.html#section-1.1)). Snowflake calls a principal a user, and a Snowflake user can stand for a person or for a service ([access control overview](https://docs.snowflake.com/en/user-guide/security-access-control-overview)). In AWS, a principal is an entity that can perform actions on resources. It can be a root user, an IAM (Identity and Access Management) user, or a role ([IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)). Service Account Names lists what each platform calls the identity of a program.

### Credential and Factors {#credential-and-factors}

> type: concept

A credential is the secret or key that proves who a principal is. [NIST](https://pages.nist.gov/800-63-4/sp800-63b.html) names three factors:

- Something you know, such as a password.
- Something you have, such as a one-time-password device.
- Something you are, such as a biometric match.

Multi-factor authentication (MFA) needs more than one factor. A person can use all three. A program stores a secret that it knows, or it owns a private key or a certificate. Programs and Service Accounts describes each kind.

### Token {#token}

> type: concept

A token is a short string that a caller shows in place of its credential. It replaces a user name and a password with one value that the API (application programming interface) understands ([RFC 6749](https://www.rfc-editor.org/rfc/rfc6749.html#section-1.4)). Three kinds appear in this map:

- An access token lets the caller use an API.
- An ID token says who signed in. OpenID Connect defines it.
- A refresh token gets a new access token when the old one expires ([RFC 6749](https://www.rfc-editor.org/rfc/rfc6749.html#section-1.5)).

A token that follows [RFC 6750](https://www.rfc-editor.org/rfc/rfc6750.html#section-1.2) is a bearer token: any party that holds it can use it. The holder sends it in the header `Authorization: Bearer <token>` ([RFC 6750](https://www.rfc-editor.org/rfc/rfc6750.html#section-2.1)). A leaked bearer token therefore gives its finder the same access as the caller, until the token expires.

#### JSON Web Token {#jwt}

> type: concept

A JSON Web Token (JWT, said "jot") is a token format: a set of claims with a signature ([RFC 7519](https://www.rfc-editor.org/rfc/rfc7519.html#section-1)). It has three parts, joined by dots: the header, the claims, and the signature. Each part is base64url encoded ([RFC 7519](https://www.rfc-editor.org/rfc/rfc7519.html#section-3)). The script in About the Examples built this token with a shared secret:

`eyJhbGciOiJIUzI1NiIsInR5cCI6IkpXVCJ9.eyJpc3MiOiJodHRwczovL2lkcC5leGFtcGxlLmNvbSIsInN1YiI6InN2Yy1sb2FkZXIiLCJhdWQiOiJodHRwczovL2FwaS5leGFtcGxlLmNvbSIsImlhdCI6MTc5NDcwMDgwMCwiZXhwIjoxNzk0NzA0NDAwfQ.rcEL4KU9STAjyB24nWb1fMzFRPEM-ec53vwKRIApr7g`

The header decodes to `{"alg": "HS256", "typ": "JWT"}`. The claims decode to `{"iss": "https://idp.example.com", "sub": "svc-loader", "aud": "https://api.example.com", "iat": 1794700800, "exp": 1794704400}`. These five names are registered claims ([RFC 7519](https://www.rfc-editor.org/rfc/rfc7519.html#section-4.1)). They are the issuer, the subject, the audience, the time of issue, and the time of expiry. The audience is the API that the token is for. The times count seconds since 1970-01-01 UTC. Here 1794700800 is 2026-11-15 00:00:00 UTC, and the token expires one hour later.

Anyone who holds the token can read the first two parts. The signature lets the checker detect any change. HS256 is a signature made with a shared secret, by HMAC using SHA-256. RS256 is a signature made with a private key, by RSA using SHA-256 ([RFC 7518](https://www.rfc-editor.org/rfc/rfc7518.html#section-3.1)). An API that receives a JWT access token verifies the signature, the issuer, the audience, and the expiry ([RFC 9068](https://www.rfc-editor.org/rfc/rfc9068.html#section-4)).

### Caller, Issuer, Checker {#caller-issuer-checker}

> type: concept

A method with a third party has three parties: the caller, the issuer of the proof, and the checker of the proof. Each standard has its own names for them:

| Standard | Caller | Issuer | Checker | Proof |
| --- | --- | --- | --- | --- |
| OAuth 2.0 | client | authorization server | resource server | access token |
| OpenID Connect | relying party (RP) | OpenID provider (OP) | relying party | ID token |
| SAML 2.0 | user, through a browser | identity provider (IdP) | service provider (SP) | assertion |
| Kerberos | client | key distribution center (KDC) | application server | ticket |

The names come from [RFC 6749](https://www.rfc-editor.org/rfc/rfc6749.html#section-1.1), the [OpenID Connect terms](https://openid.net/specs/openid-connect-core-1_0.html#Terminology), the [SAML technical overview](https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html), and [RFC 4120](https://www.rfc-editor.org/rfc/rfc4120.html#section-1.7). With Basic authentication, no third party takes part: the service checks the user ID and password itself ([RFC 7617](https://www.rfc-editor.org/rfc/rfc7617.html#section-2)).

### Trust and Federation {#trust-and-federation}

> type: concept

Federation means that a service trusts another system to sign users or programs in. The service keeps the public keys of that system and checks every signature. Only that system handles the password. For people, the result is single sign-on (SSO): one sign-in works for many services ([SAML technical overview](https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html)).

An OpenID Connect provider publishes its settings at `<issuer>/.well-known/openid-configuration` ([OIDC Discovery](https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderConfig)). On 2026-11-15, Google's file held `"issuer": "https://accounts.google.com"`, `"token_endpoint": "https://oauth2.googleapis.com/token"`, and `"jwks_uri": "https://www.googleapis.com/oauth2/v3/certs"`. The token endpoint is the address where a client asks for a token. The `jwks_uri` leads to a JWKS (JSON Web Key Set), which is the list of the issuer's public keys. On the same day, the JWKS of GitHub Actions at `https://token.actions.githubusercontent.com/.well-known/jwks` held 4 RSA keys. Each key has `"alg": "RS256"`, `"use": "sig"`, and its own `kid`. A JWT header can carry the `kid` of its signing key, so the checker picks the right public key ([RFC 7515](https://www.rfc-editor.org/rfc/rfc7515.html#section-4.1.4)).

## Standards {#authentication-standards}

> type: category

A standard defines the messages between the caller and the service. The table compares seven of them:

| Standard | Defined by | What the caller presents |
| --- | --- | --- |
| Basic | [RFC 7617](https://www.rfc-editor.org/rfc/rfc7617.html) | The user ID and the password, joined by a colon and base64 encoded |
| API key | Each API; the [OpenAPI](https://spec.openapis.org/oas/v3.1.0.html#security-scheme-object) format for describing HTTP APIs covers it | A secret string in a header, a query parameter, or a cookie |
| OAuth 2.0 | [RFC 6749](https://www.rfc-editor.org/rfc/rfc6749.html) | An access token |
| OpenID Connect | [OpenID Foundation](https://openid.net/specs/openid-connect-core-1_0.html) | An ID token, which is a JWT |
| SAML 2.0 | [OASIS](https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html) | An XML assertion about the user |
| Kerberos | [RFC 4120](https://www.rfc-editor.org/rfc/rfc4120.html) | A ticket from the key distribution center |
| Mutual TLS (Transport Layer Security) | [RFC 8446](https://www.rfc-editor.org/rfc/rfc8446.html) | A certificate, in the TLS handshake |

The nodes below follow the order of the table.

### Basic Authentication {#basic-authentication}

> type: concept

The client joins the user ID, a colon, and the password. It encodes the result in base64 and sends it with every request: `Authorization: Basic <encoded>` ([RFC 7617](https://www.rfc-editor.org/rfc/rfc7617.html#section-2)). Base64 is an encoding that anyone can reverse, so each request carries the password in readable form. The specification says that Basic should not be used without enhancements such as HTTPS ([RFC 7617](https://www.rfc-editor.org/rfc/rfc7617.html#section-4)).

OAuth 2.0 reuses the format for client credentials. The example in [RFC 6749](https://www.rfc-editor.org/rfc/rfc6749.html#section-4.4.2) sends `Authorization: Basic czZCaGRSa3F0MzpnWDFmQmF0M2JW`. The value decodes to `s6BhdRkqt3:gX1fBat3bV`: the client ID, a colon, and the client secret.

### API Key {#api-key}

> type: concept

An API key is a secret string that the owner of an API issues to a caller. The caller sends the same string with every request. Each API chooses where the key goes. The [OpenAPI specification](https://spec.openapis.org/oas/v3.1.0.html#security-scheme-object) describes a key with two settings: `name`, which is the header, query parameter, or cookie that carries the key, and `in`, which is `query`, `header`, or `cookie`. The specification's own example is `{"type": "apiKey", "name": "api_key", "in": "header"}`. It means that the request carries the header `api_key: <key>`. A Snowflake programmatic access token also travels as a secret string with each request. Stored Secret describes it.

### OAuth 2.0 {#oauth-2-0}

> type: concept

> edge.contrasts: auth0

The abstract of [RFC 6749](https://www.rfc-editor.org/rfc/rfc6749.html) says that OAuth 2.0 lets an application obtain limited access to an HTTP service. The application acts for a resource owner, or on its own behalf. The framework defines four grant types, which are the ways to get an access token ([RFC 6749](https://www.rfc-editor.org/rfc/rfc6749.html#section-1.3)):

- Authorization code: a person approves access in a browser.
- Client credentials: a program uses its own credentials. Programs and Service Accounts covers it.
- Implicit and resource owner password credentials: [RFC 9700](https://www.rfc-editor.org/rfc/rfc9700.html#section-2.4) says that clients should not use the implicit grant and must not use the password grant.

The framework leaves other choices to each product. The token format is outside its scope ([RFC 6749](https://www.rfc-editor.org/rfc/rfc6749.html#section-1.4)). A scope is a list of strings that name the access that a client requests. The authorization server defines the strings ([RFC 6749](https://www.rfc-editor.org/rfc/rfc6749.html#section-3.3)).

Snowflake offers OAuth in two forms. With Snowflake OAuth, Snowflake is both the authorization server and the resource server. With External OAuth, a third-party identity provider is the authorization server ([authentication overview](https://docs.snowflake.com/en/user-guide/security-authentication-overview)). Snowflake lists Okta, Microsoft Entra ID, and Ping Identity PingFederate as external authorization servers ([External OAuth](https://docs.snowflake.com/en/user-guide/oauth-ext-overview)).

#### Authorization Code Flow {#authorization-code-flow}

> type: concept

A person uses this flow to let an application act for them. First, the client sends the user's browser to the authorization server, for example `GET /authorize?response_type=code&client_id=s6BhdRkqt3&state=xyz&redirect_uri=https%3A%2F%2Fclient%2Eexample%2Ecom%2Fcb` ([RFC 6749](https://www.rfc-editor.org/rfc/rfc6749.html#section-4.1.1)). Next, the user signs in and approves. Then the server sends the browser back to the client with a short-lived code. Last, the client sends the code to the token endpoint and receives an access token.

A client that cannot keep a secret, such as a mobile app, adds PKCE (Proof Key for Code Exchange). It first sends a transformed form of a random secret, the `code_challenge`. It later sends the secret itself, the `code_verifier`, with the code. The server transforms the verifier, compares it with the challenge, and denies access if they differ ([RFC 7636](https://www.rfc-editor.org/rfc/rfc7636.html#section-1.1)).

### OpenID Connect {#openid-connect}

> type: concept

> edge.contrasts: oauth-2-0

OpenID Connect (OIDC) is an identity layer on top of OAuth 2.0. It lets an application verify who the user is, based on the authentication that an authorization server performed ([OIDC Core](https://openid.net/specs/openid-connect-core-1_0.html#Introduction)). The server returns an ID token, a JWT with claims about that sign-in ([OIDC Core](https://openid.net/specs/openid-connect-core-1_0.html#IDToken)). The application asks for it with the scope `openid`, which every OIDC request must contain ([OIDC Core](https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest)). Trust and Federation shows a real discovery document.

### SAML 2.0 {#saml-2-0}

> type: concept

> edge.contrasts: openid-connect

SAML (Security Assertion Markup Language) 2.0 lets organizations that do business together exchange security information. It defines two roles: the identity provider (IdP) and the service provider (SP) ([SAML technical overview](https://docs.oasis-open.org/security/saml/Post2.0/sstc-saml-tech-overview-2.0.html)). A SAML assertion is an XML statement about a user, for example that the user is logged in at the IdP. In the browser sign-in flow, the service sends the browser to the IdP. The user signs in there. The IdP sends the browser back to the service with a SAML assertion. Snowflake supports federated authentication through SAML 2.0 security integrations. Okta and Microsoft Entra ID provide native support for it ([federated authentication](https://docs.snowflake.com/en/user-guide/admin-security-fed-auth-overview)).

### Kerberos {#kerberos}

> type: concept

Kerberos verifies the identities of principals on an open network. It uses a trusted third party and shared secret keys ([RFC 4120](https://www.rfc-editor.org/rfc/rfc4120.html#section-1.1)). The third party is the key distribution center (KDC), a network service that supplies tickets and temporary session keys ([RFC 4120](https://www.rfc-editor.org/rfc/rfc4120.html#section-1.7)). To reach a service, the client sends the application server a ticket and an authenticator. The authenticator is encrypted in the session key. The network carries the session key only in encrypted form.

### Mutual TLS {#mutual-tls}

> type: concept

In ordinary TLS, the server shows a certificate. In mutual TLS (mTLS), the client shows one as well. A TLS 1.3 client sends a Certificate message only when the server asks for client authentication ([RFC 8446](https://www.rfc-editor.org/rfc/rfc8446.html#section-4.4.2)). The server then knows the caller by its certificate. OAuth can use mTLS to authenticate a client, and to bind an access token to the client's certificate ([RFC 8705](https://www.rfc-editor.org/rfc/rfc8705.html)). On 2026-11-15, the discovery document of Microsoft Entra ID listed `self_signed_tls_client_auth` among its token endpoint authentication methods.

Snowflake lists five authentication methods for applications: workload identity federation, Snowflake OAuth, External OAuth, key-pair authentication, and programmatic access tokens ([authentication overview](https://docs.snowflake.com/en/user-guide/security-authentication-overview)). Programs and Service Accounts describes them. In the [Kafka → Iceberg → Snowflake map](../integration/kafka-iceberg-snowflake/kafka-iceberg-snowflake.md), producers and connectors use client certificates to reach Kafka.

## Products {#identity-products}

> type: category

A product is software or a service that runs the standards for you. It keeps the users and the apps, signs people in, and issues the tokens. The standards leave some choices open, such as the token format ([RFC 6749](https://www.rfc-editor.org/rfc/rfc6749.html#section-1.4)) and the scope strings ([RFC 6749](https://www.rfc-editor.org/rfc/rfc6749.html#section-3.3)). Each product makes its own choices there. The table shows the OAuth 2.0 client credentials request in five products. It also shows the request in Snowflake Open Catalog, the hosted Iceberg REST catalog that the [Iceberg in Snowflake map](../snowflake/iceberg/snowflake-iceberg.md) describes. Every request has `grant_type=client_credentials`.

| Product | Token endpoint | Names the access with | `expires_in` in the docs' sample |
| --- | --- | --- | --- |
| Auth0 | `https://{yourDomain}/oauth/token` | `audience`, the identifier of the API | 86400 |
| Okta, custom authorization server | `https://{yourOktaDomain}/oauth2/default/v1/token` | `scope`, a custom scope | 3600 |
| Microsoft Entra ID | `https://login.microsoftonline.com/{tenant}/oauth2/v2.0/token` | `scope`, the URI of the resource plus `/.default` | 3599 |
| Keycloak | `https://<host>/realms/<realm>/protocol/openid-connect/token` | The roles of the client's service account | 60 |
| Amazon Cognito | `https://<prefix>.auth.<region>.amazoncognito.com/oauth2/token` | `scope`, as `<resource server identifier>/<scope name>` | 3600 |
| Snowflake Open Catalog | `https://<account>.snowflakecomputing.com/polaris/api/catalog/v1/oauth/tokens` | `scope`, for example `PRINCIPAL_ROLE:ALL` | 3600 |

The values come from the documentation of [Auth0](https://auth0.com/docs/get-started/authentication-and-authorization-flow/client-credentials-flow/call-your-api-using-the-client-credentials-flow), [Okta](https://developer.okta.com/docs/guides/implement-grant-type/clientcreds/main/), [Microsoft Entra ID](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow), [Keycloak](https://www.keycloak.org/docs/latest/server_admin/), [Amazon Cognito](https://docs.aws.amazon.com/cognito/latest/developerguide/token-endpoint.html), and [Snowflake Open Catalog](https://docs.snowflake.com/en/user-guide/tables-iceberg-configure-catalog-integration-rest-check-config).

### Auth0 {#auth0}

> type: concept

> edge.related: client-credentials-flow

Auth0 is a hosted service. Its documentation describes it as a drop-in solution that adds authentication and authorization to your applications ([Auth0 overview](https://auth0.com/docs/get-started/auth0-overview)). It supports OAuth 2.0, OpenID Connect, SAML, and other standards ([protocols](https://auth0.com/docs/authenticate/protocols)). An Auth0 tenant is therefore an OAuth 2.0 authorization server that someone else runs for you. In the tenant, you configure applications and store users ([tenants](https://auth0.com/docs/get-started/auth0-overview/create-tenants)). Okta announced the purchase of Auth0 on 3 March 2021 ([Form 8-K](https://www.sec.gov/Archives/edgar/data/1660134/000119312521067561/d149091d8k.htm)) and completed it on 3 May 2021 ([Form 8-K](https://www.sec.gov/Archives/edgar/data/1660134/000119312521146823/d133101d8k.htm)).

The table compares the standard with the product:

| | OAuth 2.0 | Auth0 |
| --- | --- | --- |
| What it is | A specification, RFC 6749, published in October 2012 | A hosted service |
| Who makes it | The IETF | Okta |
| What you get | Rules for roles, requests, and responses | A tenant with users, applications, APIs, and a token endpoint |
| How you use it | You write servers and clients that follow the rules | You create an API and an application in the console, then call `POST https://{yourDomain}/oauth/token` |
| What it leaves open | The token format and the scope strings | Auth0 decides: `audience` names the API, and a token for a custom API lasts [86400 seconds by default](https://auth0.com/docs/secure/tokens/access-tokens) |

Auth0 has four application types: regular web, single-page, native, and machine-to-machine ([applications](https://auth0.com/docs/get-started/applications)). A program uses the machine-to-machine type. Client Credentials Flow shows Auth0 as the authorization server for an Iceberg catalog.

### Okta {#okta}

> type: concept

Okta Workforce Identity is a platform that secures the employees, contractors, and partners of an organization ([Okta](https://www.okta.com/products/workforce-identity/)). Every Okta org, which is the Okta account of one organization, has a built-in org authorization server. It issues access tokens for Okta's own resources. A custom authorization server issues tokens for your own APIs, and you define its scopes, claims, and access policies. In production, the custom server is an optional add-on ([authorization servers](https://developer.okta.com/docs/concepts/auth-servers/)).

A program uses an app integration with the sign-in method API Services ([client credentials guide](https://developer.okta.com/docs/guides/implement-grant-type/clientcreds/main/)). To call Okta's own APIs, a service app must sign in with a signed JWT. Okta names this method `private_key_jwt` ([service app guide](https://developer.okta.com/docs/guides/implement-oauth-for-okta-serviceapp/main/)).

### Microsoft Entra ID {#microsoft-entra-id}

> type: concept

> edge.related: platform-identity

Microsoft Entra ID is a cloud service for identity and access management ([Microsoft Learn](https://learn.microsoft.com/en-us/entra/fundamentals/what-is-entra#microsoft-entra-id)). Microsoft announced on 11 July 2023 that Azure Active Directory is now Microsoft Entra ID ([name change](https://learn.microsoft.com/en-us/entra/fundamentals/new-name)). An app must be registered with a tenant. The registration creates the application object, the global template. In each tenant where the app runs, a service principal is its local representation ([apps and service principals](https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals)).

In the client credentials flow, an app signs in with a shared secret, a certificate, or a federated credential ([client credentials flow](https://learn.microsoft.com/en-us/entra/identity-platform/v2-oauth2-client-creds-grant-flow)). A federated credential comes from another identity provider. A managed identity gives an Azure resource an Entra identity with no credentials to manage ([managed identities](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview)).

### Keycloak {#keycloak}

> type: concept

Keycloak is open-source identity and access management software ([keycloak.org](https://www.keycloak.org/)). It supports OpenID Connect, OAuth 2.0, and SAML, and its license is Apache 2.0 ([LICENSE](https://github.com/keycloak/keycloak/blob/main/LICENSE.txt)). A realm manages a set of users, credentials, roles, and groups. Realms are isolated from each other ([server administration guide](https://www.keycloak.org/docs/latest/server_admin/#core-concepts-and-terms)). A client with Client authentication and Service accounts roles enabled can use the client credentials grant. The service account of that client must hold the roles that the client needs.

### Amazon Cognito {#amazon-cognito}

> type: concept

Amazon Cognito is an identity platform for web and mobile apps. It combines a user directory and an authentication server. It also authorizes OAuth 2.0 access tokens and AWS credentials ([What is Amazon Cognito?](https://docs.aws.amazon.com/cognito/latest/developerguide/what-is-amazon-cognito.html)). A user pool is a user directory that follows OpenID Connect and issues JWTs to your app or API. An identity pool decides which AWS resources a user may reach. A program uses a confidential app client with a client secret and the client credentials grant. The token authorizes only the custom scopes of a resource server ([machine-to-machine authorization](https://docs.aws.amazon.com/cognito/latest/developerguide/cognito-user-pools-define-resource-servers.html)).

## Programs and Service Accounts {#programs-and-service-accounts}

> type: category

A program cannot type a password or approve a prompt on a phone. It signs in with a secret or a private key that it holds, or with an identity that its platform gives it. Service Account Names says what each platform calls such an identity. Client Credentials Flow shows the usual exchange of a credential for a short-lived token. Stored Secret, Private Key, and Platform Identity describe the three ways that a program proves who it is. Choosing a Credential compares them.

### Service Account Names {#service-account-names}

> type: concept

> edge.related: principal

The identity of a program has a different name on each platform:

| Where | Name | What it is |
| --- | --- | --- |
| Snowflake | Service user | A user with `TYPE = SERVICE`: a service or application that interacts with Snowflake without human interaction. It cannot log in with a password or enroll in MFA ([user types](https://docs.snowflake.com/en/user-guide/admin-user-management#types-of-users)). |
| Snowflake Open Catalog | Service principal | A user with `TYPE = service` ([External OAuth setup](https://docs.snowflake.com/en/user-guide/opencatalog/external-oauth-configure)). |
| Google Cloud | Service account | A special kind of account that an application or compute workload uses, identified by its email address ([service accounts](https://cloud.google.com/iam/docs/service-account-overview#what-are-service-accounts)). |
| AWS | IAM role | An identity with specific permissions and no long-term credentials. Assuming it gives temporary credentials ([IAM roles](https://docs.aws.amazon.com/IAM/latest/UserGuide/id_roles.html)). |
| Microsoft Entra ID | Service principal, managed identity | The local representation of an app in a tenant, and an identity for an Azure resource ([apps and service principals](https://learn.microsoft.com/en-us/entra/identity-platform/app-objects-and-service-principals), [managed identities](https://learn.microsoft.com/en-us/entra/identity/managed-identities-azure-resources/overview)). |
| Kubernetes | ServiceAccount | A non-human account that provides a distinct identity in a cluster ([service accounts](https://kubernetes.io/docs/concepts/security/service-accounts/#what-are-service-accounts)). |
| OAuth 2.0 | Confidential client | A client that can keep its credentials secret ([RFC 6749](https://www.rfc-editor.org/rfc/rfc6749.html#section-2.1)). |
| Auth0 | Machine-to-machine application | A non-interactive application, such as a command-line tool, a daemon, or a backend service ([applications](https://auth0.com/docs/get-started/applications)). |

Okta, Keycloak, and Amazon Cognito use the same idea. They call it an API Services app, a client with service accounts, and a confidential app client.

### Client Credentials Flow {#client-credentials-flow}

> type: concept

> edge.related: oauth-2-0

A program uses the client credentials grant to get an access token for itself ([RFC 6749](https://www.rfc-editor.org/rfc/rfc6749.html#section-4.4)). Only confidential clients may use it. The picture shows the five messages. The response holds no refresh token, so the program repeats message 1 when the token expires ([RFC 6749](https://www.rfc-editor.org/rfc/rfc6749.html#section-4.4.3)). In message 4, the API checks the signature, the issuer, the audience, and the expiry ([RFC 9068](https://www.rfc-editor.org/rfc/rfc9068.html#section-4)). It finds the public keys in the `jwks_uri` of the issuer's metadata.

![A program sends its client ID and client secret to the authorization server, receives an access token, and calls an API with it; the API checks the token](authentication-d2-diagram/client-credentials-flow.d2.svg)

Message 1 can prove the identity of the client in more than one way. On 2026-11-15, the discovery document of Microsoft Entra ID listed `client_secret_post`, `private_key_jwt`, `client_secret_basic`, and `self_signed_tls_client_auth` as token endpoint authentication methods. A secret, a signed JWT, and a certificate are three of these ways. Stored Secret, Private Key, and Mutual TLS describe them.

Snowflake signs in to an Iceberg catalog with this flow. The catalog integration holds `OAUTH_CLIENT_ID`, `OAUTH_CLIENT_SECRET`, and `OAUTH_ALLOWED_SCOPES`. They become the fields `client_id`, `client_secret`, and `scope` of message 1. Snowflake's documentation shows a test request. It sends `grant_type=client_credentials` and `scope=PRINCIPAL_ROLE:ALL` to `https://<account>.snowflakecomputing.com/polaris/api/catalog/v1/oauth/tokens`. The response holds `"token_type": "bearer"` and `"expires_in": 3600` ([check a configuration for OAuth](https://docs.snowflake.com/en/user-guide/tables-iceberg-configure-catalog-integration-rest-check-config)). The [Iceberg in Snowflake map](../snowflake/iceberg/snowflake-iceberg.md) explains the principals and roles behind the client ID. The Iceberg REST specification marks its own token endpoint as deprecated for removal. It encourages clients to set `oauth2-server-uri` to an OAuth server ([specification](https://github.com/apache/iceberg/blob/apache-iceberg-1.12.0/open-api/rest-catalog-open-api.yaml#L190-L197)).

The token server can be another product. The setting `OAUTH_TOKEN_URI` names the token server of a third-party identity provider. Without it, Snowflake uses the catalog as the token server ([CREATE CATALOG INTEGRATION](https://docs.snowflake.com/en/sql-reference/sql/create-catalog-integration-open-catalog)). Snowflake's guide for Open Catalog shows Auth0 in that role. The steps for Okta and Microsoft Entra ID are similar ([External OAuth setup](https://docs.snowflake.com/en/user-guide/opencatalog/external-oauth-configure)):

- In Auth0, you define an API. Its identifier is the audience. You add permissions named `SESSION:ROLE:<role>`, and you create an application. The client ID of that application is the identity of the program.
- In Open Catalog, a security integration with `TYPE = EXTERNAL_OAUTH` names the issuer `https://<Auth0_domain>/`, the key URL `https://<Auth0_domain>/.well-known/jwks.json`, and the audience. The claim `sub` maps to the login name of a user.
- The service principal is a user with `LOGIN_NAME='<client_id>@clients'` and `TYPE='service'`.
- The program asks Auth0 for a token: `POST https://<Auth0_domain>/oauth/token` with `grant_type=client_credentials`, `client_id`, `client_secret`, `audience`, and `scope=SESSION:ROLE:<role>`.
- A Spark client takes the same values as Iceberg catalog properties: `rest.auth.type=oauth2`, `oauth2-server-uri`, `credential`, `scope`, and `audience` ([connect with External OAuth](https://docs.snowflake.com/en/user-guide/opencatalog/external-oauth-connect)).

### Stored Secret {#stored-secret}

> type: concept

> edge.related: api-key

A stored secret is a string that the program keeps and sends. Examples are a password, an API key, a client secret, and a token. It is the simplest credential. The Snowflake programmatic access token (PAT) is an example. It is a time-limited credential that lets an application authenticate without a password ([authentication overview](https://docs.snowflake.com/en/user-guide/security-authentication-overview)). This statement creates one for a service user ([PAT guide](https://docs.snowflake.com/en/user-guide/programmatic-access-tokens)):

`ALTER USER IF EXISTS example_service_user ADD PROGRAMMATIC ACCESS TOKEN example_service_user_token ROLE_RESTRICTION = 'example_service_user_role';`

A PAT expires after 15 days by default and after 365 days at most. A service user must name the role that the sessions of the token use, and must be subject to a network policy. Snowflake shows the secret once, when it creates the token. The program sends the secret with each request: `Authorization: Bearer <token_secret>`.

Snowflake is phasing out passwords for service users. A user with `TYPE = SERVICE` cannot log in with a password. The deprecated type `LEGACY_SERVICE` still allows one. The schedule gives August to October 2026 for strong authentication for all users, and the dates vary by account ([timeline](https://docs.snowflake.com/en/user-guide/security-mfa-rollout)).

#### Leaked Secret {#leaked-secret}

> type: warning

A stored secret works for anyone who finds it. Snowflake's comparison table says that anyone who holds a compromised PAT can impersonate the application. GitHub's secret scanner detects leaked Snowflake PATs in public repositories, disables them, and notifies Snowflake administrators ([choosing authentication](https://docs.snowflake.com/en/user-guide/security-authentication-overview#choosing-authentication-for-service-to-service-applications)).

### Private Key {#private-key}

> type: concept

> edge.related: jwt

With a private key, a program proves who it is by signing a message. The service keeps only the matching public key. The private key stays with the program. Only signatures leave it. The signed message is a JWT, and the service checks its signature with the public key. [RFC 7523](https://www.rfc-editor.org/rfc/rfc7523.html#section-2.1) defines the use of a JWT as an OAuth 2.0 grant or as client authentication.

Snowflake calls this key-pair authentication. An administrator assigns the public key to the user ([key-pair authentication](https://docs.snowflake.com/en/user-guide/key-pair-auth)):

`ALTER USER example_user ADD KEY PAIR my_key PUBLIC_KEY = 'MIIBIjANBgkqh...';`

The client then builds a JWT ([SQL API authentication](https://docs.snowflake.com/en/developer-guide/sql-api/authenticating)). The claim `iss` is `<account>.<user>.SHA256:<fingerprint>`, and `sub` is `<account>.<user>`. The claims `iat` and `exp` give the time of issue and the time of expiry. The token is valid for one hour at most. The fingerprint is the SHA-256 hash of the public key in DER (Distinguished Encoding Rules) form, written in base64 with the prefix `SHA256:`. The client sends `Authorization: Bearer <JWT>`, or sets `authenticator='SNOWFLAKE_JWT'` in the [Python connector](https://docs.snowflake.com/en/developer-guide/python-connector/python-connector-connect). About the Examples builds such a JWT with a new 2048-bit key. The fingerprint has 51 characters, and the RS256 signature has 256 bytes, which is 342 characters in base64url.

Other services use the same idea:

- A Google service account signs a JWT with `aud` set to `https://oauth2.googleapis.com/token`. It sends the JWT with `grant_type=urn:ietf:params:oauth:grant-type:jwt-bearer`. The JWT lasts one hour at most ([Google](https://developers.google.com/identity/protocols/oauth2/service-account)).
- Okta requires a signed JWT, which it names `private_key_jwt`, from a service app that gets tokens with Okta scopes ([service app guide](https://developer.okta.com/docs/guides/implement-oauth-for-okta-serviceapp/main/)).
- Snowflake Open Catalog accepts a key-pair JWT in the `client_secret` field of its token request ([key pair authentication](https://docs.snowflake.com/en/user-guide/opencatalog/key-pair-auth-configure)).

A Snowflake user can hold up to two public keys at once, `RSA_PUBLIC_KEY` and `RSA_PUBLIC_KEY_2`, for key rotation ([key-pair rotation](https://docs.snowflake.com/en/user-guide/key-pair-auth#configuring-key-pair-rotation)).

### Platform Identity {#platform-identity}

> type: concept

> edge.related: trust-and-federation

The platform that runs a program can give it an identity. Then the program stores no secret, and the platform issues short-lived credentials on request:

- AWS: an IAM role has no long-term credentials. A session lasts 3600 seconds by default ([AssumeRole](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRole.html)). Code on an Amazon EC2 (Elastic Compute Cloud) instance reads the credentials of its role from the instance metadata service ([instance metadata](https://docs.aws.amazon.com/AWSEC2/latest/UserGuide/instance-metadata-security-credentials.html)). Amazon ECS (Elastic Container Service) sets the variable `AWS_CONTAINER_CREDENTIALS_RELATIVE_URI` for the same purpose ([container credentials](https://docs.aws.amazon.com/sdkref/latest/guide/feature-container-credentials.html)). AWS Lambda assumes the execution role of a function when it runs ([execution role](https://docs.aws.amazon.com/lambda/latest/dg/lambda-intro-execution-role.html)).
- Google Cloud: you attach a service account to a resource. Code on the resource uses the account as its identity ([attach service accounts](https://cloud.google.com/iam/docs/attach-service-accounts)). Application Default Credentials checks three places in order: the variable `GOOGLE_APPLICATION_CREDENTIALS`, a file from `gcloud auth application-default login`, and the attached service account ([search order](https://cloud.google.com/docs/authentication/application-default-credentials#order)).
- Microsoft Azure: a managed identity gives an Azure resource an Entra identity with no credentials to manage. Microsoft Entra ID describes it.
- Kubernetes: a pod, which is the unit in which Kubernetes runs containers, gets a signed token for its ServiceAccount in a projected volume. The token has an audience, which defaults to the API server. Its `expirationSeconds` defaults to 1 hour and must be at least 10 minutes ([projected volumes](https://kubernetes.io/docs/concepts/storage/projected-volumes/#serviceaccounttoken)).

#### Workload Identity Federation {#workload-identity-federation}

> type: concept

> edge.related: openid-connect

Workload identity federation lets a service outside the platform trust the tokens of the platform. The platform publishes its public keys, and the outside service checks each token with them. Kubernetes publishes its configuration at `/.well-known/openid-configuration` and its keys at `/openid/v1/jwks` ([issuer discovery](https://kubernetes.io/docs/tasks/configure-pod-container/configure-service-account/#service-account-issuer-discovery)).

![A pod receives a signed token from Kubernetes and connects to Snowflake with it; Snowflake fetches the keys of the issuer and checks the token](authentication-d2-diagram/workload-identity-federation.d2.svg)

Snowflake supports this as workload identity federation (WIF), with AWS, Azure, Google Cloud, and OIDC issuers as providers ([WIF guide](https://docs.snowflake.com/en/user-guide/workload-identity-federation)). The feature became generally available on 14 August 2025 ([release note](https://docs.snowflake.com/en/release-notes/2025/other/2025-08-14-wif)). For a pod on Amazon EKS (Elastic Kubernetes Service), an administrator creates a service user that names the issuer and the subject of the token:

`CREATE USER my_eks_service WORKLOAD_IDENTITY = (TYPE = OIDC ISSUER = 'https://oidc.eks.<region>.amazonaws.com/id/<issuer_id>' SUBJECT = 'system:serviceaccount:<namespace>:<service_account>') TYPE = SERVICE;`

The Python driver reads the token from the projected file: `snowflake.connector.connect(account='<snowflake_account>', authenticator='WORKLOAD_IDENTITY', workload_identity_provider='OIDC', token_file_path='/var/run/secrets/tokens/snowflake-token')`. Snowflake gets the keys of an OIDC issuer from its discovery document and its `jwks_uri`, which must be publicly reachable ([OIDC provider setup](https://docs.snowflake.com/en/user-guide/workload-identity-federation#configure-your-oidc-provider)). The token must carry the claim `iat`. Snowflake accepts the signature algorithms RS256, RS384, RS512, ES256, ES384, and ES512. It caches the keys of the issuer for up to 60 minutes ([limitations](https://docs.snowflake.com/en/user-guide/workload-identity-federation#limitations-and-considerations)).

A continuous integration (CI) job can do the same. GitHub Actions gives a job a token when the job has the permission `id-token: write`. The job requests the token from the URL in `ACTIONS_ID_TOKEN_REQUEST_URL`. The issuer is `https://token.actions.githubusercontent.com`, and the `sub` claim looks like `repo:octo-org/octo-repo:ref:refs/heads/main` ([GitHub OIDC reference](https://docs.github.com/en/actions/reference/security/oidc)). An AWS role trusts the token when its trust policy tests the claims `aud` and `sub`. The call `AssumeRoleWithWebIdentity` then returns temporary credentials, and it needs no AWS credentials of its own ([AssumeRoleWithWebIdentity](https://docs.aws.amazon.com/STS/latest/APIReference/API_AssumeRoleWithWebIdentity.html), [GitHub guide for AWS](https://docs.github.com/en/actions/how-tos/secure-your-work/security-harden-deployments/oidc-in-aws)).

### Choosing a Credential {#choosing-a-credential}

> type: concept

Snowflake compares its methods for service-to-service use in a table. The table calls workload identity federation the preferred option, because it needs no secret. It calls External OAuth a strong option. For key-pair authentication and programmatic access tokens, it names the risks of long-lived credentials ([choosing authentication](https://docs.snowflake.com/en/user-guide/security-authentication-overview#choosing-authentication-for-service-to-service-applications)).

| Method | What the program sends | How long it lasts | Snowflake's remark |
| --- | --- | --- | --- |
| Platform identity | A token that the platform issued | A Kubernetes token: 1 hour by default | Preferred option. Secretless. |
| External OAuth | An access token, which the program gets with a secret or a key | An Auth0 token: 86400 seconds by default. An Okta sample: 3600 seconds. | Strong option. |
| Private key | A JWT that the program signs | 1 hour at most | The credential is not exposed in a request. |
| Programmatic access token | The secret itself | 15 days by default, 365 days at most | The secret travels in each request. |

AWS and Google give the same advice for their platforms. AWS recommends temporary credentials from IAM roles in place of long-term credentials such as access keys ([best practices](https://docs.aws.amazon.com/IAM/latest/UserGuide/best-practices.html#bp-workloads-use-roles)). Google says that service account keys are a security risk if you manage them badly. It advises a more secure alternative whenever possible ([service accounts](https://cloud.google.com/iam/docs/service-account-overview#what-are-service-accounts)).

## About the Examples {#about-the-examples}

> type: comment

The values in this map come from three places. The first place is the examples in the RFCs and in the vendor documentation, which are linked where they appear. The second place is public pages that anyone can fetch. The map used the pages that these commands return, read on 2026-11-15:

```bash
curl -i https://api.github.com/user
curl https://accounts.google.com/.well-known/openid-configuration
curl https://login.microsoftonline.com/common/v2.0/.well-known/openid-configuration
curl https://token.actions.githubusercontent.com/.well-known/openid-configuration
curl https://token.actions.githubusercontent.com/.well-known/jwks
```

The third place is this script. It ran with Python 3.12 and the `cryptography` package 50.0.2:

```python
import base64
import hashlib
import hmac
import json

from cryptography.hazmat.primitives import hashes, serialization
from cryptography.hazmat.primitives.asymmetric import padding, rsa


def b64url(raw):
    return base64.urlsafe_b64encode(raw).rstrip(b"=").decode()


def b64url_decode(text):
    return base64.urlsafe_b64decode(text + "=" * (-len(text) % 4))


def part(obj):
    return b64url(json.dumps(obj, separators=(",", ":")).encode())


claims = {
    "iss": "https://idp.example.com",
    "sub": "svc-loader",
    "aud": "https://api.example.com",
    "iat": 1794700800,
    "exp": 1794704400,
}
signing_input = part({"alg": "HS256", "typ": "JWT"}) + "." + part(claims)
mac = hmac.new(b"shared-secret", signing_input.encode(), hashlib.sha256).digest()
token = signing_input + "." + b64url(mac)
print(token)
for piece in token.split(".")[:2]:
    print(json.loads(b64url_decode(piece)))
again = hmac.new(b"shared-secret", signing_input.encode(), hashlib.sha256).digest()
print("valid:", hmac.compare_digest(again, b64url_decode(token.split(".")[2])))

key = rsa.generate_private_key(public_exponent=65537, key_size=2048)
public_der = key.public_key().public_bytes(
    serialization.Encoding.DER, serialization.PublicFormat.SubjectPublicKeyInfo
)
fingerprint = "SHA256:" + base64.b64encode(hashlib.sha256(public_der).digest()).decode()
account, user = "MYORG-MYACCOUNT", "SVC_LOADER"
claims = {
    "iss": f"{account}.{user}.{fingerprint}",
    "sub": f"{account}.{user}",
    "iat": 1794700800,
    "exp": 1794704400,
}
signing_input = part({"alg": "RS256", "typ": "JWT"}) + "." + part(claims)
signature = key.sign(signing_input.encode(), padding.PKCS1v15(), hashes.SHA256())
key.public_key().verify(signature, signing_input.encode(), padding.PKCS1v15(), hashes.SHA256())
print(len(fingerprint), "characters in", fingerprint)
print(json.dumps(claims, indent=1))
print(len(signature), "signature bytes =", len(b64url(signature)), "characters")
```

The first part prints the JWT in the JSON Web Token node, and it prints the same token on every run. The second part makes a new 2048-bit key on every run. The fingerprint and the signature change with each run, and their lengths stay the same. The time 1794700800 is 2026-11-15 00:00:00 UTC.
