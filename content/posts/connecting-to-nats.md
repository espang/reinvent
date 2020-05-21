---
title: "Connecting to Nats"
date: 2020-05-15T18:47:38+01:00
draft: true
---
# Reimplement: Nats Client



- [ ] Connect to the server
- [ ] Publish and Subscribe
- [ ] Benchmark & Test
- [ ] Retrospective



## Part1: Establish a connection

Nats is a message broker. It has a simple text based protocol and is open source. It is a good starting point to learn more about TCP connections. 

The NATS documentation describes all commands we need to care about [here](https://docs.nats.io/nats-protocol/nats-protocol#protocol-messages) in some detail. Using the java-client library [nats-java](https://github.com/nats-io/nats.java) officially supported by Synadia as a reference. This blog describes a limited implementation of:


```clojure
(import '[io.nats.client Nats])

(def connection (Nats/connect))
```

What happens here. We create a tcp connection to a server. The server will send us a (INFO)[https://docs.nats.io/nats-protocol/nats-protocol#info] message on connection. This info message contains information about the nats-server. It is also used to discover more nats-nodes that could be used when one of the nats-servers is not available. The first implementation will be limited to handle the default options. 

After receiving this message the java-client sends a (CONNECT)[https://docs.nats.io/nats-protocol/nats-protocol#connect] followed by a (PING)[https://docs.nats.io/nats-protocol/nats-protocol#pingpong] message. The server will respond with a `PONG` to every `PING` it gets. Equally should our client respond with `PONG` in case we receive a `PING`. The nats-server logsmost of this when started with the flags `-DV`. 

```
[4210] 2020/05/15 19:05:10.505979 [DBG] 127.0.0.1:45690 - cid:2 - Client connection created
[4210] 2020/05/15 19:05:10.508479 [TRC] 127.0.0.1:45690 - cid:2 - <<- [CONNECT {"lang":"java","version":"2.6.6","protocol":1,"verbose":false,"pedantic":false,"tls_required":false,"echo":true}]
[4210] 2020/05/15 19:05:10.508510 [TRC] 127.0.0.1:45690 - cid:2 - <<- [PING]
[4210] 2020/05/15 19:05:10.508524 [TRC] 127.0.0.1:45690 - cid:2 - ->> [PONG]
```

I will use (aleph)[https://github.com/ztellman/aleph] for TCP connections for now. It uses netty under the hood and exposes manifold streams. I have some experience with GO and I think the streams are similar to channels in GO (or clojure.core.async). It is the first time for me using both aleph and manifold, but I think it simplifies the connection handling for now. I intend to remove some of the libraries over time to get a better understanding. Be it by using java's networking libraries or netty directly. 

Let's start to write my client. 

```clojure
(:require '[aleph.tcp :as tcp]
          '[clojure.core.async :as async]
          '[manifold.stream :as s])

(defn receiver [client]
  (async/go-loop []
    (println (slurp @(s/take! @client)))
    (recur)))

(defn connect-to-nats [host port]
  (let [client (tcp/client {:host host
                            :port port})]
    (receiver client)
    client))

(def client (connect-to-nats "localhost" 4222))
```

This code is connecting to a nats-server running on the local host on port 4222. It also starts a background task in which it prints all messages received from NATS. Executing this code shows that we receive following messages from the nats-server:

```
INFO ... \r\n
PING\r\n
PING\r\n
-ERR 'Stale Connection'\r\n
```

The server sends two `PING` commands and then disconnects the client. The client has to act on the error. 

> Some protocol errors result in the server closing the connection. Upon receiving these errors, the connection is no longer valid and the client should clean up relevant resources.
 



