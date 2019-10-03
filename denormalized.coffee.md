    CouchDB = require 'most-couchdb/with-update'

    class DenormalizedDB extends CouchDB

      denormalized: ->
        @changes include_docs:true
        .filter ({doc}) -> doc?._id?
        .map ({doc}) -> doc
        .skipRepeatsWith (a,b) -> a._id is b._id and a._rev is b._rev
        .filter @filter
        .multicast()

      all_docs: ->
        for await doc from @findAsyncIterable selector: _id: $gt: null
          if doc?._id? and @filter doc
            yield doc

      filter: ({_id}) -> _id[0] isnt '_'

    module.exports = DenormalizedDB
