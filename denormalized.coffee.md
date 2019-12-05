    CouchDB = require 'most-couchdb/with-update'

    class DenormalizedDB extends CouchDB

      denormalized: ->
        for await {doc} from @changesAsyncIterable include_docs:true
          if doc?._id? and @filter doc
            yield doc

      all_docs: ->
        for await doc from @findAsyncIterable selector: _id: $gt: null
          if doc?._id? and @filter doc
            yield doc

      filter: ({_id}) -> _id[0] isnt '_'

    module.exports = DenormalizedDB
