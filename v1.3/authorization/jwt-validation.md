---
lastmod: 2021-02-21
old_version: true
date: 2018-11-03
linktitle: JWT Validation
title: JWT Validation
description: Implement JWT validation with KrakenD API Gateway to secure your APIs and prevent unauthorized access.
weight: 20
source: https://github.com/krakend/krakend-jose
menu:
  community_v1.3:
    parent: "060 Authentication & Authorization"
---
The component `krakend-jose` is responsible for the JWT validation and **protects endpoints from public usage**, requiring end-users to provide a valid token to access its contents.

Before digging any further, some answers to frequently asked questions:

1) **KrakenD does not generate the tokens itself**. Still, you can plug it into any SaaS or self-hosted Identity Provider (**IdP**) using industry standards (e.g.: Auth0, Azure AD, Google Firebase, Keycloak, etc.)

2) **KrakenD does not need to validate all calls using your IdP**. KrakenD validates every incoming call's signature and **it doesn't make token introspection** (asking the IdP data about the token owner).

3) **If you don't have an identity server**, you can still use your classic monolith/backend login system and adapt it to return a JWT payload (which is a simple JSON). From here, let KrakenD [sign the token for you](/docs/v1.3/authorization/jwt-signing/) and start using tokens right away.

4) **Your self-hosted identity server doesn't need to be exposed to the Internet**, as it can live behind KrakenD and let the token generation requests be proxied through KrakenD. If you use a SaaS solution, of course, it's exposed.

## Key concepts
KrakenD uses the **JSON Web Token** specification (**JWT**), an industry-standard representing claims securely between two parties. A **JWT** is an encoded JSON object containing key-value pairs of attributes signed by a trusted authority. It carries the information your end-users pass to the system to be recognized as legitimate users with other metadata.

All tokens transmitted between users and KrakenD have to be signed using **JWS**, to make sure they are legitimate and not forged by an attacker. JWS represents digitally signed content using JSON data structures that are base64url encoded using the format `header.payload.signature`.

Finally, KrakenD needs to retrieve from the trusted authority (your Identity Provider) the keys that let the system validate the signature. These keys are transmitted between KrakenD and the IdP using the **JWK** format, a JSON object representing a set of cryptographic keys. Depending on the system and implementation you have in your IdP, objects will use one or another algorithm. **JWA** represents the set of algorithms you can use to sign your tokens.

The introduction above is very superficial; the recommended read is the RFC:

- **JWT**: [Definition of tokens: structure and composition of header and payload](https://tools.ietf.org/html/rfc7519)
- **JWS** [Signature](https://tools.ietf.org/html/rfc7515)
- **JWK** [Key transmission](https://tools.ietf.org/html/rfc7517)
- **JWA** [Definition of cyphering and signing algorithms](https://tools.ietf.org/html/rfc7518)
- **JWE** is not supported by KrakenD (Premise: Sensitive data should not be transmitted using tokens).

## JWT tokens definition
KrakenD uses **standard JWT tokens** to protect endpoints, using JSON Web Signature (**JWS**), to check the tokens' digital signature integrity of the contained claims and defending against attacks using tampered tokens.

A JWT token is a `base64` encoded string with the structure `header.payload.signature`.

A typical request to an endpoint requiring JWT validation includes a `Bearer` in the `Authorization` header:

{{< terminal >}}
GET /resource HTTP/1.1
Host: krakend.example.com
Authorization: Bearer eyJhbGciOiJIUzI1NiIXVCJ9.(truncated).ktIOfzak2ekD7IrCa9-UiO4QA
{{< /terminal >}}

Or instead, you can send the token **inside a cookie** (see [`cookie_key`](/docs/v1.3/authorization/jwt-validation/#jwt-validation-settings)).

## JWT header requirements
When KrakenD decodes the `base64` token string passed in the `Bearer` or a cookie, it expects to find in its **header** section the following **three fields**:

    {
      "alg": "RS256",
      "typ": "JWT",
      "kid": "MDNGMjU2M0U3RERFQUEwOUUzQUMwQ0NBN0Y1RUY0OEIxNTRDM0IxMw"
    }

The `alg` and `kid` values depend on your implementation, but they must be present.


{{< note title="Important!" >}}
Make sure you are declaring the right `kid` in your JWT. Paste a token in a [debugger](https://jwt.io/#debugger-io). to find out.

The value provided in the `kid` must match with the `kid` declared at the `jwk-url` or `jwk_local_path`.
{{< /note >}}

The example above used [this public key](https://albert-test.auth0.com/.well-known/jwks.json), notice how the `kid` matches both the single key present in the JWK document and the token header.

**KrakenD is built with security in mind** and uses **JWS** (instead of plain JWT or JWE), and the `kid` points to the right key in the JWS. This is why this entry is mandatory to validate your tokens.

## Basic JWT validation
The JWT validation must be present inside every endpoint definition needing it. If several endpoints are going to require JWT validation consider using the [flexible configuration](/docs/v1.3/configuration/flexible-config/) to avoid repetitive declarations.

Enable the JWT validation by adding the namespace `"github.com/devopsfaith/krakend-jose/validator"` inside the `extra_config` of the desired `endpoint`.

For instance, to protect the endpoint `/protected/resource`:

{{< highlight JSON "hl_lines=4-10" >}}
{
    "endpoint": "/protected/resource",
    "extra_config": {
        "github.com/devopsfaith/krakend-jose/validator": {
            "alg": "RS256",
            "audience": ["http://api.example.com"],
            "roles_key": "http://api.example.com/custom/roles",
            "roles": ["user", "admin"],
            "jwk-url": "https://albert-test.auth0.com/.well-known/jwks.json"
        }
    },
    "backend": [
        {
        "url_pattern": "/"
        }
    ]
}
{{< /highlight >}}

This configuration makes sure that:

- The token is well-formed and didn't expire
- The token has a valid signature
- The role of the user is either `user` or `admin` (taken from a key in the JWT payload named `http://api.example.com/custom/roles`)
- The token is not revoked in the bloom filter (see [revoking tokens](/docs/v1.3/authorization/revoking-tokens/))

## JWT validation settings
The following settings are available for JWT validation. There are a lot of options, although generally only the **fields `alg` and `jwk-url` or `jwk_local_path` are mandatory**, and the rest of the keys can be added or not at your best convenience or depending on other options.

These options are for the `extra_config`'s namespace `"github.com/devopsfaith/krakend-jose/validator"` placed in every endpoint (use [flexible configuration](/docs/v1.3/configuration/flexible-config/) to avoid code repetition):

- `alg` (*recognized string*): The hashing algorithm used by the issuer. See the [hashing algorithms](#hashing-algorithms) section for a comprehensive list of supported algorithms.
- `jwk-url` (*string*): The URL to the JWK endpoint with the public keys used to verify the token's authenticity and integrity.
- `jwk_local_path` (*string*): Local path to the JWK public keys. Instead of pointing to an external URL (`jwk-url`) the public keys are kept locally, in a plain JWK file (security alert!), or encrypted. When encrypted, also add:
    - `secret_url` (*url*): An URL with a custom scheme using one of the supported providers (e.g.: `awskms://keyID`) (see providers below)
    - `cypher_key` (*string*): The cyphering key.
- `cache` (*boolean*): Set this value to `true` to store the required keys (from the JWK descriptor) in memory for the next `cache_duration` period and avoid hammering the key server, recommended for performance. The cache can store up to 100 different public keys simultaneously.
- `cache_duration` (*int*): Change the default duration of 15 minutes. Value in **seconds**.
- `audience` (*list*): Set when you want to reject tokens that do not contain the given audience.
- `roles_key` (*string*):  When validating users through roles, provide the key name inside the JWT payload that lists their roles. If this key is nested inside another object, use the dot notation `.` to traverse each level. E.g.: `resource_access.myclient.roles` represents the payload `{resource_access: { myclient: { roles: ["myrole"] } } `.
- `roles` (*list*):  When set, the JWT token not having at least one of the listed roles are rejected.
- `roles_key_is_nested` (*bool*):  If the roles key is using a nested object using the `.` dot notation must be set to `true` in order to traverse the object.
- `scopes` (*string*): A list of scopes to validate, separated by spaces.
- `scopes_key`: The key name where the scopes can be found. The key can be a nested object using the `.` dot notation, e.g.: `data.data2.scopes`
- `scopes_matcher` (*string*): Valid options are `all` or `any`. When `all` is used, every single scope defined in the endpoint must be present in the token. Otherwise, any matching scope will let you pass.
- `issuer` (*string*): When set,  tokens not matching the issuer are rejected.
- `cookie_key` (*string*): Add the key name of the cookie containing the token when it is not passed in the headers
- `disable_jwk_security` (*boolean*): When `true`, disables security of the JWK client and allows insecure connections (plain HTTP) to download the keys. Useful for development environments.
- `jwk_fingerprints` (*strings list*): A list of fingerprints (the certificate's unique identifier) for certificate pinning and avoid man-in-the-middle attacks. Add fingerprints in base64 format.
- `cipher_suites` (*integers list*): Override the default cipher suites. Use it if you want to enforce an even higher security standard.
- `jwk_local_ca` (*string*): Path to the CA's certificate that verifies a secure connection when downloading the JWK. Use when not recognized by the system (e.g., self-signed certificates).
- `propagate-claims` (*list*): Enables passing claims in the backend's request header (see below)
- `key_identify_strategy` (*string*): Allows strategies other than `kid` to load keys. Allowed values are: `kid`, `x5t`, `kid_x5t`

For the full list of recognized algorithms and cipher suites, scroll down to the end of the document.

Here there is an example using an external `jwk-url`:

{{< highlight JSON >}}
{
"endpoint": "/foo"
"extra_config": {
    "github.com/devopsfaith/krakend-jose/validator": {
        "alg": "RS256",
        "jwk-url": "https://url/to/jwks.json",
        "cache": true,
        "audience": [
            "audience1"
        ],
        "roles_key": "department",
        "roles_key_is_nested": false,
        "roles": [
            "sales",
            "development"
        ],
        "scopes_key": "scopes",
        "scopes_matcher": "any",
        "scopes": "resource1:action1 resource2:action1 resource1:action2",
        "issuer": "http://my.api.com",
        "cookie_key": "TOKEN",
        "disable_jwk_security": true,
        "jwk_fingerprints": [
            "S3Jha2VuRCBpcyB0aGUgYmVzdCBnYXRld2F5LCBhbmQgeW91IGtub3cgaXQ=="
        ],
        "cipher_suites": [
            10, 47, 53
        ]
    }
}
}
{{< /highlight >}}

### Validation process
KrakenD does the following validation to let users hit protected endpoints:

- The `jwk-url` must be accessible by KrakenD at all times (caching is available)
- The token is [well formed](https://jwt.io/#debugger-io)
- The `kid` in the header is listed in the `jwk-url` or `jwk_local_path`.
- The content of the JWK Keys (`k`) is **base64** urlencoded
- The algorithm `alg` is supported by KrakenD and matches exactly the one used in the endpoint definition.
- The token hasn't expired
- The signature is valid.
- The given `issuer` matches (if present in the configuration)
- The given `audience` matches (if present in the configuration)
- The given claims are within the endpoint accepted `roles` (if present in the configuration))

The configuration allows you to define the set of required roles. A user who passes a token with roles `A` and `B`, can access an endpoint requiring `"roles": ["A","C"]` as it has one of the required options (`A`).

If the token is expired, the signature doesn't match, the required claims do not match, or the token is revoked, a `401 Unauthorized` is returned.

When the token doesn't include the defined ACL's required roles, a `403 Forbidden` is returned.

When you generate tokens for end-users, make sure to set a **low expiration**. Tokens are supposed to have short lives and are recommended to expire in a few minutes or hours.

### Accepted providers for encrypting payloads
When using a `jwk_local_path` the `secret_url` scheme accepts different providers:

#### Local secrets
The local secrets require an URL with the following scheme:
```
base64key://base64content
```
The URL host must be base64 encoded and must decode to exactly 32 bytes. Here is an example of the `extra_config`:

```
{
    "jwk_local_path":"./jwk.txt",
    "secret_url":"base64key://smGbjm71Nxd1Ig5FS0wj9SlbzAIrnolCz9bQQ6uAhl4=",
    "cypher_key":"gCERmfqHMoEu3+utqBa/R1oMZYIvh0OOKtJmnX/hDPDxbXCGXGvO3SF7B5FWxrJnRW7rnjGIV4eP2VLrYX2q9pJM49BpP+A9"
}
```
This config will use the key `smGbjm71Nxd1Ig5FS0wj9SlbzAIrnolCz9bQQ6uAhl4=` for decrypting de `cypher_key` and then decrypting the content of the file `./jwt.txt`.

See this test to [understand how to generate and encrypt payloads](https://github.com/krakend/krakend-jose/blob/master/jwk_test.go).

#### Amazon KMS
```
awskms://keyID
```
The URL Host + Path are used as the key ID, which can be in the form of an Amazon Resource Name (ARN), alias name, or alias ARN. See https://docs.aws.amazon.com/kms/latest/developerguide/viewing-keys.html#find-cmk-id-arn for more details. Note that ARNs may contain ":" characters, which cannot be escaped in the Host part of a URL, so the `awskms:///<ARN>` form should be used.

[More information about AWS KMS](https://docs.aws.amazon.com/kms/latest/developerguide/viewing-keys.html#find-cmk-id-arn)


#### Azure's Key Vault
```
azurekeyvault://keyID
```
The credentials are taken from the environment unless the `AZURE_KEYVAULT_AUTH_VIA_CLI` environment variable is set to true, in which case it uses the `az` command line.


[More information about Azure Key Vault](https://docs.microsoft.com/en-us/azure/key-vault/general/basic-concepts)
#### Google Cloud KMS
```
gcpkms://projects/[PROJECT_ID]/locations/[LOCATION]/keyRings/[KEY_RING]/cryptoKeys/[KEY]
```
You can take the URL [from the GCP console](https://cloud.google.com/kms/docs/object-hierarchy#key).

#### Hashicorp's Vault
```
hashivault://keyID
```

Environment variables `VAULT_SERVER_URL` and `VAULT_SERVER_TOKEN` are used.



## Passing claims to the backend URL

Since KrakenD 1.2.0, it is possible to use data present in the claims to inject it into the backend's final URL. The notation of the `url_pattern` field includes the parsing of `{JWT.some_claim}`, where `some_claim` is an attribute of your claim.

For instance, when your JWT payload is represented by something like this:

    {
        "sub": "1234567890",
        "name": "Mr. KrakenD"
    }

Having a `backend` defined like this:

    {
        "url_pattern": "/foo/{JWT.sub}",
        "method": "POST"
        ...
    }

The call to your backend would produce the request:

    POST /foo/1234567890

Keep in mind that this syntax in the `url_pattern` field is only available if the backend loads the extra_config `"github.com/devopsfaith/krakend-jose/validator"` and that **it does not work with nested attributes** in the payload.

If KrakenD can't replace the claim's content for any reason, the backend receives a request to the literal URL `/foo/{JWT.sub}`.

## Propagate JWT claims as request headers
Since KrakenD 1.3.0, it is possible to forward claims in a JWT as request headers. It is a common use case to have, for instance, the sub claim added as an `X-User` header to the request.

**Important:** The endpoint `headers_to_pass` needs to be set as well, so the backend can see it.

```
"extra_config": {
        "github.com/devopsfaith/krakend-jose/validator": {
          "propagate-claims": [
            ["sub", "x-user"]
          ],
          ...
        }
      },
```
In this case, the `sub` claim's value will be added as `x-user` header to the request. If the claim does not exist, the mapping is just skipped.

## A complete running example

The [KrakenD Playground](/docs/v1.3/overview/playground/) demonstrates how to protect endpoints using JWT and includes two examples ready to use:

- Integration with an external third party using a [Single Page Application from Auth0](https://auth0.com/docs/applications/spa/)
- Integration with an internal identity provider service (mocked) using a symmetric key algorithm and a signer middleware.

To try it, [clone the playground](https://github.com/krakend/playground-community) and follow the README.

## Supported hashing algorithms and cipher suites

### Hashing algorithms

Accepted values for the `alg` field are:

- `EdDSA`: EdDSA
- `HS256`: HS256 - HMAC using SHA-256
- `HS384`: HS384 - HMAC using SHA-384
- `HS512`: HS512 - HMAC using SHA-512
- `RS256`: RS256 - RSASSA-PKCS-v1.5 using SHA-256
- `RS384`: RS384 - RSASSA-PKCS-v1.5 using SHA-384
- `RS512`: RS512 - RSASSA-PKCS-v1.5 using SHA-512
- `ES256`: ES256 - ECDSA using P-256 and SHA-256
- `ES384`: ES384 - ECDSA using P-384 and SHA-384
- `ES512`: ES512 - ECDSA using P-521 and SHA-512
- `PS256`: PS256 - RSASSA-PSS using SHA256 and MGF1-SHA256
- `PS384`: PS384 - RSASSA-PSS using SHA384 and MGF1-SHA384
- `PS512`: PS512 - RSASSA-PSS using SHA512 and MGF1-SHA512

### Cipher suites

Accepted values for cipher suites are:

- `5`: TLS_RSA_WITH_RC4_128_SHA
- `10`: TLS_RSA_WITH_3DES_EDE_CBC_SHA
- `47`: TLS_RSA_WITH_AES_128_CBC_SHA
- `53`: TLS_RSA_WITH_AES_256_CBC_SHA
- `60`: TLS_RSA_WITH_AES_128_CBC_SHA256
- `156`: TLS_RSA_WITH_AES_128_GCM_SHA256
- `157`: TLS_RSA_WITH_AES_256_GCM_SHA384
- `49159`: TLS_ECDHE_ECDSA_WITH_RC4_128_SHA
- `49161`: TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA
- `49162`: TLS_ECDHE_ECDSA_WITH_AES_256_CBC_SHA
- `49169`: TLS_ECDHE_RSA_WITH_RC4_128_SHA
- `49170`: TLS_ECDHE_RSA_WITH_3DES_EDE_CBC_SHA
- `49171`: TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA
- `49172`: TLS_ECDHE_RSA_WITH_AES_256_CBC_SHA
- `49187`: TLS_ECDHE_ECDSA_WITH_AES_128_CBC_SHA256
- `49191`: TLS_ECDHE_RSA_WITH_AES_128_CBC_SHA256

**Default suites** are:

- `49199`: TLS_ECDHE_RSA_WITH_AES_128_GCM_SHA256
- `49195`: TLS_ECDHE_ECDSA_WITH_AES_128_GCM_SHA256
- `49200`: TLS_ECDHE_RSA_WITH_AES_256_GCM_SHA384
- `49196`: TLS_ECDHE_ECDSA_WITH_AES_256_GCM_SHA384
- `52392`: TLS_ECDHE_RSA_WITH_CHACHA20_POLY1305
- `52393`: TLS_ECDHE_ECDSA_WITH_CHACHA20_POLY1305
