    chai = require 'chai'
    chai.should()
    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout

    describe 'The module', ->
      it 'should load denormalized', ->
        require '../denormalized'
      it 'should load invariant', ->
        require '../invariant'
      it 'should load the module', ->
        require '..'

    describe 'Invariants', ->
      uri = "http://#{process.env.COUCHDB_USER ? 'admin'}:#{process.env.COUCHDB_PASSWORD ? 'password'}@couchdb:5984/test"
      CouchDB = require '../denormalized'

      db = new CouchDB uri

      before ->
        await db.create()
      after ->
        await db.destroy()

      {invariant,DenormalizedDB} = require '..'

      it 'should run the example in the README', ->

        panel = 'http://example.com'

        class Business
          @SUBSCRIPTION: 'subscription'
          @Subscription: ({type}) -> type is Business.SUBSCRIPTION

          constructor: (db_uri) ->
            @db = new DenormalizedDB db_uri

            invariant 'A subscription document should have a panel link', @db, (S) =>
              S
              .filter Business.Subscription
              .map (doc) =>
                url = "#{panel}/s/#{doc.s}"

                return if doc.panel?.url is url
                doc.panel ?= {}
                doc.panel.url = url
                await @db.put doc
                return

            invariant.start()
            return

        uut = new Business uri

        await db.put
          _id: 'S:1'
          type: Business.SUBSCRIPTION
          s: 1

        await sleep 1000
        invariant.stop()

        doc = await db.get 'S:1'

        doc.should.have.property 'panel'
        doc.panel.should.have.property 'url', 'http://example.com/s/1'
