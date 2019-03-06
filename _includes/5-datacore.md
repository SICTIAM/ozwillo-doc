## Datacore
{: #s5-datacore} 

As said previously, the best and easiest way to discover, try out and learn about Datacore APIs is to use [its live Playground (preproduction)](https://data.ozwillo-preprod.eu).

In addition to going through the Playground's live examples, be sure so read the various other documentation available on the [Datacore wiki](https://github.com/SICTIAM/ozwillo-datacore/wiki).

Using Datacore APIs requires the `datacore` [scope](#s1-scopes).

Have a look at the Datacore's [API reference](https://data.ozwillo-preprod.eu/dc-ui/index.html#swagger) in its live Swagger technical Playground ([how to get access](#s2-online-experimentation)) and try it out there.

#### Data workflow
{: #s5-data-workflow}

Last but not least, besides the business and technical details of using the Datacore APIs, it is very important for applications consuming the Datacore API to adapt the way they work with data - their data and work flow - correspondingly. And especially so when they not only reuse but share and collaborate on data.

In short, the way to do it is to

- [design a **collaborative** data workflow](https://github.com/SICTIAM/ozwillo-datacore/wiki/Developing-apps-that-collaborate-on-data) that not only minds other users but also other apps,
- [design the rights policy](https://github.com/SICTIAM/ozwillo-datacore/wiki/Permissions,-rights-and-governance) that goes with it and [setting it up](https://github.com/SICTIAM/ozwillo-datacore/wiki/Configuring-Projects-and-Default-Rights).

And when interacting with data always **"think Datacore"**, that is at all times having the Datacore's data in mind in addition to the application-local data :

- taking in to account the facts that current, local data might not be up-to-date or in sync with the Datacore's,
- and that there may be more data in the Datacore.