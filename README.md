# Service Provider API for OP Identity Service Broker

2019-03-28 (DRAFT. Expires in 2019-04-12)

NOTE! The API endpoints are not yet live. 

OP Identification Service Broker allows Service Providers to implement strong electronic identification (Finnish bank credentials, Mobile ID) easily to websites and mobile apps via single API.

To identify the user the Service Provider (your website) redirects the user to the Identification Service Broker (OP) with an authorization request. The user chooses the Identity Provider (a bank or Mobile ID) and is redirected there where he/she authenticates with his/her own credentials. OP will process the authentication result, and return the user to your website with verified information about the identity of the user.

Table of contents:
1. Definitions
2. Prerequisites
3. Security concerns
4. Flow with hosted Identity Service Broker UI
5. Flow with embedded Identity Service Broker UI
6. GET /api/embedded-ui/{client_id}
7. GET /oauth/authorize/
8. POST /oauth/token
9. Identity token
10. GET /oauth/profile
11. GET /.well-known/openid-configuration
12. JWKS
13. Public Sandbox for customer testing
14. Service Provider code example
15. Libraries for Service Provider
16. Javascript
17. PHP
18. Extra material
19. Support
20. Pricing

## 1. Definitions

- **Service Provider (SP)** is the service asking for the user identity.
- **Identity Service Broker (ISB)** is the Checkout service that lets the user choose an identity provider and that passes the requested user identity information to the service provider.
- **Identity Provider (IdP)** is a provider of identification, i.e. a Bank or mobile ID.
- **Identity Service Broker UI** is a list of Identity Providers shown on the UI. There are two options for displaying the UI. Service Provider can redirect the user to the hosted UI in the Identity Service Broker or embed the UI into its own UI.
- **OIDC** or OpenID Connect is a standard easy to use protocol for identifying and authenticating users.
- **JWT** or JSON Web Token is a standard for wrapping attributes into a token. JWS is a signed and JWE an encrypted JWT token.
- **JWKS** JSON Web Key Set is a standard way to exchange public keys between SP and ISB.

## 2. Prerequisites

To identify users using the Identity Service Broker and the OIDC API for Service Providers, you need the following pieces of configuration:

* Client identifier

  Your service is identified by a unique client identifier string, which OP will generate for you during the onboarding process.

* OP OIDC authorization endpoint

  The OP OIDC authorization endpoint for production use is `https://isb.op.fi/authorize`. For testing please use the sandbox endpoint `https://isb-test.op.fi/authorize`.

* OP OIDC token endpoint

  The OP OIDC token endpoint for production use is `https://isb.op.fi/token`. For testing please use the sandbox endpoint `https://isb-test.op.fi/token`.

* OP OIDC profile endpoint

  The OP OIDC profile endpoint for production use is `https://isb.op.fi/profile`. For testing please use the sandbox endpoint `https://isb-test.op.fi/profile`. This endpoint provides exactly the same information as the token endpoint and as such is redundant.
  
* RSA keypair to sign requests
 
   Signing is used for verifying that requests originate from the SP. Signing is used in requests to two endpoints: /oauth/authorize and /oauth/token.
 
To generate a 2048 bit RSA key run the command `openssl genrsa -out private.pem 2048` (you could replace the filename private.pem with one of your own choosing).

* RSA keypair to decrypt identity token

  OP will encrypt the identity token identifying the user with your public key and you will have to decrypt it with your private key. Keys are generated the same way as signing keys. Both encryption and signing public keys must be published in the SP's JKWS endpoint. Keep the private portions of these keypairs private.

* OP JWKS endpoint

  Identity tokens are signed by OP to protect their content. You must verify the signature against OP's public key which can be fetched from the OP JWKS endpoint `https://isb.op.fi/jwks/broker`. For testing please use the sandbox endpoint `https://isb-test.op.fi/jwks/broker`. Note that the keys are rolled over at times. The endpoint may contain several valid keys. You may safely cache keys for at most one day. When fetching keys from endpoint you must verify the TLS certificate to ensure that the keys are genuine.

## 3. Security concerns

- Private RSA keys must be protected and not revealed to users.
- The keys should be rotated every now and then. When depracating a key you should remove it from your JWKS endpoint at least one day before deactivating it to prevent disruptions to service. We may cache your public keys for up to one day on the ISB.
- Keys must not be sent to the user's browser. I.e. processing the identification should be done server side, not in browser side Javascript.

## 4. Flow with hosted Identity Service Broker UI

Checkout identification service uses the OpenID Connect Authorization Code flow. I.e. the following steps are taken to identify a user:

1. Service Provider directs the user to Checkout's service endpoint with parameters documented below. `ftn_idp_id` shall not exist among request parameters
2. Checkout lets the user identify themselves using a provider of their choosing.
3. Once identified, the user is passed back to the Service Provider's `redirect_uri` with an access code.
4. The Service Provider makes a direct API call to the Checkout API and gets an encrypted and signed identity token in exchange for the access code.

![Flow graph](./flow.png?raw=true)

## 5. Flow with embedded Identity Service Broker UI

Checkout identification service uses the OpenID Connect Authorization Code flow. I.e. the following steps are taken to identify a user:

1. Service Provider uses the /api/embedded-ui/{client_id} API to get the data to display the embedded Identity Service Broker UI on the SP UI.
2. Service Provider lets the user choose identity provider.
3. Service Provider directs the user to Checkout's service endpoint with parameters documented below. `ftn_idp_id` shall be delivered as request parameter
4. Once identified, the user is passed back to the Service Provider's `redirect_uri` with an access code.
5. The Service Provider makes a direct API call to the Checkout API and gets an encrypted and signed identity token in exchange for the access code.

![Flow graph](./embedded-ui-flow.png?raw=true)

## 6. GET /api/embedded-ui/{client_id}

To display the embedded Identity Service Broker UI the Service Provider shall use the /api/embedded-ui/{client_id} API of the Identity Service Broker to get the needed data. Client_id is the client identifier that specifies which service provider is asking for identification. Service Provider does not need to use this API if it uses the flow with hosted Identity Service Broker UI.

The query string of the request can include the following optional parameter:
- **lang** indicates the language for the returned data (`fi`, `sv` or `en`). If parameter is omitted the default language is `fi`.

Example API calls:

`GET https://isb.isb-sandbox.checkout-developer.fi/api/embedded-ui/example_service_provider`

`GET https://isb.isb-sandbox.checkout-developer.fi/api/embedded-ui/example_service_provider?lang=en`

The API returns json data.

Example of returned data:
```json
{
  "identityProviders": [
    {
        "name": "Osuuspankki",
        "imageUrl": "https://isb.isb-sandbox.checkout-developer.fi/public/images/idp/op_140x75.png",
        "ftn_idp_id": "fi-op-tupas"
    },
    {
        "name": "Nordea",
        "imageUrl": "https://isb.isb-sandbox.checkout-developer.fi/public/images/idp/nordea_140x75.png",
        "ftn_idp_id": "fi-nordea-tupas"
    }
  ],
  "isbProviderInfo": "Identification is provided by Checkout Finland Oy",
  "isbIconUrl": "https://isb.isb-sandbox.checkout-developer.fi/public/images/checkout.png",
  "isbConsent": "By continuing, I accept that the service provider will receive my name and personal identity code"
}
```

Service Provider needs to use and display these two fields `isbProviderInfo` and `isbConsent` on the UI.

API errors:

| Error | Description | Action |
| --- | --- | --- |
| 404 Not found / Service provider not found | the given client_id is not valid | error is shown on ISB |

## 7. GET /oauth/authorize/

To initiate the identification process the service provider directs the user to Checkout's OIDC endpoint either by redirect or by direct link. The query string of the request must include the following parameters:

- **client_id** is the client identifier that specifies which service provider is asking for identification.
- **redirect_uri** specifies to which URI on your site (the service provider) you want the user to return to once identification is done. This URI must be registered with Checkout (except when using the sandbox environment) to prevent other services misusing your credentials.
- **response_type** value must be `code`.
- **scope** is a comma separated list of scopes, or  basically sets of information requested. This must include `openid` and `personal_identification_code` . For example `openid profile personal_identity_code`. The `profile` includes `name`, `given_name`, `family_name` and `birthdate`. If the Service Provider's purpose for identifying the user is to create new identification methods, i.e. for example to create an user account with username and password, then the Service Provider must report such purpose by adding either `weak` (for weak identifiers, for example password account) or `strong` (for strong electronic identification) to the scopes. Using weak or strong as a purpose may affect pricing so please do check your contract and/or ask Checkout for advice.

The following optional parameters may be used:
- **ui_locales** selects user interface language (`fi`, `sv` or `en`).
- **nonce** value is passed on to identity token as is.
- **prompt** can be set to `consent` to indicate that the user should be asked to consent to personal data being transferred. In this case the Identity Service Broker will display a verification screen after the user has been authenticated.
- **state** is an opaque value you can use to maintain state between request and callback. Use of `state` is recommended.
- **ftn_idp_id** shall be delivered if the SP has the embedded Identity Service Broker UI. Parameter contains the id of the user chosen idp.

Example identification requests:

`GET https://isb.isb-sandbox.checkout-developer.fi/oauth/authorize?client_id=example_service_provider&response_type=code&redirect_uri=https%3A%2F%2Fexample-service-provider.example%2Fbell&state=GIlBncQk4vsbThjMNBJ49G&scope=openid%20profile`

`GET https://isb.isb-sandbox.checkout-developer.fi/oauth/authorize?prompt=consent&client_id=example_service_provider&response_type=code&redirect_uri=https%3A%2F%2Fexample-service-provider.example%2Fbell&state=GIlBncQk4vsbThjMNBJ49G&scope=openid%20profile%20strong`

`GET https://isb.isb-sandbox.checkout-developer.fi/oauth/authorize?ftn_idp_id=op&client_id=example_service_provider&response_type=code&redirect_uri=https%3A%2F%2Fdsp.isb-sandbox.checkout-developer.fi%2Fbell&state=nRL4A7Xul4GSF_Z5mkEN8_&scope=openid%20profile%20personal_identity_code`


Once the identification process is done or if there is a recoverable error, the user is directed back to the service provider to the URI specified in the request. The following parameters are included in the query string:
- **state** is passed as is from the request.
- **code** is the authorization code for use in the next phase (only included after succesful identification).
- **error** is the reason why identification failed (in case of error only).

Example return:

`GET https://example-service-provider.example/bell?code=eyJhb[...]4bGg&state=GIlBncQk4vsbThjMNBJ49G` using Hapi / Bell library on javascript

`GET http://example-service-provider/?code=eyJh[....]_0w&state=77deb5b7f773ef6dafc12d9cf0588f57` using league/oauth2-client library on PHP

API errors:

| Error | Description | Action |
| --- | --- | --- |
| 400 Bad Request / invalid_request | if no valid SP can determined from parameters, redirect_uri is invalid or validation for state-parameter fails | error is shown on ISB |
| access_denied | authorization has been canceled | redirected to the SP with error |
| invalid_request | request parameter validation fails | redirected to the SP with error |
| invalid_scope | openid or personal_identity_code scope is missing. Validation fails | redirected to the SP with error |
| login_required | prompt-parameter can't have "login" value | redirected to the SP with error |
| invalid_ftn_idp_id | invalid ftn_idp_id given when SP has the embedded Identity Service Broker UI | redirected to the SP with error |

## 8. POST /oauth/token

The actual user identity token from the token endpoint can be fetched using the /oauth/token API. The following parameters shall be included as request payload parameters:
- **code** authorization code, which was returned in succesful /oauth/authorize/ reply. Mandatory.
- **redirect_uri** specifies to which URI on your site (the service provider) you want the user to return to once identification is done. This URI must be registered with Checkout (except when using the sandbox environment) to prevent other services misusing your credentials. Mandatory.
- **grant_type** needs to have value `authorization_code`. Mandatory.
- **client_id** is the client identifier that specifies which service provider is asking for identification. Not needed if `authorization` header is used.
- **client_secret** A shared secret (a password really) to authenticate your service to the Checkout API. Not needed if `authorization` header is used.

The request can be authenticated by HTTP Basic Auth with client id and client secret. Use the client id as the username and client secret as the password in the standard `Authorization` header. The header value is the word `Basic` followed by a space and a base64 encoded string of username and password joined by a colon (`:`) as described below:

```
Authorization: Basic czZCaGRSa3F0Mzo3RmpmcDBaQnIxS3REUmJuZlZkbUl3
```

where `czZCaGRSa3F0Mzo3RmpmcDBaQnIxS3REUmJuZlZkbUl3` is an example value


Using `authorization` header is not mandatory if the `clientId`and the `clientSecret` parameters are included in the request payload.

The authorization code can only be used once.

Example identification request:

`POST https://isb.isb-sandbox.checkout-developer.fi/oauth/token`

The API returns json data.

Example of returned data:
```json
{
  "access_token": "eyJh[...]2A",
  "token_type": "Bearer",
  "expires_in": 3600,
  "id_token": "eyJl[...]Nw"
}
```

API errors:

| Error | Description | Action |
| --- | --- | --- |
| 401 Unauthorized / invalid_client | the given client_id or client_secret is not valid | error is shown on ISB |
| 400 Bad Request / invalid_grant | the token has already been exchanged or token validation failed | error is shown on ISB |

Parameter explanations:
- **access_token** Access Token for the /oauth/profile API (OIDC UserInfo Endpoint)
- **token_type** OAuth 2.0 Token Type value. The value is allways `Bearer`
- **expires_in** Expiration time of the Access Token in seconds since the response was generated
- **id_token** Identity Token

## 9. Identity token

The identity token is a JWT token that contains identity attributes about the user, for example name, date of birth or personal identity code. The token is signed by Checkout's RSA key. The signed token is embedded and encrypted into an JWE token using the service provider's public key.

To obtain the user attributes from the identity token you need to first decrypt the JWE token (`id_token`) received from the '/oauth/token' API. Decryption is done using the Service Provider private RSA key. The decrypted JWS token is signed using Checkout's RSA certificate to prevent tampering. Service Provider needs to verify that the signature is valid using the JWT library of your choice. The payload of the JWS token embedded in the JWE token contains user information.

The information received depends on the scope of identification request and on what attributes are available. Do note that not all sources of information have given name and family name available as separate attributes. The following attributes may be available currently:

- **birthdate**: Birth date
- **given_name**: Given name
- **family_name**: Family name
- **name**: Family name and given name
- **personal_identity_code**: The Finnish personal identity code

In addition there are these standard attributes:

- **auth_time**: Time of authentication in seconds since UNIX epoch
- **sub**: Subject identifier, not persistent, feel free to ignore

Example:

```json
{
  "sub": "a589adb6-1550-4b17-90c4-a19e8c3f3c0e",
  "name": "von Möttonen Matti Matias",
  "given_name": "Matti Matias",
  "family_name": "von Möttonen",
  "birthdate": "1900-01-01",
  "auth_time": 1519629890
}
```

## 10. GET /oauth/profile

This API is the OIDC UserInfo Endpoint. There is no need to call this if Service Provider has already got all the needed user information in processing the reply from the /oauth/token.
Service Provider can send request to the OIDC UserInfo Endpoint to obtain Claims about the End-User using an Access Token obtained from the /oauth/token API reply. The UserInfo Endpoint is an OAuth 2.0 [RFC6749] Protected Resource that complies with the OAuth 2.0 Bearer Token Usage [RFC6750] specification. The Access Token shall be sent using the Authorization header field as described below:

```
Authorization: Bearer eyJh[...]2A
```

where `eyJh[...]2A` is the access_token.

Example identification request:

`GET https://isb.isb-sandbox.checkout-developer.fi/oauth/profile`

API errors:

| Error | Description | Action |
| --- | --- | --- |
| 401 Unauthorized / invalid_token | Invalid Access Token | error is shown on ISB |
| 401 Unauthorized / invalid_client | the given client_id is not valid | error is shown on ISB |

The API returns json data. The information received depends on the scope of identification request and on what attributes are available. Do note that not all sources of information have given name and family name available as separate attributes. The following attributes may be available currently:

- **birthdate**: Birth date
- **given_name**: Given name
- **family_name**: Family name
- **name**: Family name and given name
- **personal_identity_code**: The Finnish personal identity code

In addition there is this standard attribute:

- **sub**: Subject identifier, not persistent, feel free to ignore

Example of returned data:
```json
{
  "sub": "1",
  "name": "von Möttonen Matti Matias",
  "personal_identity_code": "010101-011"
}
```

## 11. GET /.well-known/openid-configuration

## 12. JWKS

## 13. Public Sandbox for customer testing

The sandbox differs from the production in three major ways.

- The sandbox environment provides test data instead of real personal information.
- To use the sandbox environment you need to use the separate API endpoints described above.
- Common shared credentials and client id are used for the sandbox environment.

These id's and keys are used for the sandbox environment:

- **Client identifier**: saippuakauppias
- **Client secret**: correct-horse-battery-staple
- **Token decryption key**: See `sandbox-sp-key.pem`
- **Signature verification key**: See `sandbox-isb-public-key.pem`

## 14. Service Provider code example

Currently there is PHP-based service provider demo application available. See https://github.com/CheckoutFinland/Identity-Service-Broker-integration-example .

## 15. Libraries for Service Provider

See the examples directory for examples on how to implement a service provider based on various libraries and languages.

## 16. Javascript

Bell is a simple library to take care of the OpenID Connect flow. See https://github.com/hapijs/bell .

Node-jose can be used to decrypt and verify the identity token. See https://github.com/cisco/node-jose .

## 17. PHP

oauth2-client makes it simple to integrate your Service Provider application with Checkout ISB OpenID Connect flow. See https://github.com/thephpleague/oauth2-client .

Jose-php can be used to decrypt and verify the identity token. See https://github.com/nov/jose-php .

## 18. Extra material

To learn more about OpenID Connect, see the specification: https://openid.net/specs/openid-connect-core-1_0.html

## 19. Support
If you have any questions please contact our support team:

Email: [asiakaspalvelu@checkout.fi](mailto:asiakaspalvelu@checkout.fi)
Phone: [+358800 552 010](tel:+358800552010)

Support is available from Monday to Friday between 06–23 (except on public holidays).

## 20. Pricing

Pricing will be announced later in https://checkout.fi
