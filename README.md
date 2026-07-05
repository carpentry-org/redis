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
