---
sidebar_position: 1
---

# Modeling Data

All data in Automerge must be stored in a document. A document can be modeled in a variety of ways, and there are many design patterns that can be used. An application could have many documents, typically identified by a UUID. 

In this section, we will discuss how to model data within a particular document, including how to version and manage data with Automerge in production scenarios.

## How many documents?

You can decide which things to group together as one Automerge document (more fine grained or more coarse grained) based on what makes sense in your app. Having hundreds of docs should be fine — we've built prototypes of that scale. One major automerge project, [PushPin](https://github.com/automerge/pushpin), was built around very granular documents. This had a lot of benefits, but the overhead of syncing many thousands of documents was high. One of the first challenges in synchronizing large numbers of documents is that nodes are likely to have overlapping but disjoint documents and neither side wants to disclose things the other doesn't know about (at least in our last system, knowing the ID of a document was evidence a client should have access to it.)  

We believe on the whole there's an art to the granularity of data that is universal. When should you have two JSON documents or two SQLite databases or two rows? We suspect that an Automerge document is best suited to being a unit of collaboration between two people or a small group. 


## TypeScript support

Given that you have a document, how can you create safety rails for it's data integrity? In a typical SQL database, a table would have it's own schema, and you create one-way migrations for application versions. Automerge is flexible on the schema, and will let you add and remove properties and values at will. To improve the programming experience, a Document can be typed to have it's own schema using TypeScript.

```js
type D = { 
  count: Automerge.Counter,
  text: Automerge.Text,
  cards: [] 
}
let doc = Automerge.change<D>(Automerge.init(), (doc: D) => {
  doc.count = new Automerge.Counter()
  doc.text = new Automerge.Text()
  doc.cards = []
})
```

## Merging two disjointed documents 

Few applications have multiple documents that need to be merged together at will. For example, in a Kanban-style app, if you wanted to merge two boards into one board, you can't do that unless the boards come from a common ancestor. To enable this use case, you can create a root change that two documents share. 

To create an initial root change for your Automerge document, you can make a change that has the same hash every time it is created. We do this by setting the actorId to a static string `'0000'` and setting the time to always be 0.

```js
let doc = Automerge.init('0000', {time: 0})
```

We then modify this document to create the initial values for all of the types in the document.

```js
type D = { 
  count: Automerge.Counter,
  text: Automerge.Text,
  cards: [] 
}
let schema = Automerge.change<D>(Automerge.init('0000', {time: 0}), (doc: D) => {
  doc.count = new Automerge.Counter()
  doc.text = new Automerge.Text()
  doc.cards = []
})
```

> NOTE: You only have to create this initial change the first time the document loads. You can check if you have a local document already before making this initial document.

Great, now we have an initial document that represents the base schema of our data model. But because we initialized this Automerge document with a fixed actorId, we need to create a new document that has a random id. We don't want all of our resulting changes to be with the actorId `0000`, because that should be reserved for the initial change only.

```js
let change = Automerge.getLastLocalChange(schema)
const [ doc , ]= Automerge.applyChanges(Automerge.init<D>(), [change])
```

The full resulting code is:

```js
function createFrozenAncestor()  {
  let schema = Automerge.change<D>(Automerge.init('0000', {time: 0}), (doc: D) => {
    doc.count = new Automerge.Counter()
    doc.text = new Automerge.Text()
    doc.cards = []
  })
  let change = Automerge.getLastLocalChange(schema)
  const [ doc , ]= Automerge.applyChanges(Automerge.init<D>(), [change])
  return doc
}
```

Now, `doc` is initialized and ready to be used as any other Automerge document. You can save that document to disk as you would normally with `Automerge.save(doc)` and load it later when your app starts.


## Versioning

Often, there comes a time in the production lifecycle where you will need to change the schema of a document. Because Automerge uses a JSON document model, it's similar to a NoSQL database, where properties can be arbitrarily removed and added at will. 

However, we strongly recommend versioning your documents. This allows you to detect older document versions and modify, upgrade, or migrate those documents on the fly. This also provides some enforcement of schema versions. All merged documents will have to be children of the initial document. If the initial document schema changes, you will need to migrate the older document to the new document. 

For example, if you change the `cards` property in the above example to an `Automerge.Table`, users who upgrade to the latest version of the application may see unexpected behavior. Below is an example of adding versions to your documents. 

```js
type D = { 
  version: 1,
  count: Automerge.Counter,
  text: Automerge.Text,
  cards: []
}

type E = { 
  version: 2,
  count: Automerge.Counter,
  text: Automerge.Text,
  cards: Automerge.Table
}

let doc1 = getDocumentFromNetwork()
let doc2 = localDocument()

if (doc1.version === 1) {
  doc1 = migrate(doc1)
}
```

This is heavily dependent upon your user experience requirements and your data model. In some cases, you may want to enforce that users are on the most up-to-date version of the application. In other cases, it is important that the application never prevents the user from editing the database on the local machine, even in the case where the application is out of date. However, this is overall an understudied area of CRDTs that needs more work -- hopefully from you! Ink & Switch wrote about this topic in the [Cambria](https://www.inkandswitch.com/cambria) paper, and there is still much left to do to make nicer primitives for developers to use, rather than handcrafting each migration.

## Performance

Automerge documents hold their entire change histories. It is fairly performant, and can handle a significant amount of data in a single document's history.  Performance depends very much on your workload, so we strongly suggest you do your own measurements with the type and quantity of data that you will have in your app. 

Some developers have proposed “garbage collecting” large documents. If a document gets to a certain size, a central authority could emit a message to each peer that it would like to reduce it in size and only save the history from a specific change (hash). Martin Kleppman did some experiments with a benchmark document to see how much space would be saved by discarding history, with and without preserving tombstones. See [this video at 55 minutes in](https://youtu.be/x7drE24geUw?t=3289). The savings are not all that great, which is why we haven't prioritised history truncation so far. 

Typically, performance improvements can come at the networking level. You can set up a single connection (between peers or client-server) and sync many docs over a single connection. The basic idea is to tag each message with the ID of the document it belongs to. There are possible ways of optimising this if necessary. In general, having fewer documents that a client must load over the network or into memory at any given time will reduce the syncronization and startup time for your application. 

