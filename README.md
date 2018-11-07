# Beaker Browser Specs

Every specification has its own file. To discuss the content of a spec, open issues and PRs on this repo.

## Getting started

Looking to learn about how Beaker works? Start here:

 - **User data and identity**
   - [Beaker&nbsp;user&nbsp;identities](./beaker-identities.md). APIs and UI flows for user identities.
   - [Beaker&nbsp;user&nbsp;filesystem](./beaker-user-fs.md). The filesystem for users' personal information.
 - **Filesystem APIs**
   - [Object-store&nbsp;folders](./object-store-folder.md). Managed folders which help applications share data.
 - **Manifest files and metadata**
   - [dat.json](https://github.com/datprotocol/dat.json). The dat.json standard manifest file. Used to describe a dat.
   - [Dat types](./dat-types.md). Standard dat "type" values and their effects in Beaker.

## Directory

### Phase 0 - Proposals Under Consideration

|Spec|Description|Last&nbsp;Updated|
|-|-|-|
|`unwalled.garden`|Not yet written. A collection of schemas used by Beaker.||
|`DatArchive` API|Not yet written. Refer to [the documentation](https://beakerbrowser.com/docs/apis/dat) for now.||
|`PeerSockets` API|Not yet written. Refer to [the PR](https://github.com/beakerbrowser/beaker-core/pull/6) for now.||
|`DatPubkeyFile` API|Not yet written. Refer to [this gist](https://gist.github.com/pfrazee/e4a9d1bdd095564991b5b75a5fe49bd7) for now.||
|[`Dat Service` API](./dat-service.md).|Proposed.||

### Phase 1 - Accepted Drafts

|Spec|Description|Last&nbsp;Updated|
|-|-|-|
|[Beaker&nbsp;user&nbsp;filesystem](./beaker-user-fs.md)|The filesystem for users' personal information.|Nov 6, 2018|
|[Beaker&nbsp;user&nbsp;identities](./beaker-identities.md)|APIs and UI flows for user identities.|Nov 7, 2018|
|[Dat types](./dat-types.md)|Standard dat "type" values and their effects in Beaker.|Nov 6, 2018|
|[Object-store&nbsp;folders](./object-store-folder.md)|Managed folders which help applications share data.|Nov 6, 2018|

### Phase 2 - Stable

|Spec|Description|Last&nbsp;Updated|
|-|-|-|
|[dat.json](https://github.com/datprotocol/dat.json)|The dat.json standard manifest file. Used to describe a dat.|April 20, 2018|

## Status badges

![Not written](https://img.shields.io/badge/Draft-Not%20written-red.svg)
![Draft](https://img.shields.io/badge/Draft-In%20progress-yellow.svg)
![Stable](https://img.shields.io/badge/Draft-Stable-green.svg)
![Deprecated](https://img.shields.io/badge/Draft-Deprecated-lightgrey.svg)

![Not implemented](https://img.shields.io/badge/Status-Not%20implemented-red.svg)
![Implemented](https://img.shields.io/badge/Status-Implemented-green.svg)

## Deprecated

 - [index.json](./index-json.md)
 - [Record Protocols](https://github.com/beakerbrowser/record-protocols-spec)
