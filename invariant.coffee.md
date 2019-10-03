    sleep = (timeout) -> new Promise (resolve) -> setTimeout resolve, timeout
    assert = require 'assert'
    most = require 'most'

    invariants = new Map

    invariant = (comment,db,handler) ->
      unless db? and handler?
        console.error "✘ Invariant `#{comment}` is not specified"
        return

      if invariants.has db
        handlers = invariants.get db
      else
        handlers = []
        invariants.set db, handlers

      handlers.push {comment,handler}
      console.error "✔ Invariant `#{comment}` is ready"
      return

    heal = (t,p) ->
      try
        await p
      catch error
        console.warn t, '→', error.message,
          error.response?.error?.method, error.response?.error?.path # superagent's

    running = true

    keep = (fun) ->
      while true
        await heal "Restarting", fun()
        return if not running
        await sleep 5000
      return

    {INVARIANTS_LOG} = process.env

    invariant.start = ->

      for [db,handlers] from invariants
        do (db,handlers) ->

          observer = (doc) ->
            if INVARIANTS_LOG is 'yes'
              console.log "Handling #{doc._id} #{doc._rev} (start)"
              _s = Date.now()
            s = most.of(doc).multicast()
            for {comment,handler} in handlers
              try await s.thru(handler).map( (p) -> heal comment, p ).awaitPromises().drain()
            await sleep 10 # give the other DBs a chance to run
            if INVARIANTS_LOG is 'yes'
              console.log "Handling #{doc._id} #{doc._rev} took #{Date.now()-_s}ms"
            return

Maintain invariants on all changes.

          keep ->
            db.denormalized()
            .map observer
            .awaitPromises() # queue
            .drain()

Also maintain invariants on existing database content.

          keep ->
            for await doc from db.all_docs()
              try await observer doc

      return

    invariant.stop = ->
      running = false

    module.exports = invariant
