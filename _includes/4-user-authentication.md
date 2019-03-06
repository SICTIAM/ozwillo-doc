## User Authentication
{: #s4-user-authentication} 

### Introduction
{: #s4-introduction}

The platform authentication and authorization service implements the following international standards:

- <a href="https://openid.net/specs/openid-connect-core-1_0.html" target="_blank">OpenID Connect Core 1.0</a> (using the Authorization Code flow only)
- <a href="https://openid.net/specs/openid-connect-discovery-1_0.html#ProviderConfig" target="_blank">OpenID Connect Discovery 1.0</a> (provider configuration only)
- <a href="https://openid.net/specs/openid-connect-session-1_0.html" target="_blank">OpenID Connect Session Management 1.0</a>
- <a href="https://tools.ietf.org/html/rfc7662" target="_blank">OAuth 2.0 Token Introspection (RFC 7662)</a>
- <a href="https://tools.ietf.org/html/rfc7009" target="_blank">OAuth Token Revocation (RFC 7009)</a>
- <a href="https://tools.ietf.org/html/rfc7523" target="_blank">JSON Web Token (JWT) Profile for OAuth 2.0 Client Authentication and Authorization Grants (RFC 7523)</a> (using a JWT as an authorization grant only, used during provisioning)
- <a href="https://tools.ietf.org/html/rfc7636" target="_blank">Proof Key for Code Exchange (PKCE) by OAuth Public Clients (RFC 7636)</a> (S256 method only)

In this matter, the platform acts as an OpenID Provider, which is defined as (see OpenID <a href="https://openid.net/specs/openid-connect-core-1_0.html#Terminology" target="_blank">terminology</a>):

> **OAuth 2.0 Authorization Server** that is capable of Authenticating the End-User and providing Claims to a Relying Party about the Authentication event and the End-User.

In this respect, the provider acts as a Relying Party:

> **OAuth 2.0 Client** application requiring End-User Authentication and Claims from an OpenID Provider.

You may refer to the previous [Authentication in brief](#s1-authorization) and [scopes](#s1-scopes) paragraphs for a reminder, but the outcome of a successful authentication is the creation of an `access_token` that links three resources (during a limited period of time):

- a user
- an application instance identified by a `client_id`
- scopes

Then, anytime the application instance tries to interact with user-related resources (typically due to user action) through the platform API, this `access_token` is sent to the platform so that it can check the operation is allowed.

That's how the platform provides both authentication and authorization features.

### Prerequesite
{: #s4-prerequesite}

You will [soon](#s4-1-authentication-request) see that you ask the platform to authenticate users against a `client_id`. Since this in an [outcome](#s3-1-ozwillo-request) of the provisioning process, you should implement provisioning first.

In the particular case where what you offer is not an instantiable application, but a unique and singleton one, your product may be considered as a service, and you may ask for a `client_id` to <a href="mailto:dev@sictiam.fr">dev@sictiam.fr</a>.

### Authorization code flow
{: #s4-auth-code-flow}

You may refer to the official OpenID Connect <a href="https://openid.net/specs/openid-connect-core-1_0.html#CodeFlowSteps" target="_blank">authorization code flow</a> in addition to this documentation.

Two particularities may be noted in comparison with the official spec:

- if there is a one-to-one mapping between `client_id`s and application instances, we also add a one-to-many relation between applications instances and services. It means that several services (end-user endpoints) may exist within the same `client_id`;

- the `id_token` sent by the platform as an outcome of [step #5](#s4-5-ozwillo-response) contains the <a href="https://openid.net/specs/openid-connect-core-1_0.html#IDToken" target="_blank">official</a> required fields, plus specific properties (`app_user` and `app_admin` boolean values).

Still within the context of the official spec, some optional features may not be implemented as of today, for instance passing a `max_age` parameter to the authentication endpoint.

#### #1 Authentication request
{: #s4-1-authentication-request}

In this step, the application instance has detected an authentication of the user is needed and delegates it to the platform by **redirecting** the end-user request to the platform authentication endpoint.

<pre>
HTTP/1.1 302 Found
Location: {authentication_endpoint}?{parameters}
</pre>

Knowing the platform <a href="https://accounts.ozwillo-preprod.eu/.well-known/openid-configuration" target="_blank">OpenID configuration</a>, the end-user navigator will issue the following request :

##### Request command

<pre>
GET /a/auth?{parameters} HTTP/1.1
Host: accounts.ozwillo-preprod.eu
</pre>

##### Query {parameters}

| Field name | Field description | Field type and format |
| :-- | :-- | :-- |
| **response_type** | determines the authorization processing flow, in our case the value is <a href="https://openid.net/specs/openid-connect-core-1_0.html#AuthRequest" target="_blank">always</a> "code" | string |
| response_mode | determines the mechanism used to send the response, in our case the value is always "query" | string |
| **client_id** | the `client_id` associated to the application instance | string |
| **scope** | list of scopes requested by the instance, it must at least contain `openid` | string (scopes separated by spaces) |
| **redirect_uri** | the redirection URI to which the response will be sent | URI string |
| **state** | opaque value used to maintain state between the request and the callback | string |
| **nonce** | unique random string used to mitigate replay attacks | string |
| code_challenge | code challenge for <a href="https://tools.ietf.org/html/rfc7636" target="_blank">PKCE</a> | string |
| code_challenge_method | if code_challenge is present, value must be "S256" | string |
| prompt | whether to prompt the user for authentication and/or consent; value can be "none", "login", and/or "consent" (space separated) | string |
| id_token_hint | ID Token previously issued by the platform being passed as a hint about the expected user | JWT string |
| max_age | allowable elapsed time in seconds since the last time the user was actively authenticated, failing that he'll be prompted to reauthenticate | integer |
| claims | request that specific claims about the user be returned (only userinfo requested claims are handled, id_token is ignored) | JSON string |
| ui_locales | user's preferred languages for the user interface (only used when not already authenticated) | string (BCP47 codes separated by spaces) |

The <a href="https://openid.net/specs/openid-connect-core-1_0.html#ClaimsParameter" target="_blank">`claims` parameter</a> value is a URL-escaped JSON string following a precise structure. It should have a `userinfo` property (any other property than `userinfo` will be ignored), whose value is another JSON object. Properties of that other object are claim names (unrecognized names will be ignored, so beware of typos!), and values can either be `null` or a JSON object with an `essential` boolean property. A `null` value is equivalent to an `{"essential":false}` object, and the `essential` property tells the platform that the claim is essential for the application (see the [introduction](#s1-claims) for what it means for a claim to be _essential_.)

As a result, a request could look like:

<pre>
GET /a/auth?
 response_type=code
 &client_id={client_id}
 &scope=openid
 &claims=%7B%22userinfo%22%3A%7B%22nickname%22%3Anull%2C%22locale%22%3Anull%7D%7D
 &redirect_uri=https%3A%2F%2Fapp.example.com%2Fcb
 &state=security_token%3D{random_value}%26url%3Dhttps%3A%2F%2Fapp.example.com%2Fhome
 &nonce={another_random_value} HTTP/1.1
Host: accounts.ozwillo-preprod.eu
</pre>

In this example, some characters (among which { " : , } / = &) have been URL-escaped.

For instance, the `claims` value, URL-unescaped, reads `{"userinfo":{"nickname":null,"locale":null}}`, which requests the `nickname` and `locale` claims as _voluntary claims_.

#### #2 PSN response
{: #s4-2-ozwillo-response}

Several operations are then conducted on the platform side:

1. validate the incoming authentication request from step #1;
2. authenticate the user;
3. ask user to consent to the scopes and claims requested by the instance.

The user experience (points 2 and 3) may vary if previous interaction between the users and the platform has occured or not: do they have an account (or do they have to register)? Are they currently logged to the platform? Have they previously granted *this* instance to use *those* scopes and/or claims? In particular, if all the answers to these questions are yes, 2 and 3 are validated silently by the platform, meaning that no sign-in or consent web page is shown.

In short, the authentication endpoint implements a rich behaviour and may not display the same user interface depending on an existing history between the end user and the platform through a given web client.

##### Success

If the previous operations succeed, and if the `redirect_uri` value specified in [step #1](#s4-1-authentication-request) matches one of the `redirect_uris` specified during the [provisioning acknowledgement](#s3-3-provider-acknowledgement), the following response will be sent:

<pre>
HTTP/1.1 302 Found
Location: https://app.example.com/cb?state=security_token%3D{random_value}%26url%3Dhttps%3A%2F%2Fapp.example.com%2Fhome&code={code}&session_state={session_state}
</pre>

This redirect workflow means your server will finally receive the following request sent from the end-user navigator:

<pre>
GET /cb?state=security_token%3D{random_value}%26url%3Dhttps%3A%2F%2Fapp.example.com%2Fhome&code={code}&session_state={session_state} HTTP/1.1
Host: app.example.com/
</pre>

##### Failure

The same `redirect_uri` callback will be notified of the authentication error according to the <a href="https://openid.net/specs/openid-connect-core-1_0.html#AuthError" target="_blank">spec</a>.

#### #3 Response validation
{: #s4-3-response-validation}

You should verify that the `security_token` (within the `state`) is the one you sent first to the platform during [step #1](#s4-1-authentication-request), to decide if you can trust the response. To do so, you should link it to the current user session. This operation ensures the user accessing the platform response is the same that initiated the authentication request.

If validation passes, you finally exchange the received `code` for an `access_token`. 

The `session_state` is an opaque value sent by the platform that can be used with <a href="https://openid.net/specs/openid-connect-session-1_0.html" target="_blank">OpenID Connect Session Management 1.0</a>.

#### #4 Requesting an access token
{: #s4-4-token-request}

The previous steps occured through the end-user navigator thanks to redirects. From now on, the interaction is done directly between the platform and provider APIs.

##### Request command

Referring to the `token_endpoint` declared in <a href="https://accounts.ozwillo-preprod.eu/.well-known/openid-configuration" target="_blank">OpenID configuration</a>, you should issue the following request:

<pre>
POST /a/token HTTP/1.1
Host: accounts.ozwillo-preprod.eu
<strong>Content-Type: application/x-www-form-urlencoded</strong>
Authorization: Basic {base64 encoding of client_id:client_secret}
</pre>

The authorization header needs to be set as described in [Calling the platform without an access_token](#s2-auth-without-token). As shown in the `Content-Type` header, the following request body is sent as a serialized form.

##### Request body

| Field name | Field description | Field type and format |
| :-- | :-- | :-- |
| **grant_type** | in our case the value is <a href="https://openid.net/specs/openid-connect-core-1_0.html#TokenRequest">always</a> "authorization_code" | string |
| **redirect_uri** | the redirection URI to which the response will be sent | URI string |
| **code** | the `code` sent to you in [step #1](#s4-2-ozwillo-response) | string |
| code_verifier | code verifier for <a href="https://tools.ietf.org/html/rfc7636" target="_blank">PKCE</a> | string |

#### #5 PSN response
{: #s4-5-ozwillo-response}

Similarly to step #2, the platform may accept or reject the previous request (especially depending on the `Authorization` header and the `code` parameter).

##### Success status and headers

In case of success, the following response will be sent:

<pre>
HTTP/1.1 200 OK
Content-Type: application/json
Cache-Control: no-store
Pragma: no-cache
</pre>

##### Success body

| Field name | Field description | Field type and format |
| :-- | :-- | :-- |
| **access_token** | access_token token value issued by the Authorization Server | string |
| **token_type** | in our case the value is <a href="https://openid.net/specs/openid-connect-core-1_0.html#TokenResponse">always</a> "Bearer" | string |
| scope | reminder of scopes associated with this access token | strings separated by spaces |
| expires_in | expiration time of the access token in seconds | integer |
| **id_token** | value associated with the authenticated session | JWT string |

##### Failure

If the platform rejects the previous request, a HTTP 400 error will be <a href="https://openid.net/specs/openid-connect-core-1_0.html#TokenErrorResponse" target="_blank">sent back</a>.

#### #6 Token decoding and validation
{: #s4-6-token-validation}

The `id_token` sent in the previous step is a JWT (JSON Web Token). You can do some tests by using this <a href="http://jwt.io/" target="_blank">online tool</a> or dedicated <a href="http://jwt.io/#libraries" target="_blank">libraries</a> (please check <a href="https://openid.net/developers/libraries/" target="_blank">these ones</a> too).

These libraries also help you check the `id_token` signature knowing Ozwillo public keys. The public keys URI is named `jwks_uri` in the <a href="https://accounts.ozwillo-preprod.eu/.well-known/openid-configuration" target="_blank">OpenID configuration</a>.

If the signatures do not match, you can not trust the JWT. If they do match, the JWT contains the following fields:

| Field name | Field description | Field type and format |
| :-- | :-- | :-- |
| **iss** | issuer identifier, typically "https://accounts.ozwillo-preprod.eu" | URI string |
| **sub** | subject identifier, meaning the platform user id | string |
| **aud** | audience, in our case the `client_id` | string |
| **iat** | time at which the JWT was issued | integer |
| **exp** | expiration time | integer |
| **nonce** | used to mitigate replay attacks | string |
| **app_user** | PSN specific, used to describe the user role (see [explanation](#s1-app-admin-app-user)) | boolean |
| **app_admin** | PSN specific, used to describe the user role (see [explanation](#s1-app-admin-app-user)) | boolean |

The last expected validation is that you check the `nonce` sent back in this response is the same you sent in [step 1](#s4-1-authentication-request).

If all goes well and depending on your use cases, you may now enrich and qualify your end-user session with the issued `access_token` and `id_token`. The `access_token` is especially useful for further interactions use the platform APIs, typically the user info endpoint.

The `access_token` must not be leaked in a client-side cookie.
{: .focus .important}

### Sign-out
{: #s4-sign-out}

Signing out a user is a 3-step process.

#### #1 Revoke known tokens

To prevent tokens from being used maliciously would they be leaked, the first step is to use the *revocation endpoint* as defined by <a href="https://tools.ietf.org/html/rfc7009" target="_blank">RFC 7009</a> to revoke all the tokens known to the application.

Authentication to the revocation endpoint is done using the same mechanism and credentials as at the token endpoint to exchange authentication codes for tokens.

##### Request command

Knowing the platform <a href="https://accounts.ozwillo-preprod.eu/.well-known/openid-configuration" target="_blank">OpenID configuration</a>, the request looks like:

<pre>
POST /a/revoke HTTP/1.1
Host: accounts.ozwillo-preprod.eu
<strong>Content-Type: application/x-www-form-urlencoded</strong>
Authorization: Basic {base64 encoding of client_id:client_secret}
</pre>

The authorization header needs to be set as described in [Calling the Kernel without an access_token](#s2-auth-without-token) and the request body is sent as form serialization.

##### Request body

| Field name | Field description | Field type and format |
| :-- | :-- | :-- |
| **token** | the value of the `access_token` | string |
| **token_type_hint** | the type of the token, in this case the string "access_token" | string |

#### #2 Invalidate the application's local session

Whichever mean the application uses to maintain the user session, it should be invalidated so that when the users come back to the application (and/or if they had the application opened in several windows or tabs), they won't be recognized and the authentication process is triggered again.

#### #3 Single sign-out

If the user is still signed in on the platform, going back to the service is going to transparently sign-in her/him back. Indeed the service redirects to the platform authentication endpoint (see the [authentication request](#s4-1-authentication-request)), and knowing a session exists between the user and the platform this step succeeds silently.

This would be a surprising behavior for the user so he must be given the choice to sign out from the whole platform. This is done using the *end session endpoint* as defined by <a href="https://openid.net/specs/openid-connect-session-1_0.html#RPLogout" target="_blank">RP-Initiated Logout in OpenID Connect Session Management 1.0</a>.

According to the platform <a href="https://accounts.ozwillo-preprod.eu/.well-known/openid-configuration" target="_blank">OpenID configuration</a>, you should then issue a redirect to conclude the sign-out:

<pre>
HTTP/1.1 302 Found
Location: https://accounts.ozwillo-preprod.eu/a/logout?{parameters}
</pre>

As a result, the end-user client will issue the following request:

##### Request command

<pre>
GET /a/logout?{parameters} HTTP/1.1
Host: accounts.ozwillo-preprod.eu
</pre>

##### Query {parameters}

| Field name | Field description | Field type and format |
| :-- | :-- | :-- |
| **id_token_hint** | the `id_token` JWT value that was issued at the end of authentication | string |
| **post_logout_redirect_uri** | the redirection URI to which the response will be sent | URI string |
| state | opaque value used to maintain state between the request and the callback | string |

The `post_logout_redirect_uri` value must match one of the `post_logout_redirect_uris` specified during the [provisioning acknowledgement](#s3-3-provider-acknowledgement).
{: .focus .important}