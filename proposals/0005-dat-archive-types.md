# Dat archive types

In Dat, an archive is a collection of files and directories, and when used in the context of a Web browser, maps very cleanly to the concept of a website. In Beaker, Dat archives are used to represent websites. Also, for what it's worth, we often call a Dat archive a "dat".

In this proposal, we introduce the concept of "archive types." Types are a way to specify the role, contents, and/or data formats of a given Dat archive.

The archive type is indicated with a new `type` field to the dat.json, which can have a string or array of strings which indicate the type. Here's an example:

```js
{
  title: "Bob",
  type: ["user"]
}
```

An archive can have multiple types. The meaning of the types is conventional and should be determined by applications, though we will document some uses in this proposal.

## Reading and modifying

The `type` may be set as parameters in `DatArchive.create()`, `DatArchive.fork()`, or `archive.configure()`.

```js
var bob = DatArchive.create({
  title: 'Bob',
  type: ['user']
})
var alice = DatArchive.fork('dat://alice.com', {
  title: 'My Alice Fork',
  type: ['user', 'user-fork']
})
await bob.configure({type: ['user', 'my-user-type']})
```

The `type` may be read from `archive.getInfo()`:

```js
console.log(await bob.getInfo())
/* => {
  title: 'Bob',
  type: ['user', 'my-user-type'],
  ...
}
```

## Permissions

TODO when can archive types be changed?

## Proposed types

### Type: 'user'

Archives with the `'user'` type (aka "user dats") are meant to represent users in applications. They often contain profile information and some amount of content published by the user. We can think of a user dat's URL as the user's ID.

User dats are represented specially by the browser. They are used in user-session UIs as login targets (see [0004-simple-user-sessions](./0004-simple-user-sessions.md)) and represented in builtin UIs as the user's identities.

### Type: 'application'

Archives with the `'application'` type (aka "application dats") are meant to represent applications.

Applications often contain HTML, CSS, and Javascript files. They may in future proposals be given special capabilities and limitations, such as:

 - The ability to be "installed" and given special privileges and behaviors, such as registering for handling "intents" or having buttons added to the browser UI
 - A restriction against modifications via the `DatArchive` API, to protect against malicious edits
