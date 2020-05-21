---
title: "NATS"
date: 2020-05-21T11:31:25+01:00
draft: false
---
# Reimplement NATS

This project is about implementing a simple [NATS](https://nats.io/) client and server. The first version from Derek Collision written in ruby on a weekend was handling around 150k msg/s. I aim to achieve the same - but not in a weekend. I will use Clojure for the project and learn netty to implement the transport layer. The code is available on my github page [here](https://github.com/espang/nats). 

### Client

Implement a limited NATS client. The client should be able to

* create a connection to a nats server via TCP,
* parse the protocol and
* offer an API to use publish and subscribe.

### Server

Implement a limited NATS server. The server should be able to

* accept connections from any number of clients,
* parse the protocol and
* handle subscriptions and forward published messages to all subscribers.

### Benchmark

Implement a benchmark for the server and the client. Client and Server should be "interchangeable" with the actual implementations.
