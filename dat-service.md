# DatService

This specification defines a dat service API which is another kind of "beaker site". If traditional dat site is analogus to [hyperdrive][] in dat terms, then dat service would be analogus to [hypercore][] as it provides alternative API for the underlying data in dat. 


## Specification

The DatService API provides an optional backend for beaker sites. Opt-in site is given ability to provide alternative API for underlaying content.

### DatService

Site can opt-in into service mode by providing a `service` key in [`dat.json`][] mapped to a relative path to a service implementation:

```json
{
  "title": "My web service",
  "description": "A simple web service built with the Beaker Browser",
  "type": ["webservice"],
  "links": {
    "license": [{
      "href": "http://creativecommons.org/licenses/by-nc/2.5/",
      "title": "CC BY-NC 2.5"
    }]
  },
  "fallback_page": "/404.html",

  "service": "./web_service.js",
  "service_root": "/api/"
}
```

Optionally `service_root` field can be provided, in which case only URLs under that path would be handled by a service. If omitted all requests (except for service implementation itself) will be handled by a service.


### DatService API

DatService implementation is just a JS module with `default` export (optionally async) function. It will be invoked with a [`Request`][] instance and it **MUST** return corresponding [`Response`][] instance. (Returning anything else is considered service error and results in 5XX error).

```js
export default async function (request) {
  const [_, id, ...path] = request.url.pathname.split("/")
  const archive = await DatArchive.load(id)
  const content = await archive.readFile(path.join("/"))
  return new Response(content, {
    status: 200,
    headers: {
        "Content-Security-Policy": "default-src 'self'"
    }
  })
}
```

DatService is loaded in an isolated worker context which maybe be activated or terminated by beaker arbirarily based on unspecified constraints. In fact multiple service instances maybe activeted in parallel for better througput. That is to say no assumbtions can be made in regards to state outside of the function body.


#### IPC Interface

Above example illustrated a REST interface for the service, but that is only useful in cases where operations have both inputs and outputs. That being said each request can be treated as a concurrent task with an IPC mechanims instead:

##### Service (Worker)
```js
export default async function(inbox) {
  const archive = await DatArchive.load(SOCIAL_FEED)
  
  for await (const post of inbox.body) {
    await archive.writeFile(`posts/${Time.now()}.json`, message)
  }
}
```

##### Service Client (UI thread or worker)
```js
// An app using social feed.
class LibSocialClient {
  constructor() {
    this.encoder = new TextEncoder()
    this.stdin = await fetch(SERVICE_URL, {
      body: new ReadableSream({
        start: (controller) => {
          this.stdout = controller
        }
      })
    })
  }
  encodeJSON(json) {
    return this.encoder.encode(JSON.stingify(message))
  }
  post(message) {
    const data = this.encodeJSON(message)
    this.stdout.enqueue(data)
  }
}

```

> #### Concern with IPC over Rest
> 
> It would be very unfortunate if it will be impossible to transfer transferable objects like [ArrayBuffer][], [MessagePort][] to the service without copying. That being said that's a limitation that [ReadableStream][]s would have to address.

#### Service Context

Dat service could refer to itself via `self` global binding representing the scope it was loaded into. It is simlar to [`WorkerGlobalScope`] in that it:

Provides access to the following properties:

- [`location`](https://developer.mozilla.org/en-US/docs/Web/API/WorkerGlobalScope/location) (Corresponds to the URL of the service)
- [`performance`](https://developer.mozilla.org/en-US/docs/Web/API/WorkerGlobalScope/performance)
- [`console`](https://developer.mozilla.org/en-US/docs/Web/API/WorkerGlobalScope/console)

Provides access to the following functions:

  - [`close`](https://developer.mozilla.org/en-US/docs/Web/API/WorkerGlobalScope/close) (terminates this service instance)
  - [`atob`](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/atob)
  - [`btoa`](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/btoa)
  - [`setTimeout`](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/setTimeout)
  - [`clearTimeout`](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/clearTimeout)
  - [`setInterval`](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/setInterval)
  - [`clearInterval`](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/clearInterval)
  - [`fetch`](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/fetch)
  - [`createImageBitmap`](https://developer.mozilla.org/en-US/docs/Web/API/WindowOrWorkerGlobalScope/createImageBitmap)


In addition service will be given access to other [Beaker APIs][]. APIs w require opt-in via [`dat.json`][] will need to opt-in to have access to those APIs.

> Note: It might be better if services had to opt-in into all of the IO operations (fetch, Beaker APIs) that would make them viable for contract style applications e.g. enforced rules of participation.


#### Service Updates

There is no special mechanism for managing service updates. That is because all the building blocks are already there to allow graceful update handling. Specifically service can listen to own changes and call `self.close()` when that happens, that would effectiely cause next request to be handled by a new service instance. Alternatively service could choose different stategy to handle update.

> Question: Why expose `.close()` in first place?
> 
> It may appear that lifetime of the service matches lifetime of the request, but that's not the case. Service is very likely to outlive several requests. It just designed to prevent you from assuming it will or that single instance will be active at a time. This allows scheduler to optimize resource usage by killing off services that are no longer in use or runing multiple instances concurrently in case of high demand.

#### Let it crash

Dat Service adopts [Let it Crash][] phylosophy - Any unhandled exception or failed promise would cause worker to be terminated and all the pending requests will result in `500` - "Internal Server Error". Subsequent request will be handled by newly spawn worker instance.

In practice most exceptions could be handled gracefully by enclosing async function body in `try / catch` bloc, but given that errors could escape through missing `await` or timer callback such services will be terminated.


### Use cases

Beaker sites are basically what static sites are on conventional web. They are still able of providing collaborative experiences by publishing & subscribing to user generated content via [DatArchive][] API but that poses certain limitations that this API attempts to address:


#### Decoupling data storage from presenation

Same underlaying data could be consumed & presented in many different ways. In some cases it's more effective to compute presentation from some data set than is to store computed presentation.

While technically it is possible to do today it is only at the application layer. Meaning same data can't be requested in different format by another application, neither it is possible to do it from the browser itself.

Dat Service API would enable this simply by taking `Content-Type` header of the requset into account e.g. it could server same unherlying content as markdown, html, or structured JSON.

> Note: Term "presentation" here is used to describe mapping between data layout at the storage to a format being consumed.
>  
> In these terms [HyperDrive][] and [HyperDB][] are both presentations of [HyperCore][].
> 
> Another example would be a fritter post, which currently is stored as JSON, but it does have an HTML presentation and Markdown presentation. It would be nice if provider could choose best format for storing independent of best format for consuming same data, and it's not unreasonable to imagine different cases preferring different format.
> 
> It is worth pointing out that presentation can be 1:n relationship e.g. fritter thread can have JSON & HTML presentation even though undrlaying data may span across multiple file and machines.

#### Hooking into browser pipeline

Today there is no way to hook into browser pipeline meaning there is no way to dynamically generate / compute requested content.

Service API would be able to address that would unlock some interesting opportunities:

- Semi-atomatic dependency management. For example if application depends on JS modules from other Dat's service could on demand obtain them and write to local cache.
- Code transpilation. Huge amount of code base is authored in commonjs format, but with dat service it becomes visible to have service that transpiles that into ES module format.



#### Composable ecosystem

At the moment dat sites are not really composable, they can't provide an API nor are able to interact with each other. At best they can import JS modules from each other but that reqires some trust in that imported module won't corrupt data or cause some other harm.

Services address this as they are free to provide whatever API they choose. In fact if someone writes module transiplation service everyone in the ecosystem will be able to use it. Parties will in fact be able to choose amount of coordination on their own merits by either interating with pinned version of the service or the lastest.

It's not unreasonable to expect service pipelines either where service A consumes service B which under the hood might be consuming service C etc. 

#### Data Integrity

At the moment if multiple sites operate on the same dataset there is a chance it could be corrupted that is unless all parties are fully compatible in regards to data format. In practice full compatibility is diffcult to achieve and requries a lot of coordination. There is an attempt to address data corruption via data schemas but that still requires coordination between apps especially so when migrating to different underlying data model.

Services can address this simply by providing a high level REST API. For instance identity service could provide API for reading & updating user name. It is also free to change data model from JSON to binary to CRDTs as long as provided API is backwards compatible. In fact when breaking APIs are needed separate migartion service could be developed to provide an adapter for legacy users.


##### Batch Operations

In some cases there are relations with-in datasets that spans across multiple files. Updates to such data requires changes across multiple files and without atomic operations it's prone to errors.

In recent years [GraphQL][] & alike also become popular which in nutshell provide batched read operations.

Dat services would be able to address these use cases by providing an API for atomic operations that are handled in one batch without communication overhead.


#### Legacy Code 

There is a lot cool open source projects availabel in the wild e.g [hypothesis client][] that can't simply be adapted to beaker ecosystsem as they assume some backend with defined REST API.

Services would enable adapting those by emulating REST APIs of the backend they are able to work with. 

### Future features

#### Non-hyperdrive mounts

Dat's primary data structure, Hypercore, is a log structure which can be repurposed for many different cases. The primary use-case is Hyperdrive, the filesystem data structure which drives the Beaker FS and websites. Other feasible data structures might be a key-value store, a video/audio feed, or even a log of data. This is especially useful for implementing CRDTs.

It would make sense to provide hypercore API to the DatService and let it provide an alterantive API on top. In fact existing sites could in theory be just sites that use default hyperdrive service.


#### Hashbase support

In the future hashbase.io could adopt a dat service support and by consequnce allow dat applications to be staly fully functional even through https gateway. It would require athentification & some permission workflows but could potentially enable whole range of interesting use cases:

- Public Inbox - Service could for example allow sending message into managed inbox to users who authentificated through OAuth.

#### Wasm

In the future it would make sense to allow authoring services in Wasm for more more computation heavy applications. 


#### What do we lose by choosing IPC API over REST

- REST API provides addressing. Any computed data that could be:
    - Subset of what's in the underlaying file.
    - Can be a join of datasets across files.
    - Can be content that is lazily obtained, via computation or IO.
    - Can be subscribtion like a swarm of messages or feed of messages on fritter.

- REST assigned URLs make it possible for all that data to be usable by the rest of the browser pipeline:
    - One can navigate to the URL and look at the data (view).
    - Conference call video/audio stream.
    - HTML like microformats can be computed out of raw dataset on the fly for integration with existing systems like IndieWeb. 


- Drop in replacement for existing server stack.
    - If you have node server for a thing chances are you'd be able to convert it toa service without even changing client code.
    - If you want to bridge with existing client with configurable backend you'de be able to do so without forking a client.
- REST is RPC ready.

  Doing RPC over IPC requires managing references to pair input with output and that raises whole range of questions including GC semantics. REST as primive already addresses that and putting IPC over just implies long lived request.
  
  
- Remote Dat Service

  Choosing REST API provides an opportunity to run Dat Services on the server such that legacy browsers could use them via same interface. Exposing some custom API for background tasks would make that far more difficult.
  
  
#### Risks / Concerns / Considerations

- **Dynamic content**

  If Beaker adobts services architecture it may alter an ecosystem. Supporting this on hashbase.io may become non-trivial requirement.
  
  Services would most definetly make supporting such Dat apps more difficult. That being said any kind of IPC or other computation primitive would as well & whether that is a good tradeoff is worth considering.

  It is also worth pointing out that non static sites like [Fritter][] (or anything that writes content) is not really functional over hashbase.io
  
- **Service as enhancement vs replacement**

  We need ensure that design of the services encourages them to be used for enhancement rather than replacement for static sites.
  
  Ideally services will be used to serve alternate presentations of the data is still consumabre without a service e.g. HTML version of markdown / JSON file. Another example would be service could provide data query interface to map / reduce data but the data being transform should be still available.
  
  Ideally browser should be able to load the service in the local worker without having say having hashbase run it on the server. 

[`dat.json`]:https://beakerbrowser.com/docs/apis/manifest
[DatArchive]:https://beakerbrowser.com/docs/apis/dat
[`Response`]:https://developer.mozilla.org/en-US/docs/Web/API/Response/Response
[`Request`]:https://developer.mozilla.org/en-US/docs/Web/API/Request
[`WorkerGlobalScope`]:https://developer.mozilla.org/en-US/docs/Web/API/WorkerGlobalScope
[Beaker APIs]:https://beakerbrowser.com/docs/apis/
[Let it crash]:http://wiki.c2.com/?LetItCrash
[hyperdrive]:https://github.com/mafintosh/hyperdrive
[hypercore]:https://github.com/mafintosh/hypercore
[GraphQL]:https://graphql.org/
[hypothesis client]:https://github.com/hypothesis/client
[ArrayBuffer]:https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Global_Objects/ArrayBuffer
[MessagePort]:https://developer.mozilla.org/en-US/docs/Web/API/MessagePort
[ReadableStream]:https://developer.mozilla.org/en-US/docs/Web/API/Streams_API/Using_readable_streams
[HyperDrive]:https://github.com/mafintosh/hyperdrive
[HyperDB]:https://github.com/mafintosh/hyperdb
[HyperCore]:https://github.com/mafintosh/hypercore
[Fritter]:http://github.com/beakerbrowser/fritter
