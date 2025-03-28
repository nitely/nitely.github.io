---
layout: post
title: "HTTP/2 zero latency write coalescing"
date: 2025-03-28 15:23:00 -0300
updated: 2025-03-28 15:23:00 -0300
---

Write coalescing is an I/O optimization technique where multiple small writes are merged into a single larger write before sending data to the underlying system. In Http/2 we can batch multiple frames and send them all at once. This reduces the number of syscalls, avoids sending tiny TCP packets, and reduces latency caused by doing multiple writes. Moreover, it can be done adding zero latency.

## Nagle's Algorithm

Most OS implement the [Nagle's Algorithm](https://en.wikipedia.org/wiki/Nagle%27s_algorithm), which improves efficiency of TCP/IP networks by reducing the number of packets that need to be sent over the network. It has a [bad interaction](https://en.wikipedia.org/wiki/Nagle%27s_algorithm#Interaction_with_delayed_ACK) with another algorithm called [delayed ACK](https://en.wikipedia.org/wiki/TCP_delayed_acknowledgment) which can cause a delay of up to 500 milliseconds.

In nim-hyperx this has always been disabled, since I wouldn't ever send tinygrams, and I'd hope nim-hyperx users are not doing that either.

A few weeks ago a Nim user was having [a performance issue](https://github.com/nim-lang/Nim/issues/24741) when using Nim's std HTTP client. I immediately thought this has to be the Nagle's Algorithm biding—this is one of those things that if you know, you know—and indeed it was.

While I was reading on the algorithm once again, I thought it was a really good idea, and it got me thinking, can I implement this in my application code?

After doing a lot of rubberducking with the IA about buffering and batching techniques, and thinking how could I do this in nim-hyperx, the idea came out to me.

## Write coalescing

Instead of sending frames right away for every stream in `socket.send` calls, the frames are added into a buffer. The whole buffer is sent in single `socket.send` call at some point. The buffering is done while there is a `socket.send` in progress. There is a coroutine that continuously waits for the buffer to contain one or more frames and sends the buffer right away.

The buffer has a size, and when this size is reached, the coroutine adding to the buffer will wait for the buffer to be flushed. This avoids consuming unbounded memory when we add to the buffer faster than the coroutine can `socket.send`, which is usually the case under some load.

The relevant nim-hyperx code looks like this:

```nim
# This task is started on client creation
# and runs until the client is closed
proc sendTask(client: Client) {.async.} =
  var buf = ""
  while true:
    while client.sendBuf.len == 0:
      client.sendBufDrainSig.trigger()
      await client.sendBufSig.waitFor()
    swap buf, client.sendBuf
    client.sendBuf.setLen 0
    client.sendBufDrainSig.trigger()
    check not client.sock.isClosed, newConnClosedError()
    await client.sock.send(addr buf[0], buf.len)

# This is called by streams
proc send(client: Client, frm: Frame) {.async.} =
  client.sendBuf.add frm  # Add the frame to the buffer
  client.sendBufSig.trigger()  # Fire send buffer ready signal
  if client.sendBuf.len > 64 * 1024:  # Wait for the buffer to be flushed if size > 64KB
    await client.sendBufDrainSig.waitFor()
```

This is not only good for some silly benchmarks done in localhost, where there is no real I/O latency. It works better in the real world, as the longer the `socket.send` call takes to complete, the more frames are batched in the buffer.
