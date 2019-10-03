Maintain invariants
-------------------

The goal of this module is to help maintain invariants in a set of databases.
This is typically used to enforce business rules inside a database or between databases.

Specifically it is targeted at CouchDB.

```coffeescript
{invariant,DenormalizedDB} = require 'denormalized-invariants'

class Business
  Subscription: ({type}) -> type is 'subscription'

  constructor: (db_uri) ->
    @db = new DenormalizedDB db_uri

    invariant 'A subscription document should have a panel link', @db, (S) =>
      S
      .filter Business.Subscription
      .map (doc) =>
        url = "#{panel}#/s/#{doc.s}"

        return if doc.panel?.url is url
        doc.panel ?= {}
        doc.panel.url = url
        await @db.put doc
        return

    invariant.start()
    return
```

These are called invariants because the goal of the operative part of the invariant is to attempt to maintain its textual description.
