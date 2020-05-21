---
title: "NATS: Parser"
date: 2020-05-21T12:13:20+01:00
draft: false
---
# NATS: Parser

NATS uses a text based protocol on top of TCP. The protocol is described [here](https://docs.nats.io/nats-protocol/nats-protocol). The client will receive following operations:

* INFO
* PING
* PONG
* MSG
* +OK
* -ERR

some of them have additional data and all of them end with `\r\n`. `MSG` is slightly different as it contains two parts/ lines.

Netty creates a chain of ChannelHandlers on top of tcp connections. Incoming data is handled by ChannelInboundHandlers - an interface that also implements ChannelHandlers. The parser will be based on [ByteBuf](https://netty.io/4.0/api/io/netty/buffer/ByteBuf.html) the buffer that is used by those interfaces. Let's implement a function that given a ByteBuf returns the optional first message and mutates the used buffer. Mutating the buffer in this case means moving the readerIndex. The parser will start at the current readerIndex and checks whether it can parse a complete nats-message with the data readable. Readable data is data between the readerIndex and the writerIndex. Given a buffer with following state

```
 -------+----------------+-----
        |PING\r\nPONG\r\n|
 -------+----------------+-----
        |                |
    readerIndex      writerIndex
```

the parser should parse the ping message and move the readerIndex forward by 6 to the start of the pong message. Given a buffer with 


```
 -------+------+-----
        |PING\r|
 -------+------+-----
        |      |
  readerIndex  writerIndex
```

the parser should return no message and the buffer's readerIndex shouldn't be changed. 







```Clojure
(defn parse [^ByteBuf buf] {:msg/type :unknown})
```
