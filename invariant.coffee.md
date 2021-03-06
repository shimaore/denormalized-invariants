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

    heal = (t,p,doc) ->
      try
        await p
      catch error
        console.warn t, '→', error.message,
          error.response?.error?.method, error.response?.error?.path, # superagent's
          doc?._id, doc?._rev

    running = true

    keep = (fun) ->
      while true
        await heal "Restarting", fun()
        return if not running
        await sleep 5000
      return

    {INVARIANTS_LOG} = process.env

    NS_PER_MSEC = `1_000_000n`

    class TimeoutError extends Error
      constructor: ->
        super 'Invariant took too long'

    class Timer
      constructor: (timeout)->
        @timeout = (BigInt timeout) * NS_PER_MSEC
        @start = process.hrtime.bigint()

      cancel: ->
        now = process.hrtime.bigint()
        return false if (now - @start) < @timeout
        throw new TimeoutError

    invariant.start = ->

      for [db,handlers] from invariants
        do (db,handlers) ->

          observer = (doc,label) ->
            if INVARIANTS_LOG is 'yes'
              console.log "Handling #{label} #{doc._id} #{doc._rev} (start)"
              _s = process.hrtime.bigint()
            s = most.of(doc).multicast()
            for {comment,handler} in handlers
              timer = new Timer 20000
              cancel = -> timer.cancel()
              try await handler(s,cancel).map( (p) -> heal comment, p, doc ).awaitPromises().drain()
            await sleep 10 # give the other DBs a chance to run
            if INVARIANTS_LOG is 'yes'
              _s = process.hrtime.bigint() - _s
              console.log "Handling #{label} #{doc._id} #{doc._rev} took #{_s/NS_PER_MSEC}ms"
            return

Maintain invariants on all changes.

          keep ->
            for await doc from db.denormalized()
              try
                await observer doc, 'changes'
              catch error
                console.error error
            return

Also maintain invariants on existing database content.

          keep ->
            for await doc from db.all_docs()
              try
                await observer doc, 'alldocs'
              catch error
                console.error error
            return

      return

    invariant.stop = ->
      running = false

    module.exports = invariant
