## Provisioning
{: #s3-provisioning}

This section explains the provisioning protocol triggered between the platform and the provider APIs when a [purchase act](#def-purchase-act) occurs on the portal. The outcome of a successful provisioning is an application instance that is both:

1. allocated and reachable on the provider servers
2. declared with appropriate settings, especially regarding access rights

where (2) enables the creation of [desk shortcuts](#def-desk-shortcuts) so that users can access (1)
{: .where}

The creation of an application instance on the provider side may imply configuration changes or new resources allocation. These implementation details are left unconstrained: the protocol focuses only on API communication. It means existing deployment operations does not have to be affected and should rather be exposed through this protocol.

### Prerequesite
{: #s3-prerequesite}

Since provisioning starts when a user performs a purchase act, it means the purchased application is declared in the [catalog](#def-catalog) **and** the platform knows how to forward the request to the provider through its [app factory](#def-app-factory).

Thus the following information is needed to have a well described *and* installable application in the catalog:

##### Commercial information
{: #s3-commercial-info}

| Field name | Field description | Field type and format |
| :-- | :-- | :-- |
| **name** | name in the default language (try to remain below 20 characters) | string |
| name#{l} | name in the given language {l} (try to remain below 20 characters) | string |
| **description** | description in the default language | markdown string |
| description#{l} | description in the given language | markdown string |
| **tos_uri** | terms of service URI implicitly accepted on purchase, in the default language | URI string |
| tos_uri#{l} | terms of service URI implicitly accepted on purchase, in the given language | URI string |
| **policy_uri** | privacy policy URI implicitly accepted on purchase, in the default language | URI string |
| policy_uri#{l} | privacy policy URI implicitly accepted on purchase, in the given language | URI string |
| **icon** | app icon URI in the default language, a 64px x 64px png is expected | URI string |
| icon#{l} | app icon URI in the given language, a 64px x 64px png is expected | URI string |
| screenshot_uris | list of screenshot URIs, a 850px x 450px png are expected | array of URI strings |
| **contacts** | list of URLs to contact the provider or its support | array of URI (URL or mailto) strings |
| supported_locales | list of supported locales (end-user UI) in the application | array of `{l}` strings |
| geographical_areas | optional geographical areas of interest | array of territory_ids |
| restricted_areas | if the application can only be purchased by organizations in given areas | array of territory_ids |

A few comments on this table:

- by default the store will only present the applications that support (`supported_locales`) the user preferred language (the user may disable this restriction);
- `{l}` identifies a language following <a href="https://tools.ietf.org/html/bcp47" target="_blank">BCP 47</a>. Some examples regarding the fields introduced above: `name#es`, `description#fr-FR`, `description#fr-BE`. When looking up values, matching is currently done following the <a href="https://docs.oracle.com/javase/7/docs/api/java/util/ResourceBundle.Control.html#getCandidateLocales(java.lang.String,%20java.util.Locale)" target="_blank">Java algorithm for finding resource bundles</a>; it might change in the future to follow the <a href="http://www.unicode.org/reports/tr35/#LanguageMatching" target="_blank">Unicode Consortium rules for language matching</a>.
- the *default* language information is the fallback in case the user preferred language information is not available (no matching custom `{l}` version). Following <a href="https://tools.ietf.org/html/rfc2277#section-4.5" target="_blank">BCP 18</a> advices, it should “be understandable by an English-speaking person”; this also matches the <a href="http://www.unicode.org/cldr/charts/latest/supplemental/likely_subtags.html#und">CLDR Likely Subtags</a> for when the locale cannot be determined;
- <a href="http://daringfireball.net/projects/markdown/syntax" target="_blank">here</a> is a description of the markdown syntax. In particular, you may include raw text separated by two line breaks to shape paragraphs.

##### Store filters
{: #s3-store-filters}

| Field name | Field description | Field type and format |
| :-- | :-- | :-- |
| **payment_option** | "FREE" or "PAID" setting | string |
| **target_audience** | is the application intended to "CITIZENS", "PUBLIC_BODIES" and/or "COMPANIES" | array of strings |
| category_ids | IDs of the application store categories, not supported for the moment | array of strings |
| visible | if false, the application is not visible in the application store | boolean |

##### App factory

| Field name | Field description | Field type and format |
| :-- | :-- | :-- |
| **instantiation_uri** | instantiation endpoint | URI string |
| **instantiation_secret** | secret used to compute the instantiation request signature | string |
| **cancellation_uri** | cancellation endpoint, to cancel pending instantiations | URI string |
| **cancellation_secret** | secret used to compute the cancellation request signature | string |

There is no official spec about secret strings, but 30 characters within a rich character set (richer than hexadecimal) can be seen as a bare minimum. Here is a <a href="https://www.grc.com/passwords.htm" target="_blank">tool</a> to generate random strings.

##### Summary

You will likely need to read the remainder of this section to fully understand how fields are used, especially those related to the app factory. It means these setup steps are required before receiving provisioning requests :

1. gather all parameters info needed as described in the previous tables
2. send it to <a href="mailto:dev@sictiam.fr">dev@sictiam.fr</a> along with your display name as an application provider (typically the name of your company)
3. you will be notified when the application is made available on the [preproduction](#s2-preproduction-sandbox)

To ease the process, you may submit a first and simplified version of the commercial info fields, focusing on simple versions of **required** fields, and resubmit it later with enriched contents.

### Protocol
{: #s3-protocol}

#### #1 PSN request
{: #s3-1-ozwillo-request}

##### Description

When a purchase act occurs, the platform creates a new [application instance](def-application-instance) in its catalog, in a pending state (since it has not been provisioned for the moment). This instance is given a unique and constant `instance_id` and credentials (`client_id` and `client_secret`). The `client_secret` must remain to be known only by the platform and the provider.

The current behavior of the platform is that `instance_id` and `client_id` are given the same unique GUID value, but this is really an implementation detail subject to change. The spec is that `instance_id` is an identifier that won't change over time, when `client_id` may be refreshed.
{: .focus .important}

##### Request command

Don't forget to read our [conventions](#s1-authorized-users) and [recommendations](#s2-recommendations).
{: .focus .soft}

The following HTTP request is sent from the platform to the provider at `instantiation_uri` (decomposed in `instantiation_path` and `instantiation_host` below). 

<pre>
POST {instantiation_path} HTTP/1.1
Host: {instantiation_host}
Accept: application/json, application/*+json
X-Hub-Signature: sha1={HMAC-SHA1 digest of payload}
Content-Type: application/json;charset=UTF-8
</pre>

The body of this request is encoded in JSON. A HMAC-SHA1 hash of the body is computed using the application's `instantiation_secret` as key. This computed HMAC-SHA1 signature is then inserted in the `X-Hub-Signature` header in hexadecimal, prefixed with the string `sha1=` (as per <a href="https://pubsubhubbub.github.io/PubSubHubbub/pubsubhubbub-core-0.4.html#authednotify" target="_blank">PubSubHubbub Core 0.4</a>).
{: #ref-hmac-signature}

As introduced in [Recognize and trust the platform](#s2-trust-ozwillo), the provider must recompute the signature of the request body with the same secret key and method. If signatures match, this reasonably certifies the instantiation request comes from the platform (which is the only one knowing the `instantiation_secret`) and then can be trusted. If not, you may log the request for further analysis but you must not process it. Note that the `sha1=` prefix must be compared case-sensitively, but the case of the hexadecimal signature value doesn't matter.

##### Request body

| Field name | Field description | Type |
| :-- | :-- | :-- |
| **instance_id** | unique and constant identifier of the instance | string |
| **client_id** | used for authentication purposes | string |
| **client_secret** | used for authentication purposes | string |
| **user** | a description of [purchaser](#def-purchaser), containing at least its user id | User object |
| organization | a description of the organization, containing at least organization id and name | Organization object |
| **instance_registration_uri** | the endpoint that must be used to acknowledge the provisioning back to the platform | URI string |
| **authorization_grant** | an OAuth 2.0 authorization grant to be able to access the Datacore during the provisioning (e.g. to get details about the organization) | AuthorizationGrant object |
{: .request}

###### Embedded User object

| Field name | Field description | Type |
| :-- | :-- | :-- |
| **id** | user id of the purchaser | string |
| **name** | (non unique) display name of the purchaser | string |
{: .request}

###### Embedded Organization object

| Field name | Field description | Type |
| :-- | :-- | :-- |
| **id** | organization id of the purchasing organization | string |
| **name** | organization name | string |
| **type** | "PUBLIC_BODY" or "COMPANY"  | string |
| dc_id | Datacore identifier of the organization | string |
{: .request}

It's important to know that purchasers may install an application either on behalf an organization, or for their personal use. In the latter case, there is no organization associated to the purchase, so there is no `organization` field in the request body.

That said, if your application's `target_audience` (as declared in [store filters](#s3-store-filters)) does not contain `CITIZENS`, it means purchase acts will always be on behalf of an organization and thus the `organization` field will always be sent.

###### Embedded AuthorizationGrant object

| Field name | Field description or fixed value |
| :-- | :-- |
| **grant_type** | `urn:ietf:params:oauth:grant-type:jwt-bearer` |
| **assertion** | a JWT to be sent as-is to the Kernel's Token Endpoint |
| **scope** | scope of the access token you can obtain from the JWT, space separated (fixed to the value `datacore`) |
{: .request}

You can use those field values to obtain an [access token](#s2-auth-with-token) and make requests to the Datacore during the provisioning. To do that, make a `POST` request to the `token_endpoint` similar to [the one](#s4-4-token-request) made when authenticating users,
using the fields values as-is.

<pre>
POST /a/token HTTP/1.1
Host: accounts.ozwillo-preprod.eu
Content-Type: application/x-www-form-urlencoded
Authorization: Basic {base64 encoding of client_id:client_secret}

grant_type={grant_type}&assertion={assertion}&scope={scope}
</pre>

The authorization header needs to be set as described in [Calling the platform without an access_token](#s2-auth-without-token).

You should then receive a successful response with the following fields in the JSON payload:

| Field name | Field description | Field type and format |
| :-- | :-- | :-- |
| **access_token** | access_token token value issued by the Authorization Server | string |
| **token_type** | in our case the value is <a href="https://openid.net/specs/openid-connect-core-1_0.html#TokenResponse">always</a> "Bearer" | string |
| scope | reminder of scopes associated with this access token | strings separated by spaces |
| expires_in | expiration time of the access token in seconds | integer |

##### Response from provider

Accepted status codes:

- 2xx: request acknowledged, the app factory will do the job
- 4xx: request cannot be honoured (for instance, the organization already owns an instance of this application and it is limited to one)
- any other non-2xx will be declared a failure too (maybe we'll follow redirect, but let's say for now that we won't)

#### #2 Provider provisioning
{: #s3-2-provider-provisioning}

By provider provisioning we mean the installation process needed on the provider servers. As previously explained in the [introduction](#s3-provisioning) to this section, this step can be implemented in various ways:

- fully automatic (for instance - but not restricted to - a multitenant architecture where a single software instance manages several application instances)
- half automatic (requires a manual validation)
- fully manual (requires a manual installation)

Moreover this process also depends on existing software and architectural choices. That being said, we can give the following guidelines:

- there should be an internal process that stores [step #1](#s3-1-ozwillo-request) request properties and link them to services declared in [step #3](#s3-1-ozwillo-request). This configuration will be useful during authentication (for instance, you will soon see that a service accessible at a given `service_uri` has to know to what `instance_id` it belongs);
- you should be able to treat both preproduction and production provisioning requests, and identify them as such. Knowing you will start implementing provisioning against Ozwillo preproduction, and then move it to production while keeping the preproduction tests up, you need to know to what API [hosts](#s2-preproduction-sandbox) your servers communicate with.

From a user perspective, until the [step #3](#s3-3-provider-acknowledgement) is done, purchase acts are materialized as grey and disabled desktop shortcuts.

#### #3 Provider acknowledgement
{: #s3-3-provider-acknowledgement}

##### Description

In this step, the provider declares the precise configuration of the installed instance, including in particular:

- services composing the instance (the user endpoints)
- scopes needed for the instance to work (for instance the OpenID Connect `email` scope if you need to know the user email to configure the instance)
- scopes declared by the instance, so that other instances may require them as needed if they want to use this very instance exposed API

This list is not exhaustive and is detailed in the <span class="h5">Request body</span> paragraph below.

##### Request command

The following HTTP request is sent from the provider to the platform at the `instance_registration_uri` received in [step #1](#s3-1-ozwillo-request) (decomposed in `instance_registration_path` and `instance_registration_host` below).

<pre>
POST {instance_registration_path} HTTP/1.1
Host: {instance_registration_host}
Accept: application/json, application/*+json
Authorization: Basic {base64 encoding of client_id:client_secret}
Content-Type: application/json;charset=UTF-8
</pre>

The authorization header needs to be set as described in [Calling the platform without an access_token](#s2-auth-without-token).

##### Request body

| Parameter name | Parameter description | Type |
| :-- | :-- | :-- |
| **instance_id** | application instance id | string |
| **services** | services (at least one) to be declared on the platform, more details below | array of Service objects |
| **destruction_uri** | destruction endpoint of this instance | URI string |
| **destruction_secret** | secret used to compute the destruction request signature | string |
| status_changed_uri | status-changed endpoint of this instance | URI string |
| status_changed_secret | secret used to compute the status-changed request signature | string |
| needed_scopes | scopes needed by the instance | array of NeededScope objects |
| scopes | scopes declared by the instance | array of Scope objects |

**Embedded Service objects**
{: #s3-3-provider-acknowledgement-service}

| Field name | Field description | Type |
| :-- | :-- | :-- |
| **local_id** | identifier that needs to be unique within the instance, for instance "front-end" | string |
| **name** | name in the default language (try to remain below 20 characters) | string |
| name#{l} | name in the given language (try to remain below 20 characters) | string |
| **description** | description in the default language | markdown string |
| description#{l} | description in the given language | markdown string |
| **tos_uri** | terms of service URI implicitly accepted on purchase, in the default language | URI string |
| tos_uri#{l} | terms of service URI implicitly accepted on purchase, in the given language | URI string |
| **policy_uri** | privacy policy URI implicitly accepted on purchase, in the default language | URI string |
| policy_uri#{l} | privacy policy URI implicitly accepted on purchase, in the given language | URI string |
| **icon** | service icon URI in the default language, a 64px x 64px png is expected | URI string |
| icon#{l} | service icon URI in the given language, a 64px x 64px png is expected | URI string |
| screenshot_uris | list of screenshot URIs, a 850px x 450px png are expected | array of URI strings |
| **contacts** | list of URLs to contact the provider or its support | array of URI (URL or mailto) strings |
| supported_locales | list of supported locales (end-user UI) by the service | array of {l} strings |
| geographical_areas | optional geographical areas of interest | array of territory_ids |
| restricted_areas | if the service can only be purchased by organizations in given areas | array of territory_ids |
| **payment_option** | "FREE" or "PAID" setting | string |
| **target_audience** | is the service intended to "CITIZENS", "PUBLIC_BODIES" and/or "COMPANIES" | array of strings |
| category_ids | IDs of the service store categories, not supported for the moment | array of strings |
| visibility | whether the service is visible in the application store ("VISIBLE" or "HIDDEN"), and whether this can be toggled by an app_admin from the portal ("NEVER_VISIBLE"); defaults to "HIDDEN" | string |
| access_control | whether only members (app_user or app_admin) have access to the service or it's open to anyone ("RESTRICTED" or "ANYONE"), and whether this can be toggled by an app_admin from the portal ("ALWAYS_RESTRICTED"); defaults to "RESTRICTED" | string |
| **service_uri** | URL entrypoint of the service | URI string |
| notification_uri | endpoint used when there are notifications for this service | URI string |
| **redirect_uris** | whitelist of authentication callbacks, each URI must be unique to this service within the instance | array of URI strings |
| post_logout_redirect_uris | whitelist of post-logout callbacks, each URI must be unique to this service within the instance | array of URI strings |

{::comment}Document subscription_uri and subscription_secret when they're implemented.{:/}

**NB**: `redirect_uris` and `post_logout_redirect_uris` values can't be shared by different services within the same application instance. In some cases the platform may identify a service thanks to either `instance_id` + `redirect_uri` or `instance_id` + `post_logout_redirect_uri` pairs. 
{: .focus .important}

A few remarks on this table:

- you may compare it with the declaration of [applications](#s3-prerequesite) in the catalog, and find many common features due to the fact both applications and services are [store entries](#def-store-entry);
- `redirect_uris` and `post_logout_redirect_uris` are related to [User authentication](#s4-user-authentication).

Older versions of the Kernel used boolean properties `visible` and `restricted` in place of `visibility` and `access_control`, that only allowed to model three different states (a restricted service was necessarily hidden from the application store). For backwards compatibility, those old properties are still accepted, and will have precedence over the new ones as soon as one of them is explicitly present in the payload (this means that the absence of `visible` and `restricted` is no longer equivalent to having any one of `visible: false` and/or `restricted: false`, in case there are also a `visibility` or `access_control` property.) Similarly, the old properties will be returned in payloads if they can describe the state of the new properties' values. This means the API should be entirely backwards compatible. You are strongly encouraged to update your app factories and applications to the new model though, as the old properties are deprecated and will eventually be removed. The table below maps between the old values and the new ones; on reading a service definition, any other combination of `visibility` and `access_control` will result in the `visible` and `restricted` properties being absent:

| | | visibility | access_control |
| visible | restricted | | |
| :-: | :-: | :-: | :-: |
| false | false | HIDDEN | RESTRICTED |
| true | false | VISIBLE | ANYONE |
| false | true | NEVER_VISIBLE | ALWAYS_RESTRICTED |

**NB**: due to this backwards-compatibility rules, providers **should** remove/delete the `visible` and `restricted` properties from any payload they send to the platform, as they otherwise risk to see the `visibility` and `access_control` properties being ignored.
{: .focus .important}

**Embedded NeededScope objects**

| Field name | Field description | Type |
| :-- | :-- | :-- |
| **scope_id** | full identifier of the needed scope | string |
| **motivation** | motivation for using the scope, in the default language | string |
| motivation#{l} | motivation for using the scope, in the given language | string |

The motivation helps users understand why they should grant specific privileges (associated to the `scope_id`) to the instance, and thus help them decide if they will.

**Embedded Scope objects**
{: #s3-3-provider-acknowledgement-scope}

| Field name | Field description | Type |
| :-- | :-- | :-- |
| **local_id** | local identifier of this scope (within the instance) | string |
| **name** | name of the scope in the default language | string |
| name#{l} | name of the scope in the given language | string |
| **description** | description of the scope in the default language | string |
| description#{l} | description of the scope in the given language | string |

Scope identifiers are simple strings that correspond to the permission scopes that client applications require to access part of this API's functionality. For instance, application X may define an API to create events within a particular instance Y; it would then declare the scope "addevent" of this instance.

When a third party instance Z wants to add an event to Y on behalf of a user, it needs to require the `{Y instance_id}:addevent` as part of its token negotiation (the user is asked to confirm that they want to give the "add events on Y" permission to Z).

The full identifier of an instance scope is in the form `{instance_id}:{local_id}`.

##### Response from the platform

Possible status codes:

- 201: means the acknowledgement is valid and processed;
- <span id="ref-ack-422">422</span>: means there is a problem with the acknowledgement request, additional information about the particular error is given in the response body.

For successful 201 responses, the body response is in the form of:

<pre>
{
  "back-end": "a15243e3-17a0-4511-b15c-c88e6784e287",
  "front-end": "31336385-f2ff-4488-8835-1f7da53669b9"
}
</pre>

Where `back-end` and `front-end` would be the `local_id` of two declared services within the instance, and the corresponding GUIDs being their internal ids.

The 201 responses also include a `Location:` header with the URI of a resource about the created instance ([see below](#s6-api-app-instances)).

##### Effects

At this stage:

- the application instance is created;
- its services are declared in the catalog;
- its scopes are declared and available for use by other applications;
- the purchaser has desktop shortcuts for the created services;
- `visible: true` services are listed in store.

#### #3bis Provider dismiss
{: #s3-3bis-provider-dismiss}

If the instance creation has failed for some reason during [step #2](#s3-2-provider-provisioning), the provider must issue a DELETE request on the instance registration URL so that the platform does not show endless "pending" instance creation requests:

##### Request command

The following HTTP request is sent from the provider to the platform. 

<pre>
DELETE /apps/pending-instance/{instance_id} HTTP/1.1
</pre>

#### Instance status change
{: #s3-status-change}

Instance status change is not part of the initial provisioning that occurs between steps #1 to #3, but will occur whenever an admin of the organization that purchased the application decides to destroy it.

As of March 10th, this is only available in preproduction.
{: .focus .soft}

##### Request command

The following HTTP request is sent from the platform to the provider at the `status_changed_uri` defined in [step #3](#s3-3-provider-acknowledgement) (decomposed in `status_changed_path` and `status_changed_host` below)

<pre>
POST {status_changed_path}
Host: {status_changed_host}
X-Hub-Signature: sha1={HMAC-SHA1 digest of payload}
Content-Type: application/json;charset=UTF-8
</pre>

This scheme allows sharing the same status-changed URI and secret among several instances, but the platform also supports having separate status-changed URI and secret for each instance. See more on how to check if this request reliably comes from the platform depending on [verifying the signature](#ref-hmac-signature).

##### Request body

| Field name | Field description | Type |
| :-- | :-- | :-- |
| instance_id | identifier of the instance | string |
| status | new status of the instance: either `STOPPED` or `RUNNING` | string |

##### Response from provider

The status-changed endpoint must respond with a successful status (200, 202 or 204) in a timely manner (not necessarily waiting for the underlying resources to be actually released/archived or reacquired/restored). The platform will then change the instance's state from its database and it will be impossible for users to authenticate to it, and services will disappear from the store.

If the request times out, the Kernel will change the instance's state in its database nevertheless. Any (timely) non-successful status will abort the status change (so it can be retried later).

Note that the above does not describe the current behavior. Currently, the response is ignored, and the provider might not even be notified! This means that a `STOPPED` instance should continue to redirect users to the platform for authentication, and should treat a successful authentication as a signal that the instance actually is now `RUNNING` (remember that when an instance is in a `STOPPED` status, it's impossible to authenticate on it).
{: .focus .important}

#### Instance destruction
{: #s3-destruction}

Instance destruction happens one week after the instance status has been changed to `STOPPED` (and has not been changed back to status `RUNNING` in the mean time).

##### Request command

The following HTTP request is sent from the platform to the provider at the `destruction_uri` defined in [step #3](#s3-3-provider-acknowledgement) (decomposed in `destruction_path` and `destruction_host` below).

<pre>
POST {destruction_path}
Host: {destruction_host}
Accept: application/json, application/*+json
X-Hub-Signature: sha1={HMAC-SHA1 digest of payload}
Content-Type: application/json;charset=UTF-8
</pre>

Similarly to status changes, this scheme allows sharing the same destruction URI and secret among several instances, but the platform also supports having separate destruction URI and secret for each instance. See more on how to check if this request reliably comes from the platform depending on [verifying the signature](#ref-hmac-signature).

##### Request body

| Field name | Field description | Type |
| :-- | :-- | :-- |
| instance_id | identifier of the instance | string |

##### Response from provider

The destruction endpoint must respond with a successful status (200, 202 or 204) in a timely manner (not necessarily waiting for the underlying resources to be actually released or archived). The platform will then delete the instance (and its associated resources, like services) from its database and it will be impossible to undo the change.

If the request times out, the Kernel will delete the instance from its database nevertheless. Any (timely) non-successful status will abort the destruction (so it can be retried later).

### FAQ
{: #s3-faq}

**What is the difference between cancellation and destruction URIs?**

Pending instances (those for which a request has been made to the app factory, but for which no [acknowledgement](#s3-3-provider-acknowledgement) has been sent) can be canceled using the same mechanism as [Instance destruction](#s3-destruction) but at the application wide `cancellation_uri`.

Note that because the `instance_id` is sent in the request payload, the `cancellation_uri` and the `destruction_uri` for each instance (and their secrets) can technically be the same. If you choose to make them different, then the `cancellation_uri` should reject the destruction of provisioned instances.

TODO: use cases examples.
{: .todo}
