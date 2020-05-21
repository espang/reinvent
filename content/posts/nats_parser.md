---
title: "NATS: Parser"
date: 2020-05-21T12:13:20+01:00
draft: false
---
# NATS: Parser

This parser is tighlty coupled to netty. Netty is optimised to minimize garbage and to reuse allocated objects/ memory. This parser tries to piggyback on that optimisation. The parser will be called after each read operation on the tcp socket and will be important for the speed of the system.

## Protocol

NATS uses a text based protocol on top of TCP. The protocol is described [here](https://docs.nats.io/nats-protocol/nats-protocol). The client will receive following operations:

* INFO
* PING
* PONG
* MSG
* +OK
* -ERR

some of them have additional data and all of them end with `\r\n`. `MSG` is slightly different as it contains two parts/ lines.

## Netty

Netty creates a chain of ChannelHandlers on top of tcp connections. Incoming data is handled by ChannelInboundHandlers - an interface that also implements ChannelHandlers. Netty uses [ByteBuf](https://netty.io/4.0/api/io/netty/buffer/ByteBuf.html) which has a readerIndex and a writerIndex describing the readable area of the buffer - readable are the bytes between readerIndex and writerIndex. The parser will get a ByteBuf and operate on the readable area. Given a buffer with following state

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

## Implementation

The protocol is very simple to parse and can be parsed in a single pass byte by byte. Let's define some helpful functions first:

```Clojure
(defn ^:private consume-byte! [[b1 b2] ^ByteBuf buf]
  (let [b (.readByte buf)]
    (or (= b1 b)
        (= b2 b))))
```

`consume-byte!` takes a `ByteBuf` and upto two bytes which are usualy a pair of the same letter, e.g. the bytes represented by 'p' and 'P'. The function consumes the byte at the position of the readerIndex and increases the readerIndex. The function returns a boolean true if the consumed byte was b1 or b2 and false otherwise. After validating a specific byte the protocol also has whitespaces in messages and any number of spaces and tabs should be handled as whitespace and are valid. Define a helper function to consume whitespace:

```Clojure
(defn ^:private consume-whitespace! [^ByteBuf buf]
  (let [next-non-ws-index (.forEachByte
                           buf
                           ByteBufProcessor/FIND_NON_LINEAR_WHITESPACE)]
    (if (< (.readerIndex buf) next-non-ws-index)
      (do 
        (.readerIndex buf next-non-ws-index)
        true)
      false)))
```

`consume-whitespace!` moves the readerIndex to the next non whitespace byte expecting at least 1 whitespace. Netty has a range of ByteBufProcessors implemented to find specific characters. One of them is used in `consume-whitespace` to find the first non whitespace byte. Another can be used to find the end `\r\n` of all nats messages which is useful to consume the rest of a message.

```Clojure
(defn ^:private read-line [^ByteBuf buf]
  (let [newline-index (.forEachByte buf ByteBufProcessor/FIND_LF)
        array         (byte-array (- newline-index ; newline
                                     1             ; return
                                     (.readerIndex buf)))]
    (.readBytes buf array)
    (when (and (consume-byte! return buf)
               (consume-byte! newline-byte buf))
      (String. array "UTF-8"))))
```

`read-line` gets a `ByteBuf` and converts all bytes from readerIndex to `\r\n` into a string. The ByteBufProcessor finds the first newline and returns the index. `readBytes copies the bytes upto `\r` into array. `consume-byte!` is used to validate that the byte before `\n` was `\r` and checks `\n` again to move the readerIndex after the newline. Parsing nats messages can be done by applying those helperfunctions: 

```Clojure
(defn parse [^ByteBuf buf]
  (.markReaderIndex buf)
  (try
    (let [first-byte (.readByte buf)]
      (cond
        (is-byte? m first-byte) (parse-msg buf)
        (is-byte? p first-byte) (parse-pingpong buf)
        (is-byte? i first-byte) (parse-info buf)
        :else :protocol-error))    
    (catch IndexOutOfBoundsException ex
      (.resetReaderIndex buf)
      nil)))
```

The parser relies on error handling to reset the readerIndex to the start in case the buffer only contains an incomplete message. Other errors and errors in the protocol should be handled differently and probably result into closing the connection. The `parse` function only checks the first byte and calls the appropriate function depending on that first byte.

```Clojure
(defn ^:private parse-info [^ByteBuf buf]
  (if (and (consume-byte! n buf)
           (consume-byte! f buf)
           (consume-byte! o buf)
           (consume-whitespace! buf))
    (let [content (read-line buf)]
      (if (some? content)
        {:msg/type :info
         :content  (json/read-str (str/trim content))}
         :protocol-error))
    :protocol-error))
```
`parse-info` validates that we receive the right command followed by whitespaces - the byte 'i' was read in the `parse` function. Here we expect the next bytes to be `nfo` followed by at least one whitespace. The rest of the line is the information about the server as json ending with \r\n. `read-line` is used to return the content of the info message. 

The parser is part of the nats-repository on my [github-page](https://github.com/espang/nats) and the parser can be found in `\src\nats\decode.clj`. Feedback is very welcome. The blog and the code is likely to change as long as this is an active project. 