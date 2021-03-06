Aleph exposes data from the network as a [Manifold](https;//github.com/ztellman/manifold) stream, which can easily be transformed into a `java.io.InputStream`, [core.async](https://github.com/clojure/core.async) channel, Clojure sequence, or [many other byte representations](https://github.com/ztellman/byte-streams).  It exposes simple default wrappers for HTTP, TCP, and UDP, but allows access to full performance and flexibility of the underlying [Netty](https://github.com/netty/netty) library.

```clj
[aleph "0.4.0-alpha7"]
```

### HTTP

Aleph follows the [Ring](https://github.com/ring-clojure) spec fully, but also allows for the handler function to return a [Manifold deferred](https;//github.com/ztellman/manifold) to represent an eventual response.  This feature may not play nicely with Ring middleware which modifies the response, but this can be easily fixed by reimplementing the middleware using Manifold's [let-flow](https://github.com/ztellman/manifold/blob/master/docs/deferred.md#let-flow) operator.

```clj
(require '[aleph.http :as http])

(defn handler [req]
  {:status 200
   :headers {"content-type" "text/plain"}}
   :body "hello!")

(http/start-server handler {:port 8080})
```

For HTTP client requests, Aleph models itself after [clj-http](https://github.com/dakrone/clj-http), except that every request immediately returns a Manifold deferred representing the response.

```clj
(require
  '[manifold.deferred :as d]
  '[byte-streams :as bs])

(-> @(http/get "https://google.com/")
  :body
  bs/to-string
  prn)

(d/chain (http/get "https://google.com")
  :body
  bs/to-string
  prn)
```

### WebSockets

On any HTTP request which has the proper `Upgrade` headers, you may call `(aleph.http/websocket-connection req)`, which returns a deferred which yields a **duplex stream**.  Messages from the client can be received via `take!`, and sent to the client via `put!`.  An echo WebSocket handler, then, would just consist of:

```clj
(require '[manifold.stream :as s])

(defn echo-handler [req]
  (let [s @(http/websocket-connection req)]
    (s/connect s s))))
```

This takes all messages from the client, and feeds them back into the duplex socket, returning them to the client.  WebSocket text messages will be emitted as strings, and binary messages as byte arrays.

WebSocket clients can be created via `(aleph.http/websocket-client url)`, which returns a deferred which yields a duplex stream that can send and receive messages from the server.

### TCP

A TCP server is similar to an HTTP server, except that for each connection the handler takes two arguments: a duplex stream and a map containing information about the client.  The stream will emit Netty `ByteBuf` objects, which can be coerced into other byte representations using the [byte-streams](https://github.com/ztellman/byte-streams) library.  The stream will accept any messages which can be coerced into a binary representation.

An echo TCP server is very similar to the above WebSocket example:

```clj
(require '[aleph.tcp :as tcp])

(defn echo-handler [s info]
  (s/connect s s))

(tcp/start-server echo-handler {:port 10001})
```

A TCP client can be created via `(aleph.http/tcp-client {:host "example.com", :port 10001})`, which returns a deferred which yields a duplex stream.

### UDP

A UDP socket can be generated using `(aleph.udp/socket {:port 10001, :broadcast? false})`.  If the `:port` is specified, it will yield a duplex socket which can be used to send and receive messages, which are structured as maps with the following data:

```clj
{:host "example.com"
 :port 10001
 :message ...}
```

Where incoming packets will have a `:message` that is a Netty `ByteBuf` that can be coerced using `byte-streams`, and outgoing packets can be any data which can be coerced to a binary representation.  If no `:port` is specified, the socket can only be used to send messages.

### license

Copyright © 2014 Zachary Tellman

Distributed under the MIT License
