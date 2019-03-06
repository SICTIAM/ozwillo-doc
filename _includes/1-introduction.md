## Introduction
{: #s1-introduction}

### User interface and portal features
{: #s1-ui}

From a user standpoint, the portal acts similarly to a mobile operating system, since it's possible to:

- find and install applications from the store
- launch installed services (see the difference between applications and services [below](#s1-terminology))
- manage one's personal and application settings

The main user interface components are:
{: #s1-ui-hosts}

- the portal (as defined above) and its store, served under `services.sictiam.fr`
- authentication-related pages served under `accounts.sictiam.fr`

Except from the store, the portal pages are reserved to registered and logged users. They especially give access to:

- one's desk, made of shortcut icons to launch services;
- one's network, to create and manage organizations and links within organizations. Thanks to this, it is possible for someone to **install applications on behalf organizations**, in addition to a personal use, if one is an admin of this organization;
- one's application settings, where one can give others access to a given service, typically an organization admin authorizing other organization members.

### Programming interface
{: #s1-programming-interface}

From a provider standpoint, adapting to the platform means:

1. include the application deployment process within the [provisioning protocol](#s3-provisioning) described below. This is dealt with requests exchanged between REST API and provider endpoints (typically over a REST API too)
2. if user authentication is needed, [delegate it](#s4-user-authentication) to the platform's OpenID Connect Identity Provider
3. create, update and consume linked and shared data through the Datacore REST API. The easiest way to try this out is to [use its live Playground](#s2-online-experimentation).

{: .focus .soft}

In other words, programming interface is made of:

- the API surface dedicated to authentication and authorization available under `accounts.sictiam.fr`
- the Datacore API available (along with its live Playground) under `data.sictiam.fr`
- other APIs (for instance for provisioning) available under `kernel.sictiam.fr`

As [previously](#s1-ui-hosts) introduced, the `accounts` subdomain also serves a few web pages: single sign-in, single sign-out, forgotten password...
{: .focus .soft}

APIs access is over HTTPS, providers endpoints must be too.
{: .focus .important}

### Terminology
{: #s1-terminology}

We make no asumption regarding what words final users will use between *applications* or *services*. It means that one could think "I am launching this app" when another would say "I am accessing this service".

But when it comes to the platform's APIs and thus to this documentation, they have a different and precise meaning. From the general to the particular:

<dl>
  <dt>Application</dt>
  <dd>an abstract application, not directly usable, declared in the catalog and visible in the store, that can be the object of an instantiation. It's the product you sell;</dd>
  <dt id="def-application-instance">Application instance</dt>
  <dd>or <strong>instance</strong> for short, a runnable copy of an application, created for a particular customer (which may be an individual or an organization) after a successful provisioning;</dd>
  <dt>Service</dt>
  <dd>an "endpoint" of an application instance, addressable through a URL (the service URI). Services are declared in the catalog and may or may not be visible in the store. An application instance is made of one or more services.</dd>
  <dt id="def-desk-shortcuts">Desk shortcuts</dt>
  <dd>The icons shown in a user's desk (under the portal) are links to services URIs.</dd>
</dl>

Additional definitions to complement the picture:

<dl>
  <dt id="def-catalog">Catalog</dt>
  <dd>a database of all applications, instances and services. It is used internally by the portal to display a user’s desktop, to browse the app store, etc.;</dd>
  <dt>Store</dt>
  <dd>a system that allows users to: find services that are publicly available and add them to their desktops, or find applications that are instantiable through provisioning. The store is the visible surface of the catalog;</dd>
  <dt id="def-store-entry">Store entry</dt>
  <dd>A visible application or a visible service (but not an application instance, whose visible surface is always a service);</dd>
  <dt id="def-purchase-act">Purchase act</dt>
  <dd>The act of requesting the installation of a story entry (charged or free), done by a portal end user;</dd>
  <dt id="def-purchaser">Purchaser</dt>
  <dd>A user that performed a purchase act;</dd>
  <dt id="def-app-factory">App factory</dt>
  <dd>A REST API (implemented by the provider) that the platform can call to initiate the provisioning of an application instance, after a purchase act has occured.</dd>   
</dl>

What may be confusing at first is that both *abstract* — not instantiated — applications and *tangible* — instantiated — services can be found in the store. From a user standpoint it makes sense: one does not worry about the technical implications behind a purchase act. But since the documentation reader cares, here is the difference:

- installing an application triggers the provisioning protocol, leading to software deployment on the provider side, and then to a declaration of the created instance to the platform. It requires communication between pla he platform and the provider servers;
- installing a service is simply the process of bookmarking the service URI as a user desk shortcut. This is instantaneous and does not require communication between the platform and the provider servers.

### Visibility and access control
{: #s1-visibility-restricted}

If a store entry is visible, it only means it is publicly referenced and shown under the store.

A `visible:false` (hidden) application may be under review or not be available at this time. A `visibility:"HIDDEN"` service is typically targeted to certain users (depending on who purchased the instance): it is only shown as a desk shortcut for given authorized users, but not listed in the store. Visibility of a service can be toggled from the portal by the service owner, unless the provider declared it `visibility:"NEVER_VISIBLE"`.

The fact that a service is visible or not in the store is yet separated to how its access is managed, possibly being restricted to certain users. All of these scenarios are possible:

- a visible service whose access is restricted to given users (for example if the service owner wants to explicitely grant access to requesting users)
- a hidden service with public access (for example if the service owner does not want to reference it on the platform's store)
- more typically services that are either visible and public, or hidden and restricted

Similarly to the visibility of a service, the access control can also take 3 values: two that can be toggled from the portal by the service owner, unless the provider declared the service as `access_control:"ALWAYS_RESTRICTED"`.

The [Provisioning](#s3-provisioning) section shows how these settings are defined.

### Authorized users
{: #s1-authorized-users}

### `app_admin` vs `app_user`
{: #s1-app-admin-app-user}

The case for _restricted_ services is that only specific users are allowed to access and interact with them. By default, the user that purchased the instance is an `app_admin`, meaning that s/he has the right to add new `app_admin` or `app_user` users.

Both `app_admin` and `app_user` are considered authorized users of the instance, but an `app_user` can not add other authorized users. These rules (who can add who) are internal to the platform and frame the organization management under the portal.

As a provider, you will also get this info (is the user an `app_admin` or `app_user`) and are free to give it whatever meaning best suits your application behavior. Yet, on application instances side, it's typical that `app_admin` maps the role with the highest privileges and is allowed to configure the roles of others (`app_user`). Then `app_user` may hold a variety of different user types from the application point of view. Since the platform can not know all business-specific roles of all applications, you can define a finer granularity of users under the global `app_user` concept linked to the platform.

To sum-up, it is typical (but not imposed) that on the application side:

- `app_admin` characterises users with the highest privileges
- `app_user` is generic and may apply to different business roles (for instance: moderator, editor...)

Now, when your services are accessed by internet users, how do you know they are indeed authorized users (`app_admin` or `app_user`)? This will be answered in depth in the [User Authentication](#s4-user-authentication) section. But let's give a glimpse of it: it will also helps introduce the *scope* concept.

### Authorization in brief
{: #s1-authorization}

Let's say an internet user arrives at one of the URIs corresponding to the protected area of your service, typically the service URI entrypoint.

Without the platform, you would check that you know this user (there is an existing session) and if not trigger authentication to check if s/he can identify as an authorized user.

With the platform, you delegate authentication and authorization: the platform is both an identity provider and an authorization server. It means that without a valid session on your server, you redirect the user to the platform's authentication page, passing it interesting parameters like:

- a `client_id` that helps identify the application instance in which the service resides
- a `scope` list that will be explained in the next paragraph
- and others that will be detailed in the [User Authentication](#s1-visibility-restricted) section

Let's focus on the `client_id`: thanks to it, the platform knows what application instance the user is trying to access. The platform then checks if s/he is indeed a valid `app_admin` or `app_user` of this instance, and now there are two cases depending on the service configuration:

- if the service is `restricted:true` authentication will only succeed for an `app_admin` or `app_user` (you can rely on it)
- if the service is `restricted:false` authentication will succeed even if the user is neither an `app_admin` nor a `app_user`, but the service will be notified of it

There are other reasons for the authentication and authorization to fail: for instance users may refuse to accept the [scopes](#s1-scopes) or [claims](#s1-claims) requested by your instance.
{: .focus .soft}

If authentication is successful, your service will be given at the end of the [authorization code flow](#s4-auth-code-flow) two tokens:

- an `id_token` assessing authentication is successful and giving information about the user identity
- an `access_token` which authorizes you to access and interact with the platform APIs (depending on granted scopes) on behalf this user, during a limited period

On your server, you may associate a user session with these two tokens to deal with forthcoming requests to the platform APIs, but they must not be leaked client-side.

In short, you can:

- restrict the access to certain web pages depending on the ability to retrieve an `id_token`
- possibly refine capabilities offered to the user depending on `app_admin` and `app_user` properties (which are wrapped in the `id_token`)
- ask to access API resources on behalf the user thanks to the `access_token`

### Scopes
{: #s1-scopes}

First of all, the platform as an authentication and authorization server is an implementation of <a href="https://openid.net/connect/" target="_blank">OpenID Connect</a> Authorization Server, and as such uses scopes introduced by the protocol to specify access privileges.

For instance and if we focus on scopes <a href="https://openid.net/specs/openid-connect-basic-1_0.html#Scopes" target="_blank">introduced</a> by OpenID Connect, the provider may ask for an `access_token` with the `email` scope. When authenticating, users are prompted to know if they want to share their email address with this instance. As a result, the `access_token` created is associated to this scope (if granted by user) on the platform side.

When the instance wants to indeed access the email through one of API endpoints, it will send an HTTP request with this `access_token`, and the platform is then able to decide if the operation is permitted or not.

As you will see later, you can ask for scopes at several occasions:

- during the application instance installation (users will globally accept or deny it for their future usage of the application instance)
- when accessing a service (prompt during authentication) and for the lifetime of the `access_token`
- on a per action basis

When a requested scope is refused by users, the instance will be able to ask for it again later, and explain a given operation won't be possible until they accept it.

Despite the example above about accessing private profile information (the email address), scopes are typically used to access the [Datacore](#s5-datacore), and are a flexible enough mechanism that allows application instances to declare their own during [provisioning](#s3-3-provider-acknowledgement-scope) to authorize inter-application requests.

When it comes to private profile information, there's actually a better mechanism:

### Claims
{: #s1-claims}

Claims represent information about the user. OpenID Connect defines a list of <a href="https://openid.net/specs/openid-connect-core-1_0.html#StandardClaims" target="_blank">standard claims</a> (most of which are implemented by Ozwillo), and a number of <a href="https://openid.net/specs/openid-connect-core-1_0.html#ScopeClaims" target="_blank">scopes</a> ([see also above](#s1-scopes)) that grant access to groups of claims; but those permissions can also be <a href="https://openid.net/specs/openid-connect-core-1_0.html#ClaimsParameter" target="_blank">requested on a per-claim basis</a>.

For instance, instead of asking for the `profile` scope (which grants access to the user's names, locale, date of birth, gender, etc.), the provider can individually ask for the claims it really needs; for example `nickname` and `locale`.

This feature is particularly important in face of the GDPR (and other similar laws), as it helps with _data minimization_: only collecting that personal data which is needed to fulfill the service, and no more.

Additionally, some claims can be requested as being _essential_ to service, and the user will not be able to access it until he fills in his profile. This feature should only be used if your application would fail in the absence of value for those claims, as could be the case when adapting existing applications to the platform; otherwise you're strongly encouraged to make your applications resilient to the absence of value for given claims, either by using default values instead, or by asking the user. As terminology goes, a claim that is not requested as _essential_ is said to be _voluntary_.

### Documentation conventions
{: #s1-conventions}

HTTP requests and responses bodies (and data models) are described in tables. Within these tables the first column is dedicated to parameter names; when they are displayed in **boldface**, it means they are mandatory.

When you see curly brackets {} in a code sample, it means the content should be evaluated. Curly brackets typically contain variable names or describe an operation, for example:

<pre>
POST {instantiation_uri}
X-Hub-Signature: sha1={HMAC-SHA1 digest of payload}
</pre>
