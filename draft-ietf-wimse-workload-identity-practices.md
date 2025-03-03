---
title: Workload Identity Practices
abbrev: Workload Identity
docname: draft-ietf-wimse-workload-identity-practices-latest
category: info

ipr: trust200902
area: "Applications and Real-Time"
workgroup: "Workload Identity in Multi System Environments"
keyword: Internet-Draft

stand_alone: yes
pi:
  rfcedstyle: yes
  toc: yes
  tocindent: yes
  sortrefs: yes
  symrefs: yes
  strict: yes
  comments: yes
  inline: yes
  text-list-symbols: -o*+
  docmapping: yes

author:
 -
      ins: A. Schwenkschuster
      name: Arndt Schwenkschuster
      email: arndts.ietf@gmail.com
      org: SPIRL
      role: editor

 -
      ins: B. Hofmann
      name: Benedikt Hofmann
      email: hofmann.benedikt@siemens.com
      org: Siemens

 -
      ins: H. Tschofenig
      name: Hannes Tschofenig
      email: hannes.tschofenig@gmx.net
      org: Siemens

 -
      ins: E. Giordano
      name: Edoardo Giordano
      email: edoardo.giordano@nokia.com
      org: Nokia

 -
      ins: Y. Rosomakho
      name: Yaroslav Rosomakho
      email: yrosomakho@zscaler.com
      org: Zscaler

normative:
  RFC2119:
  RFC7521:
  RFC7523:
  RFC6749:
  RFC8174:
  RFC8414:
  RFC7519:
  RFC7517:
  RFC8707:
  RFC8725:
  RFC8414:
  RFC8693:
informative:
  I-D.ietf-oauth-rfc7523bis:
  I-D.ietf-wimse-arch:
  I-D.ietf-wimse-s2s-protocol:
  OIDC:
     author:
       org: Sakimura, N., Bradley, J., Jones, M., de Medeiros, B., and C. Mortimore
     title: OpenID Connect Core 1.0 incorporating errata set 1
     target: https://openid.net/specs/openid-connect-core-1_0.html
     date: 8 November 2014
  OIDCDiscovery:
    author:
      org: Sakimura, N., Bradley, J., Jones, M. and Jay, E.
    title: OpenID Connect Discovery 1.0 incorporating errata set 2
    target: https://openid.net/specs/openid-connect-discovery-1_0.html
    date: 15 December 2023
  KubernetesServiceAccount:
     title: Kubernetes Service Account
     target: https://kubernetes.io/docs/concepts/security/service-accounts/
     date: 10 May 2024
  TokenReviewV1:
     title: Kubernetes Token Review API V1
     target: https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-review-v1/
     date: 28 August 2024
  TokenRequestV1:
     title: Kubernetes Token Request API V1
     target: https://kubernetes.io/docs/reference/kubernetes-api/authentication-resources/token-request-v1/
     date: 28 August 2024

--- abstract

The use of the OAuth 2.0 framework in container orchestration systems poses challenges, particularly in managing credentials such as client_id and client_secret, which can be complex and prone to errors. To address this, the industry has shifted towards a federation-based approach where credentials of the underlying workload platform are used as assertions towards an OAuth authorization server, leveraging the Assertion Framework for OAuth 2.0 Client Authentication {{RFC7521}}, specifically {{RFC7523}}.

This specification describes a meta flow in {{overview}}, gives security recommendations in {{recommendations}} and outlines concrete patterns in {{patterns}}. It referes to existing industry practices that are mainly built on top of OAuth. It may not be in line with the (currently work in progress) WIMSE architecture {{I-D.ietf-wimse-arch}} and other protocols, such as {{I-D.ietf-wimse-s2s-protocol}}.

--- middle

# Introduction

Workloads often require access to external resources to perform their tasks. For example, access to a database, a web server or another workload. These resources are protected by an authorization server and can only be accessed with an access token. The challenge for workloads is to get this access token issued.

Traditionally, workloads were provisioned with client credentials and use the corresponding client credential flow (Section 1.3.4 {{RFC6749}}) to retrieve an access token. This model comes with a set of challenges that make it insecure and high-maintenance. Secret materials need to be provisioned and rotated, which often happens manually. It also can be stolen and used by attackers to impersonate the workload.

A solution to this problem is to not provision secret material to the workload and use the platform the workload runs on to attest for it. Many workload platforms offer a credential, in most cases a JWT token. Signed by a platform-internal authorization server, the credential attests the workload and its attributes. Based on {{RFC7521}} and its JWT profile {{RFC7523}}, this credential can then be used as a client assertion towards a different authorization server.

# Terminology

The key words "MUST", "MUST NOT", "REQUIRED", "SHALL", "SHALL NOT",
"SHOULD", "SHOULD NOT", "RECOMMENDED", "NOT RECOMMENDED", "MAY", and
"OPTIONAL" in this document are to be interpreted as described in
BCP 14 {{RFC2119}} {{RFC8174}} when, and only when, they appear in all
capitals, as shown here.

For the scope of this specification, the term authorization server is only used for the authorization server of the external authorization domain as highlighted in {{fig-overview}}.

Even though, technically, the platform credential is also issued by an authorization server within the workload platform, this specification only refers to it as "platform issuer" or just "platform".

# OAuth Assertion in Workload Environments

## Overview

{{fig-overview}} illustrates a generic pattern that applies across all of the patterns described in {{patterns}}:

~~~ aasvg
 +----------------------------------------------------------+
 |                           External Authorization Domain  |
 |                                                          |
 | +-------------------------+     +--------------------+   |
 | |                         |     |                    |   |
 | |   Authorization Server  |     | Protected Resource |   |
 | |                         |     |                    |   |
 | +-------^---------+-------+     +----------^---------+   |
 +---------+---------+------------------------+-------------+
           |         |                        |
           |   3) access token           4) access
           |         |                        |
2) present assertion |                        |
           |         |     +------------------+
           |         |     |
 +---------+---------+-----+--------------------------------+
 |         |         |     |             Workload Platform  |
 |         |         |     |                                |
 |  +------+---------v-----+---+           +-------------+  |
 |  |                          |           | Credential  |  |
 |  |        Workload          <-----------> issued by   |  |
 |  |                          | 1) pull/  | Platform    |  |
 |  +--------------------------+    push   +-------------+  |
 +----------------------------------------------------------+
~~~
{: #fig-overview title="OAuth2 Assertion Flow in generic Workload Environment"}

The figure outlines the following steps which are applicable in any pattern.

* 1) retrieve credential issued by platform. The way this is achieved and whether this is workload (pull) or platform (push) initiated differs based on the platform.

* 2) present credential as an assertion towards an authorization server in an external authorization domain. This step uses the assertion_grant flow defined in {{RFC7521}} and, in case of JWT format, {{RFC7523}}.

* 3) On success, an access token is returned to the workload to access the protected resource.

* 4) The access token is used to access a protected resource in the external authorization domain. For instance by making a HTTP call.

Accessing different protected resources may require steps 2) to 4) again with different scope parameters. Accessing a protected resource in an entirely different authorization domain often requires the entire flow to be followed again, to retrieve a new platform-issued credential with an audience for the external authorization server. This, however, differs based on the platform and implementation.

## Credential issuance

Credentials can be provisioned to the workload through different mechanisms which present different complexity and security risks. The following section highlights the pros and cons of most widespread solutions.

### Environment variable

Injecting the credentials into the environmental variables allows for simple and fast deployments. Applications can directly access them through system-level mechanism, e.g., through the env command in linux. This flexibility, however, comes with security drawbacks.

* 1) environmental variables are static in nature and more dynamic solutions require to introduces higher complexity.

* 2) performing access control to environmental variable is not trivial and it might also not reach the same security results.

* 3) environmental variables have a wide set of use cases and are observed by many components. They are often captured for monitoring, observability, debugging and logging purposes and send to components outside of the workload.

Leveraging environmental variables to provide credentials presents many security limitations. This approach should be limited to cases where simplicity of the application is required, e.g., during PoCs, and the provided secrets should have a short-term validity, i.e., an initial secret during the set-up of the application.

### Files

Files allow to inject credentials into a container through the file-system and access them through primitives, e.g., Open, Close, etc. Such solutions enable secret rotation, and access control on the injected secret with standard operating system measure for files.

* 1) access control to the mounted file should be configure to limit access from unauthorized applications. E.g., on Linux solutions such as DAC (uid and guid) or MAC (SELinux,AppArmor) are available.

* 2) isolating the mounted shared memory from critical host OS paths and processes is required. E.g, on Linux this is achieved by utilising namespaces.

* 3) credentials rotation requires a solution to detect near-to-expiration secrets and substitute them. Solutions should enable configuration such that the new secret is renewed _before_ the old secret is invalidated. E.g., the solution can choose to update the secret 1h before the old secret is invalidated. This enables applications time to update their usage of the old secret to the new without downtime.

Volume solutions find their main benefit in the asynchronous provisioning of the credentials to the workload. This allows the workload to run independently of the credentials update, and to access them by reading the file, which path can be provisioned through environmental variables, only when required.

### Local APIs

This set of solution rely on local APIs to communicate between the Host and the containerised application. Some implementations rely on UNIX Domain Sockets (SPIFFE), loopback interfaces or Magic (Link-Local) Addresses (AWS Metadata service) to provision credentials. Local APIs offer the capability to re-provision updated credentials. Communication between workload and API allows the workload to re-request a new credential (or a different one). Based on the technology it is even possible to pro-actively distribute new credentials to workloads (e.g. upon expiry, during a revocation event or a change in permissions). This group of solutions rely on network isolation for their security.

* 1) credentials are only issued when request, thus reducing unnecessary credential issuance and allowing for a narrowly-scoped and short-lived secrets. For instance one-time credentials or narrowly scoped ones such as JSON Web Tokens with specific audiences.

* 2) the benefit of a more structured delivery comes with less portability between different APIs.

* 3) additional latency may be introduced due to the request and additional operational overhead.

Local APIs offer the highest level of access control to protect the credential from un-legitimate access. They particularly thrive in environments of short-lived, narrowly scoped credentials, but come with operational overhead if the workload platform does not offer it already.

## Credential format

For the scope of this document we focus on JSON Web Token credential formats. Other formats such as X.509 certificates are possible but not as widely seen as JSON Web Tokens.

The claims in the present assertion vary greatly based on use case and actual platform, but a minimum set of claims are seen across all of them. {{RFC7523}} describes them in detail and according to it, all MUST be present.

~~~json
{
  "iss": "https://example.org",
  "sub": "my-workload",
  "aud": "target-audience",
  "exp": 1729248124
}
~~~

For the scope of this specification, the claims can be described the following. Everything from {{RFC7523}} applies.

{:vspace}
iss
: The issuer of the workload platform. While this can have any format, it is important to highlight that many authorization servers leverage OpenID Connect Discovery {{OIDCDiscovery}} and/or OAuth 2.0 Authorization Server Metadata {{RFC8414}} to retrieve JSON Web Keys {{RFC7517}} for validation purposes.

sub
: Subject which identifies the workload within the domain/workload platform instance.

audience
: One or many audiences the platform issued credential is eligible for. This is crucial when presenting the credential as an assertion towards the external authorization server which MUST identify itself as an audience present in the assertion.

# Security Recommendations {#recommendations}

All security considerations in section 8 of {{RFC7521}} apply.

## Token typing

Issuers SHOULD strongly type the issued tokens to workload via the JOSE `typ` header and authorization servers SHOULD validate the value of it according to policy. See Section 3.1 of {{RFC8725}} for details on explicit typing.

Issuers SHOULD use `authorization-grant+jwt` as a `typ` value according to {{I-D.ietf-oauth-rfc7523bis}}. For broad support `JWT` or `JOSE` MAY be used by issuers and accepted by authorization servers but it's important to highlight that a wide range of tokens, meant for all sorts of purposes, use these values and would be accepted.

## Custom claims are important for context

Some platform issued credentials have custom claims that are vital for context and are required to be validated. For example in a continuous integration and deployment platform where a workload is scheduled for a GIT repository, the branch is crucial. A 'main' branch may be protected and considered trusted to federate to external authorization servers, other branches may not be and are not allowed to access protected resources.

Authorization servers that validate assertions SHOULD make use of these claims. Platform issuers SHOULD allow differentiation based on the subject claim alone.

## Token lifetime

Tokens SHOULD NOT exceed the lifetime of the workloads they represent. For example, a workload that has an expected lifetime of an hour should not receive a token valid for 2 hours or more.

For the scope of this specification, where a platform issued credential is used to authenticate to retrieve an access token for an external authorization domain, a short-lived credential is recommended.

## Workload lifecycle and invalidation

Platform issuers SHOULD invalidate those when the workload stops, pauses or ceases to exist. How these credentials are invalidated is not in scope of this specification.

## Proof of possession

Credentials SHOULD be bound to workloads and proof of possession SHOULD be performed when these credentials are used. This mitigates token theft. This proof of possession applies to the platform credential and the access token of the external authorization domains.

## Audience

For issued credentials in the form of JWTs, they MUST be audienced using the `aud` claim. Each JWT SHOULD only carry a single audience. We RECOMMEND using URIs to specify audiences. See section 3 of {{RFC8707}} for more details and security implications.

Some workload platforms provide credentials for interacting with their own APIs (e.g., Kubernetes). These credentials MUST NOT be used beyond the platform API. In the example of Kubernetes: A token used for anything else than the Kubernetes API itself MUST NOT carry the Kubernetes server in the `aud` claim.

# Other considerations

## Relation to OpenID Connect and its ID Token

The outlined pattern has been referred to as "OIDC" and respectively, the Workload Identity Token as "OIDC ID Token" defined in {{OIDC}}. The authors of this document want to highlight that this pattern is not related to OpenID Connect {{OIDC}} and the issued Workload Identity Tokens by the platforms are not "ID Tokens" in the sense of OIDC.

However, it is common practice for the authorization server to leverage {{OIDCDiscovery}} to retrieve the signing keys needed for token validation. The use of {{RFC8414}} or any other key distribution remain valid.

# IANA Considerations {#IANA}

This document does not require actions by IANA.

# Acknowledgements

Add your name here.

--- back

# Patterns {#patterns}

## Kubernetes {#kubernetes}

In Kubernetes, the primary concept of machine identity is implemented through "service accounts" {{KubernetesServiceAccount}}. These accounts can be explicitly created or a default one is automatically assigned. Service accounts utilize JSON Web Tokens (JWTs) {{RFC7519}} as their credential format, with these tokens being cryptographically signed by the Kubernetes Control Plane.

Service accounts serve multiple authentication purposes within the Kubernetes ecosystem. They are used to authenticate to Kubernetes APIs, between different workloads and to access external resources (which is particularly relevant to the scope of this document).

To programatically use service accounts, workloads can:

* use the Token Request API {{TokenReviewV1}} of the control plane

* have the token projected into the file system of the workload. This is commonly referred to as "projected service accout token".

Both options allow workloads to:

* specify a custom audience. Possible audiences can be restricted based on policy.

* specify a custom lifetime. Maximum lifetime can be restricted by policy.

* bind the token lifetime to an object lifecycle. This allows the token to be invalidated when the object is deleted. For example, when a Kubernetes Deployment is removed from the server. It is important to highlight, that invalidation is only in effect if the Token Review API {{TokenReviewV1}} of Kubernetes is used to validate the token.

To validate service account tokens, Kubernetes offers workloads to:

* make us of the Token Review API {{TokenReviewV1}}. This API introspects the token, makes sure it hasn't been invalidated and returns the claims.

* mount the public keys used to sign the tokens into the file system of the workload. This allows workloads to decentrally validate the tokens signature.

* Optionally, a JSON Web Key Set {{RFC7517}} is exposed via a webserver. This allows the Service Account Token to be validated outside of the cluster and without line of sight towards the actual Kubernetes Control Plane API.

~~~aasvg
+-------------------------------------------------------+
|                         External Authorization Domain |
|                                                       |
| +--------------------------+ +--------------------+   |
| |                          | |                    |   |
| |   Authorization Server   | | Protected Resource |   |
| |                          | |                    |   |
| +------^-------------+-----+ +----------^---------+   |
|        |             |                  |             |
+--------+-------------+------------------+-------------+
         |             |                  |
3) present assertion   |              5) access
         |             |                  |
         |       4) access token          |
         |             |                  |
+--------+-------------+------------------+-------------+
|        |             |     +------------+  Kubernetes |
|        |             |     |                  Cluster |
|    +---+-------------v-----+----+                     |
|    |                            |                     |
|    |    Workload                |                     |
|    |                            |                     |
|    +----^----------------^------+                     |
|         |                |                            |
|         |                |                            |
|    1) schedule     2) projected service               |
|         |             account token                   |
|         |                |                            |
|   +-----+----------------+-------------------+        |
|   |                                          |        |
|   |     Kubernetes Control Plane / kubelet   |        |
|   |                                          |        |
|   +------------------------------------------+        |
|                                                       |
+-------------------------------------------------------+
~~~
{: #fig-kubernetes title="OAuth2 Assertion Flow in a Kubernetes Workload Environment"}

The steps shown in {{fig-kubernetes}} are:

* 1) The Kubernetes Control Plane schedules the workload. This is much simplified and technically happens asynchronously.

* 2) The Kubernetes Control Plane projects the service account token into the workload. This step is also much simplified and technically happens alongside the scheduling with step 1.

* 3) Workloads present the projected service account token as a client assertion towards an external authorization server according to {{RFC7523}}.

* 4) On success, an access token is returned to the workload to access the protected resource.

* 5) The access token is used to access the protected resource in the external authorization domain.

As an example, the following JSON showcases the claims a Kubernetes Service Account token carries.

~~~json
{
  "aud": [  # matches the requested audiences, or the API server's default audiences when none are explicitly requested
    "https://kubernetes.default.svc"
  ],
  "exp": 1731613413,
  "iat": 1700077413,
  "iss": "https://kubernetes.default.svc",  # matches the first value passed to the --service-account-issuer flag
  "jti": "ea28ed49-2e11-4280-9ec5-bc3d1d84661a",  # ServiceAccountTokenJTI feature must be enabled for the claim to be present
  "kubernetes.io": {
    "namespace": "my-namespace",
    "node": {  # ServiceAccountTokenPodNodeInfo feature must be enabled for the API server to add this node reference claim
      "name": "127.0.0.1",
      "uid": "58456cb0-dd00-45ed-b797-5578fdceaced"
    },
    "pod": {
      "name": "my-workload-69cbfb9798-jv9gn",
      "uid": "778a530c-b3f4-47c0-9cd5-ab018fb64f33"
    },
    "serviceaccount": {
      "name": "my-workload",
      "uid": "a087d5a0-e1dd-43ec-93ac-f13d89cd13af"
    },
    "warnafter": 1700081020
  },
  "nbf": 1700077413,
  "sub": "system:serviceaccount:my-namespace:my-workload"
}
~~~
{: #fig-kubernetes-token title="Example Kubernetes Service Account Token claims"}

## Secure Production Identity Framework For Everyone (SPIFFE) {#spiffe}

Secure Production Identity Framework For Everyone, also known as SPIFFE, is a cloud native compute foundation (CNCF) adopted project which defines an API defined called "Workload API" to delivery machine identity to workloads. Workloads can retrieve either X509 based or JWT credentials without the need to authenticate making it very easy to use. How workloads authenticate on the API is not part of the specification. It is common to use platform metadata from the operating system and the workload platform for authentication on the Workload API.

For the scope of this document, the JWT formatted credential is the most relevant one. SPIFFE referres to it as "JWT-SVID" (JWT - SPIFFE Verifiable Identity Document).

Workloads are required to specify at least one audience when requesting a JWT-SVID from the Workload API.

To allow validation, SPIFFE offers

* to download a set JWK encoded public keys that can be used to validate JWT signatures. In SPIFFE this is referred to as the "JWT trust bundle".

* invoke a validation method on the Workload API to validate JWT-SVIDs

Additionally, many SPIFFE deployments choose to separately publish the signing keys as a JSON Web Key Set on a web server to allow validation where the Workload API is not available.

The following figure illustrates how a workload can use its JWT-SVID to access a protected resource outside of SPIFFE.

~~~aasvg
+---------------------------------------------------------+
|                           External Authorization Domain |
|   +-----------------------+   +----------------------+  |
|   |                       |   |                      |  |
|   | Authorization Server  |   |  Protected Resource  |  |
|   |                       |   |                      |  |
|   +-----^-----------------+   +--------^-------------+  |
+---------+------------+-----------------+----------------+
          |            |                 |
 2) present assertion  |             4) access
          |            |                 |
          |       3) access token        |
          |            |                 |
+---------+------------+-----------------+----------------+
|  +------+------------v-----------------+----+  Workload |
|  |                                          |  Platform |
|  |                  Workload                |           |
|  |                                          |           |
|  +---------------------+--------------------+           |
|                        |                                |
|                 1) get JWT-SVID                         |
|                        |                                |
|  +---------------------v--------------------+           |
|  |                                          |           |
|  |           SPIFFE Workload API            |           |
|  |                                          |           |
|  +------------------------------------------+           |
+---------------------------------------------------------+
~~~
{: #fig-spiffe title="OAuth2 Assertion Flow in a SPIFFE Environment"}

The steps shown in {{fig-spiffe}} are:

* 1) The workload request a JWT-SVID from the SPIFFE Workload API with an audience that identifies the external authorization server.

* 2) The workload presents the JWT-SVID as a client assertion in the assertion flow based on {{RFC7523}}.

* 3) On success, an access token is returned to the workload to access the protected resource.

* 4) The access token is used to access the protected resource in the external authorization domain.

The claims of a JWT-SVID for example look like this:

~~~json
{
  "aud": [
    "external-authorization-server"
  ],
  "exp": 1729087175,
  "iat": 1729086875,
  "sub": "spiffe://example.org/myservice"
}
~~~

TODO: write about "iss" in JWT-SVID.

## Cloud Providers {#cloudproviders}

Workload in cloud platforms can have any shape or form. Historically, virtual machines were the most common, with the introduction of containerization, hosted container environment or Kubernetes clusters were introduced, and lately, `serverless` functions are offered. Regardless of the actual workload packaging, distribution and runtime platform, all are in need of identity.

To create a common identity interface across cloud services and offerings, the pattern of an `Instance Metadata Endpoint` has been established by the biggest cloud providers. Next to the option for workloads to get metadata about themselves, it also allows them to receive identity. The credential types offered can vary. JWT, however, is the one that is common across all of them. The issued credential allows proof to anyone it is being presented to, that the workload platform has attested the workload and it can be considered authenticated.

Within a cloud provider the issued credential can often directly be used to access resources of any kind across the platform making integration between the services easy and `credential less`. While the term is technically misleading, from a user perspective, no credential needs to be issued, provisioned, rotated or revoked, as everything is handled internally by the platform.

Resources outside of the platform, for example resources or workloads in other clouds, generic web servers or on-premise resources, are most of the time, however, protected by different domains and authorization servers and deny the platform issued credential. In this scenario, the pattern of using the platform issued credential as an assertion in the context of {{RFC7521}}, for JWT particularly {{RFC7523}} towards the authorization server that protected the resource to get an access token.

~~~aasvg
   +-----------------------------------------------------+
   |                       External Authorization Domain |
   |                                                     |
   | +------------------------+  +---------------------+ |
   | |                        |  |                     | |
   | | Authorization Server   |  | Protected Resource  | |
   | |                        |  |                     | |
   | +------^------------+----+  +----------^----------+ |
   |        |            |                  |            |
   +--------+------------+------------------+------------+
            |            |                  |
B1) present as assertion |              B3) access
            |            |                  |
            |       B2) access token        |
            |            |   +--------------+
   +--------+------------+---+------------------------------+
   |        |            |   |                        Cloud |
   |        |            |   |                              |
   |   +----+------------v---+--+ 1) get       +----------+ |
   |   |                        |    identity  |          | |
   |   |        Workload        +--------------> Instance | |
   |   |                        |              |          | |
   |   +-----------+------------+              | Metadata | |
   |               |                           |          | |
   |           A1) access                      | Service/ | |
   |               |                           | Endpoint | |
   |   +-----------v------------+              |          | |
   |   |                        |              +----------+ |
   |   |   Protected Resource   |                           |
   |   |                        |                           |
   |   +------------------------+                           |
   +--------------------------------------------------------+
~~~
{: #fig-cloud title="OAuth2 Assertion Flow in a cloud environment"}

The steps shown in {{fig-cloud}} are:

* 1) The workload retrieves identity from the Instance Metadata Endpoint.

In case the workload needs to access a resource within the cloud (protected by the same authorization server that issued the workload identity)

* A1) The workload directly access the protected resource with the credential issued in step 1.

In case the workload needs to access a resource outside of the cloud (protected by a different authorization server). This can also be the same cloud but different context (tenant, account).

* B1) The workload presents cloud-issued credential as an assertion towards the external authorization server using {{RFC7523}}.

* B2) On success, an access token is returned to the workload to access the protected resource.

* B3) Using the access token, the workload is able to access the protected resource in the external authorization domain.

## Continuoues integration/deployment systems {#cicd}

Continuous integration and deployment systems allow their pipelines/workflows to receive identity every time they run. Particularly in situations where build outputs need to be uploaded to resources protected by other authorization server, deployments need to be made, or more generally, protected resources to be accessed, {{RFC7523}} is used to federate the pipeline/workflow identity to an identity of the other authorization server.

~~~aasvg
+----------------------------------------------------------+
|                            External Authorization Domain |
| +--------------------------+     +---------------------+ |
| |                          |     |                     | |
| |   Authorization Server   |     |  Protected Resource | |
| |                          |     |                     | |
| +-------^-------------+----+     +------------^--------+ |
|         |             |                       |          |
+---------+-------------+-----------------------+----------+
          |             |                       |
3) present assertion    |                  4) access
          |             |                       |
          |      4) access token                |
          |             |                       |
+---------+-------------v-----------------------+----------+
|                                                          |
|                    Task (Workload)                       |
|                                                          |
+--------^---------------------------^---------------------+
         |                           |
   1) schedules              2) retrieve identity
         |                           |
+--------+---------------------------v---------------------+
|                                                          |
|       Continuous Integration / Deployment Platform       |
|                                                          |
+----------------------------------------------------------+
~~~
{: #fig-cicd title="OAuth2 Assertion Flow in a continuous integration/deployment environment"}

The steps shown in {{fig-cicd}} are:

* 1) The continuous integration / deployment platform (CI-CD platform) schedules a task (considered a workload) to be performed.

* 2) The workload is able to retrieve identity from the CI-CD platform. This can differ based on the platform and potentially is already supplied during scheduling phase in step 1.

* 3) The workload presents the CI-CD issued credential as an assertion towards the authorization server in the external authorization domain based on {{RFC7521}}. In case of JWT also {{RFC7523}}.

* 4) On success, an access token is returned to the workload to access the protected resource.

* 5) Using the access token, the workload is able to access the protected resource in the external authorization domain.

Tokens of different providers look different, but all of them contain claims carrying the basic context of the executed tasks such as source code management data (e.g. git branch), initiation and more.

# Variations {#variations}

## Direct access to protected resources

Resource servers that protect resources may choose to trust multiple authorization servers, including the one that issues the platform identities. Instead of using the platform issued identity to receive an access token of a different authorization domain, workloads can directly use the platform issued identity to access a protected resource.

In this case, technically, the protected resource and workload are part of the same authorization domain.

## Custom assertion flows

While {{RFC7521}} and {{RFC7523}} are the proposed standards for this pattern, some authorization servers use {{RFC8693}} or a custom API for the issuance of an access token based on an existing platform identity credentials. These pattern are not recommended and prevent interoperability.

# Document History

   [[ To be removed from the final specification ]]

   -01

   * Add credential delivery mechanisms
   * Highlight relationship to other WIMSE work
   * Add details about token typing and relation to OpenID Connect
   * Add security considerations for audience

   -00

   * Rename draft with no content changes.
   * Set Arndt to Editor role.

__[as draft-wimse-workload-identity-bcp]__

   -02

   * Move scope from Kubernetes to generic workload identity platform
   * Add various patterns to appendix
      * Kubernetes
      * Cloud providers
      * SPIFFE
      * CI/CD
   * Add some security considerations
   * Update title

   -01

   * Editorial updates

   -00

   * Adopted by the WIMSE WG
