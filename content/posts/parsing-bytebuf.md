---
title: "Parsing Bytebuf"
date: 2020-05-17T19:36:26+01:00
draft: true
---
# Parsing the messages from the NATS Server

Implement a static function `Message parseMessage(ByteBuf buf)`. `ByteBuf` is from netty. The library I use for the TCP connection. `Message` is an abstract class used for the NATS messages. The function returns the first message on the buffer or Nothing. ByteBuf has a readerIndex and a writerIndex keeping track of the last written and last read position. The function will mutate the buffer's readerIndex when a Message is returned.

Define some basic tests:

```clojure
(ns nats.parser.parser-test
  (:require [clojure.test :refer [deftest testing is]])
  (:import [io.netty.buffer ByteBuf Unpooled]
           [nats.parser Parser Info Ping Pong]))

(defn str->bytes [s]
  (byte-array (count s) (map byte s)))

(defn bytes->str [buf]
  (apply str (map char buf)))

(defn str->ByteBuf [s] (-> s
                           str->bytes
                           (Unpooled/copiedBuffer)))

(defn ByteBuf->str [buf] (-> buf
                             (.slice)
                             (.array)
                             bytes->str))

(deftest parser
  (is (= Info
         (-> "INFO {\"server_id\":\"A0FE123\"} \r\n"
             str->ByteBuf
             (Parser/parseMessage)
             type)))
  (is (= Ping
         (-> "PING\r\n"
             str->ByteBuf
             (Parser/parseMessage)
             type)))
  (is (= Pong
         (-> "PONG\r\n"
             str->ByteBuf
             (Parser/parseMessage)
             type))))
```

