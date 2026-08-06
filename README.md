# redis

A Redis client library for Carp, supporting Redis 7.x.

## Installation

```clojure
(load "https://git.veitheller.de/carpentry/redis.git@0.2.0")
```

## Usage

```clojure
(load "redis.carp")

(defn main []
  (match (Redis.open "127.0.0.1")
    (Result.Success r)
      (do
        (println* &(Redis.ping &r))
        (println* &(Redis.set &r @"key" @"value"))
        (println* &(Redis.get &r @"key"))

        ; list operations return nested RESP arrays
        (println* &(Redis.rpush &r @"mylist" @"a"))
        (println* &(Redis.rpush &r @"mylist" @"b"))
        (println* &(Redis.lrange &r @"mylist" @"0" @"-1"))

        ; pattern match on responses
        (match (Redis.get &r @"key")
          (Result.Success resp)
            (match resp
              (RESP.Str s) (println* "got: " &s)
              (RESP.Null) (println* "key not found")
              _ (println* "unexpected type"))
          (Result.Error e) (println* "error: " &e))

        (Redis.close r))
    (Result.Error err) (IO.errorln &err)))
```

The library provides thin wrappers around all standard Redis commands through
7.2, so you can call them directly (e.g. `Redis.get`, `Redis.hset`,
`Redis.lrange`). For commands with complex or variadic arguments, use
`Redis.send` directly:

```clojure
(Redis.send &r @"SET" &[(to-redis @"key") (to-redis @"value")])
(Redis.read &r)
```

The RESP type supports recursive arrays via `Box`, so nested structures
(e.g. from `XRANGE` or `COMMAND`) are decoded faithfully.

### Transactions

`with-transaction` wraps commands in `MULTI`/`EXEC` and sends the whole thing in
one round trip. It returns a `TransactionResult`, because a transaction has
three outcomes a caller has to tell apart:

```clojure
(match (with-transaction &r
         (incr @"visits")
         (hset @"user:1" @"seen" @"now"))
  (Result.Success (TransactionResult.Executed replies))
    (println* "ran " (Array.length &replies) " commands")
  (Result.Success (TransactionResult.Aborted))
    (println* "a watched key changed, nothing ran")
  (Result.Success (TransactionResult.QueueFailed errs))
    (println* "rejected before EXEC: "
              (QueueError.message (Array.unsafe-nth &errs 0)))
  (Result.Error e) (IO.errorln &e))
```

`Executed` carries one reply per command, in order. Redis does not roll back a
command that fails at runtime, so a `WRONGTYPE` shows up as a `RESP.Err` element
inside `Executed` rather than aborting anything.

`QueueFailed` is the other error: the server refused a command outright — an
unknown name, the wrong number of arguments — and ran none of them. Each
`QueueError` says which command by index and what the server said.

#### Optimistic locking

`Aborted` is what makes `WATCH` useful. Watch the keys you are about to read,
read them, then run the transaction: if anything else touched a watched key in
between, Redis aborts and you try again with fresh values.

```clojure
(defn transfer [r from to amount]
  (let-do [done false]
    (while-do (not done)
      (ignore (Redis.watch r @from))
      (match (Redis.get r @from)
        (Result.Success (RESP.Str balance))
          (if (< (Maybe.from (Int.from-string &balance) 0) amount)
            (do (ignore (Redis.unwatch r)) (set! done true))
            (match (with-transaction r
                     (decrby @from (Int.str amount))
                     (incrby @to (Int.str amount)))
              (Result.Success (TransactionResult.Aborted)) ()
              _ (set! done true)))
        _ (do (ignore (Redis.unwatch r)) (set! done true))))))
```

Only `Aborted` retries — an `Error` or a `QueueFailed` will not fix itself by
looping. For an explicit accumulator instead of the macro, build a
`RedisPipeline` and hand it to `Redis.transaction-exec`.

### Pub/sub

After subscribing you can receive the stream of pushed messages with
`Redis.next-message`, which decodes each reply into a typed `PubSubMessage`:

```clojure
(match (Redis.subscribe &r @"news")
  (Result.Success _)
    (match (Redis.next-message &r)
      (Result.Success msg)
        (match msg
          (PubSubMessage.Message channel payload)
            (println* "got " &payload " on " &channel)
          (PubSubMessage.Subscribe channel count)
            (println* "subscribed to " &channel)
          _ (println* "other pub/sub reply"))
      (Result.Error e) (IO.errorln &e))
  (Result.Error e) (IO.errorln &e))
```

`PubSubMessage` covers `Message`, `PMessage` (pattern subscriptions), and the
`Subscribe`/`Unsubscribe`/`PSubscribe`/`PUnsubscribe` confirmations. To keep
processing messages in a loop, hand `Redis.listen` a callback — it runs until
the connection errors out.

Full API documentation lives [here](https://veitheller.de/redis).

More examples are in the [`examples`](./examples) directory.

<hr/>

Have fun!
