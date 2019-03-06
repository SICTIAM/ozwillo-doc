## Experiment
{: #s2-experiment}

### Preproduction sandbox
{: #s2-preproduction-sandbox}

The previous section (especially [Programming interface](#s1-terminology)) lists the production hosts of the platform, all of which being subdomains of `sictiam.fr`. Preproduction environment mirrors this setup under `ozwillo-preprod.eu` by providing the following hosts (thanks to the Ozwillo team that let us use freely its preproduction environment):

- `accounts.ozwillo-preprod.eu` for authentication pages and the accounts API root
- `data.ozwillo-preprod.eu` as the the Datacore API root (which also hosts its [live Playground](https://data.ozwillo-preprod.eu))
- `kernel.ozwillo-preprod.eu` as the root of other APIs
- `portal.ozwillo-preprod.eu` being the portal for logged users
- `www.ozwillo-preprod.eu` for information pages

As of today, the preproduction fulfills two different objectives:

- as its name implies, the preproduction may serve a different (more recent) software version than production, showcasing its evolution in a future update
- it's also the right place for providers to safely test and discover the APIs

That's why providers are asked to implement provisioning and authentication on preproduction first. Once validated, their applications are declared to production.

From now on, the documentation focuses on preproduction hosts, in particular in HTTP request samples. Hosts should be put in variables so that you can update them to the production ones when needed.
{: .focus .important}

### Online experimentation (Datacore)
{: #s2-online-experimentation}

The best and easiest way to discover, try out and learn about Datacore APIs is to use its live Playground user interface. Here is [the preproduction one](https://data.ozwillo-preprod.eu).

Its use is strongly advised to everybody. It is very useful to **app developers** before they attempt to code calls and implement authentication, in order to get an idea of their target even at the technical level thanks to the [Swagger technical Playground](https://data.ozwillo-preprod.eu/dc-ui/index.html#swagger) or by debugging calls made by their browser to the server. It is mandatory for **business model designers**, in order to find out in the Project Portal Playground ([manual](https://data.ozwillo-preprod.eu/dc-ui/index.html#playgroundUserManual)) already existing models and data, and afterwards to use the Import UI [documentation](https://github.com/SICTIAM/oasis-datacore/wiki/Playground-&-Import-UI-demo-scenario---Provto-&-OpenElec) in order to import their own CSV models along with validating sample data. The production one may also be useful to debug data problems.

In order to use it, one must **first ask administrators** for his account to be allowed to. To this purpose, use the contact form at the bottom of [a logged in portal page (preproduction)](https://portal.ozwillo-preprod.eu/my/network). This is because the Datacore is a regular application that is installed in the Ozwillo organization and therefore only available to its members.

### OpenID configuration
{: #s2-openid-configuration}

The platform provides API endpoints that implement OpenID Connect. Their configuration is accessible as a well-known registry resource: here is the <a href="https://accounts.ozwillo-preprod.eu/.well-known/openid-configuration" target="_blank">preproduction</a> one.

For instance, the configuration shows the `authorization_endpoint` is accessible at `https://accounts.ozwillo-preprod.eu/a/auth`

These endpoints are also listed in the [API reference](#s6-api-reference) section.

### Authentication schemes
{: #s2-authentication-schemes}

This paragraph provides an overview of authentication and authorization schemes that occur between the platform and provider APIs.

##### Recognize and trust the platform
{: #s2-trust-ozwillo}

As you will [soon](#s3-1-ozwillo-request) see, a purchase act on the portal causes the platform to POST provisioning requests to a provider endpoint. Encoding the payload of the request and sending the resulting signature as a HTTP header allows the provider to recognize the platform as the request issuer.

Knowing this request intent is to create a new application instance, it includes a `client_id`/`client_secret` pair used in the following authentication schemes.

##### Calling the platform without an access_token
{: #s2-auth-without-token}

Having an `access_token` means an end user has successfully [authenticated](#s4-user-authentication) to Ozwillo (within a given `client_id`). But there is a number of cases where the provider try to reach the platform before or outside an authenticated user context:

- precisely when [asking for](#s4-4-token-request) an `access_token` during the authentication flow
- when answering the provisioning request described in the previous paragraph
- when posting notifications or events

In those cases, the provider servers must issue HTTP requests to the platform with [basic authentication](https://tools.ietf.org/html/rfc2617#section-2) using the `client_id`/`client_secret` pair as userid and password respectively.

##### Calling the platform with an access_token
{: #s2-auth-with-token}

When the provider calls the platform with an `access_token`, it means the request is done on behalf a user. It helps the platform to decide if a particular operation is allowed depending on the user identity, [scopes]() and `client_id` associated with this `access_token`.

The corresponding requests require [OAuth 2.0 Bearer](https://tools.ietf.org/html/rfc6750#section-2.1) authentication, meaning that the `access_token` is sent in an `Authorization: Bearer {access_token}` HTTP header.

### Recommendations
{: #s2-recommendations}

##### HTTPS

Knowing that sensitive information may be exchanged between the platform and the provider APIs (`client_id`, `access_token`), it is required that communication happens over HTTPS.

It may occur that your HTTPS configuration is not trusted by the Kernel and thus that some requests are not even sent. In this case, please use an SSL report tool to check your configuration or refer to the <a href="https://www.eff.org/https-everywhere/deploying-https" target="_blank">EFF's guide</a>.

##### Robustness

Following the <a href=" https://en.wikipedia.org/wiki/Robustness_principle" target="_blank">robustness principle</a> of Jon Postel, your API endpoints should be prepared to receive more data than expected and described in the documentation. It especially enables the platform to add new functionality without breaking backward compatibility.

Your endpoints must be robust to new parameters or parameters not described in this document (in particular in response bodies and HTTP headers).
{: .focus .important}

##### Monitoring

In some cases, API calls may need a manual operation on the provider side (for instance during [provisioning](#s3-2-provider-provisioning) depending on the process you put in place) or may [fail](#ref-ack-422). It's good practice to be sure a real person is notified when such cases occur, and don't let the issue sleep.