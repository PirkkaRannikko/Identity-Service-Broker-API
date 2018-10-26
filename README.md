![Checkout Finland](checkout-logo-vaaka.svg)

# Checkout Finland Service Provider API for Checkout Identity Service Broker

2018-09-10

Checkout Identification Service Broker allows Service Providers to implement strong electronic identification (Finnish bank credentials, Mobile ID) easily to websites and mobile apps via single API.

To identify the user the Service Provider (your website) redirects the user to the Identification Service Broker (Checkout) with an authorization request. The user chooses the Identity Provider (a bank or Mobile ID) and is redirected there where he/she authenticates with his/her own credentials. Checkout will process the authentication result, and return the user to your website with verified information about the identity of the user.

## Definitions

- **JWT** or JSON Web Token is a standard for wrapping attributes into a token. JWS is a signed and JWE an encrypted JWT token.
- **OIDC** or OpenID Connect is a standard easy to use protocol for identifying and authenticating users.
- **Service Provider** is the service asking for the user identity.
- **Identity Service Broker** is the Checkout service that lets the user choose an identity provider and that passes the requested user identity information to the service provider.
- **Identity Provider** is a provider of identification, i.e. a Bank or mobile ID.
- **Identification wall** is a list of Identity Providers shown on the UI. There are two options for this, either to use the built-in Identification Wall on the ISB or to display it on the Service Provider using the API provided by the ISB.

## Prerequisites

To identify users using the Identity Service Broker and the OIDC API for Service Providers, you need the following pieces of configuration:

* Client identifier

  Your service is identified by a unique client identifier string, which Checkout will generate for you. Basically this is the Merchant ID you get when you place on order to Checkout in https://checkout.fi/verkkokauppiaalle/tilaus/ .

* Client secret

  A shared secret (a password really) to authenticate your service to the Checkout API. Checkout will generate the client secret and provide it to you. In the future you can get from Checkout’s Extranet.

* Checkout OIDC authorization endpoint

  The Checkout OIDC authorization endpoint for production use is `https://isb.isb.checkout.fi/authorize`. For testing please use the sandbox endpoint `https://isb.sandbox-isb.checkout-developer.fi/authorize`.

* Checkout OIDC token endpoint

  The Checkout OIDC token endpoint for production use is `https://isb.isb.checkout.fi/token`. For testing please use the sandbox endpoint `https://isb.sandbox-isb.checkout-developer.fi/token`.

* Checkout OIDC profile endpoint

  The Checkout OIDC profile endpoint for production use is `https://isb.isb.checkout.fi/profile`. For testing please use the sandbox endpoint `https://isb.sandbox-isb.checkout-developer.fi/profile`. This endpoint provides exactly the same information as the token endpoint and as such is redundant.

* RSA keypair to decrypt identity token

  Checkout will encrypt the identity token identifying the user with your public key and you will have to decrypt it with your private key.

  To generate a 2048 bit RSA key run the command `openssl genrsa -out private.pem 2048` (you could replace the filename private.pem with one of your own choosing). Keep this key private. To export the public portion, run the command `openssl rsa -in private.pem -pubout`. Register the public key with Checkout, but make sure the key you share begins with `-----BEGIN PUBLIC KEY-----`. Checkout will provide an initial keypair for you. In the future you can set them from Checkout's Extranet.

* Checkout's RSA certificate

  Identity tokens are signed by Checkout to protect their content. You must verify the signature against Checkout's certificate. Note that the certificate is rolled over at times. You should support having two valid certificates (old and new) to handle the transition to a new certificate without downtime. Checkout will provide you with the current RSA certificate. In the future you can get them from Checkout's Extranet.


## Security concerns

- Client secret and private RSA key must be protected and not revealed to users.
- Client secret and keys should be rotated every now and then.
- Client secret or keys must not be sent to the user's browser. I.e. processing the identification should be done server side, not in browser side Javascript.

## Flow with Identification Wall on the Identity Service Broker

Checkout identification service uses the OpenID Connect Authorization Code flow. I.e. the following steps are taken to identify a user:

1. Service Provider directs the user to Checkout's service endpoint with parameters documented below. `ftn_idp_id` shall not exist among request parameters
2. Checkout lets the user identify themselves using a provider of their choosing.
3. Once identified, the user is passed back to the Service Provider's `redirect_uri` with an access code.
4. The Service Provider makes a direct API call to the Checkout API and gets an encrypted and signed identity token in exchange for the access code.

![Flow graph](./flow.png?raw=true)

## Flow with Identification Wall on the Service Provider

Checkout identification service uses the OpenID Connect Authorization Code flow. I.e. the following steps are taken to identify a user:

1. Service Provider uses the /api/embedded-ui/{client_id} API to get the data to display the Identification Wall on the UI
2. Service Provider lets the user choose identity provider.
3. Service Provider directs the user to Checkout's service endpoint with parameters documented below. `ftn_idp_id` shall be delivered as request parameter
4. Once identified, the user is passed back to the Service Provider's `redirect_uri` with an access code.
5. The Service Provider makes a direct API call to the Checkout API and gets an encrypted and signed identity token in exchange for the access code.

![Flow graph](./embedded-ui-flow.png?raw=true)

## GET /api/embedded-ui/{client_id}

To display the Identification Wall on the Service Provider the Service Provider shall use the /api/embedded-ui/{client_id} API of the Identity Service Broker to get the data needed to display the Identification Wall. Client_id is the client identifier that specifies which service provider is asking for identification. Service Provider does not need to use this API if it uses the flow with Identification Wall on the Identity Service Broker.

The query string of the request can include the following optional parameter:
- **lang** indicates the language for the returned data (`fi`, `sv` or `en`). If parameter is omitted the default language is `fi`.

Example API calls:

`GET https://isb.sandbox-isb.checkout-developer.fi/api/embedded-ui/example_service_provider`

`GET https://isb.sandbox-isb.checkout-developer.fi/api/embedded-ui/example_service_provider?lang=en`

The API returns json data.

Example of returned data:
```json
{
  "identityProviders": [
    {
        "name": "Osuuspankki",
        "imageUrl": "https://isb.sandbox-isb.checkout-developer.fi/public/images/idp/op_140x75.png",
        "ftn_idp_id": "fi-op-tupas"
    },
    {
        "name": "Nordea",
        "imageUrl": "https://isb.sandbox-isb.checkout-developer.fi/public/images/idp/nordea_140x75.png",
        "ftn_idp_id": "fi-nordea-tupas"
    }
  ],
  "isbProviderInfo": "Identification is provided by Checkout Finland Oy",
  "isbIconUrl": "https://isb.sandbox-isb.checkout-developer.fi/public/images/checkout.png",
  "isbConsent": "By continuing, I accept that the service provider will receive my name and personal identity code"
}
```

Service Provider needs to use and display these two fields `isbProviderInfo` and `isbConsent` on the UI.

## GET /oauth/authorize/

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
- **ftn_idp_id** shall be delivered if the Identification Wall is on the Service Provider. It contains the id of the user chosen idp.

Example identification requests:

`GET https://isb.sandbox-isb.checkout-developer.fi/oauth/authorize?client_id=example_service_provider&response_type=code&redirect_uri=https%3A%2F%2Fexample-service-provider.example%2Fbell&state=GIlBncQk4vsbThjMNBJ49G&scope=openid%20profile`

`GET https://isb.sandbox-isb.checkout-developer.fi/oauth/authorize?prompt=consent&client_id=example_service_provider&response_type=code&redirect_uri=https%3A%2F%2Fexample-service-provider.example%2Fbell&state=GIlBncQk4vsbThjMNBJ49G&scope=openid%20profile%20strong`

`GET https://isb.sandbox-isb.checkout-developer.fi/oauth/authorize?ftn_idp_id=op&client_id=example_service_provider&response_type=code&redirect_uri=https%3A%2F%2Fdsp.sandbox-isb.checkout-developer.fi%2Fbell&state=nRL4A7Xul4GSF_Z5mkEN8_&scope=openid%20profile%20personal_identity_code`


Once the identification process is done or if there is a recoverable error, the user is directed back to the service provider to the URI specified in the request. The following parameters are included in the query string:
- **state** is passed as is from the request.
- **code** is the authorization code for use in the next phase (only included after succesful identification).
- **error** is the reason why identification failed (in case of error only).

Example return:

`GET https://example-service-provider.example/bell?code=eyJhb[...]4bGg&state=GIlBncQk4vsbThjMNBJ49G` using Hapi / Bell library on javascript

`GET http://example-service-provider/?code=eyJh[....]_0w&state=77deb5b7f773ef6dafc12d9cf0588f57` using league/oauth2-client library on PHP

## POST /oauth/token

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

`POST https://isb.sandbox-isb.checkout-developer.fi/oauth/token`

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

Parameter explanations:
- **access_token** Access Token for the /oauth/profile API (OIDC UserInfo Endpoint)
- **token_type** OAuth 2.0 Token Type value. The value is allways `Bearer`
- **expires_in** Expiration time of the Access Token in seconds since the response was generated
- **id_token** Identity Token

## Identity token

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

## GET /oauth/profile

This API is the OIDC UserInfo Endpoint. There is no need to call this if Service Provider has already got all the needed user information in processing the reply from the /oauth/token.
Service Provider can send request to the OIDC UserInfo Endpoint to obtain Claims about the End-User using an Access Token obtained from the /oauth/token API reply. The UserInfo Endpoint is an OAuth 2.0 [RFC6749] Protected Resource that complies with the OAuth 2.0 Bearer Token Usage [RFC6750] specification. The Access Token shall be sent using the Authorization header field as described below:

```
Authorization: Bearer eyJh[...]2A
```

where `eyJh[...]2A` is the access_token.

Example identification request:

`GET https://isb.sandbox-isb.checkout-developer.fi/oauth/profile`

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

## Sandbox

The sandbox differs from the production in three major ways.

- The sandbox environment provides test data instead of real personal information.
- To use the sandbox environment you need to use the separate API endpoints described above.
- Common shared credentials and client id are used for the sandbox environment.

These id's and keys are used for the sandbox environment:

- **Client identifier**: saippuakauppias
- **Client secret**: correct-horse-battery-staple
- **Token decryption key**: See `sandbox-sp-key.pem`
- **Signature verification key**: See `sandbox-isb-public-key.pem`

## Service Provider code example

Currently there is PHP-based service provider demo application available. See https://github.com/CheckoutFinland/Identity-Service-Broker-integration-example .

## Libraries for Service Provider

See the examples directory for examples on how to implement a service provider based on various libraries and languages.

### Javascript

Bell is a simple library to take care of the OpenID Connect flow. See https://github.com/hapijs/bell .

Node-jose can be used to decrypt and verify the identity token. See https://github.com/cisco/node-jose .

### PHP

oauth2-client makes it simple to integrate your Service Provider application with Checkout ISB OpenID Connect flow. See https://github.com/thephpleague/oauth2-client .

Jose-php can be used to decrypt and verify the identity token. See https://github.com/nov/jose-php .

## Extra material

To learn more about OpenID Connect, see the specification: https://openid.net/specs/openid-connect-core-1_0.html

## Support
If you have any questions please contact our support team:

Email: [asiakaspalvelu@checkout.fi](mailto:asiakaspalvelu@checkout.fi)
Phone: [+358800 552 010](tel:+358800552010)

Support is available from Monday to Friday between 06–23 (except on public holidays).

## Pricing

Pricing will be announced later in https://checkout.fi
