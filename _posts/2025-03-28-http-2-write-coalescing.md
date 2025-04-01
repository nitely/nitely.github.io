---
layout: post
title: "HTTP/2 zero latency write coalescing"
date: 2025-03-28 19:11:00 -0300
updated: 2025-04-01 19:54:00 -0300
---

Write coalescing is an I/O optimization technique where multiple small writes are merged into a single larger write before sending data to the underlying system. In Http/2, we can batch multiple frames from one or more streams and send them all at once. This reduces the number of syscalls, and avoids sending tiny TCP packets under load.

## Nagle's Algorithm

Most OSes implement [Nagle's Algorithm](https://en.wikipedia.org/wiki/Nagle%27s_algorithm), which improves the efficiency of TCP/IP networks by reducing the number of packets that need to be sent over the network. However, it [interacts poorly](https://en.wikipedia.org/wiki/Nagle%27s_algorithm#Interaction_with_delayed_ACK) with another algorithm called [delayed ACK](https://en.wikipedia.org/wiki/TCP_delayed_acknowledgment), which can cause a delay of up to 500 milliseconds.

Because of this, most HTTP servers and clients *usually* disable Nagle's Algorithm or at least provide an option to do so. It can be disabled by setting [TCP_NODELAY](https://linux.die.net/man/7/tcp) on the socket.

A few weeks ago a [Nim](https://nim-lang.org/) user reported [a performance issue](https://github.com/nim-lang/Nim/issues/24741) when using Nim's std HTTP client. I immediately suspected that Nagle's Algorithm was the cause—this is one of those things that if you know, you know—and indeed, it was.

While I was reading up on the algorithm again, I thought the general idea is quite good. This got me thinking: could I implement something like it in my application code without the caveats?

After some rubberducking with the IA about buffering and batching techniques, and thinking how could I implement this in [nim-hyperx](https://github.com/nitely/nim-hyperx), the idea finally came to me.

## Write coalescing

Instead of sending frames right away for every stream in `socket.send` calls, the frames are added to a buffer. Then the entire buffer is sent in a single `socket.send` call by a coroutine that waits for the buffer to contain something. The frames are buffered while a `socket.send` is in progress. This keeps latency low since single frames can be sent right away, while multiple frames will buffer.

The buffer has a size limit, and when this size is reached, we wait for it to be flushed/drained (see the code below). This prevents consuming unbounded memory when we add to the buffer faster than we can `socket.send`, which is usually the case under some load.

Zero latency (or rather delay) here means we are not waiting for the buffer to reach a certain size, nor do we wait for a given time to pass, nor do we wait for an ACK, as Nagle's Algorithm does.

The relevant code looks like this:

```nim
# This is spawned on client creation
# and runs independently until the client,
# socket, or signal is closed
proc sendTask(client: Client) {.async.} =
  var buf = ""
  while not client.isClosed:
    while client.buf.len == 0:
      await client.bufSignal.waitFor()  # Wait buffer changed signal
    swap buf, client.buf
    client.buf.setLen 0
    client.bufDrainSignal.trigger()  # Fire buffer flushed signal
    check not client.sock.isClosed, newConnClosedError()
    await client.sock.send(addr buf[0], buf.len)

# This is called by a stream to send a frame
proc send(client: Client, frm: Frame) {.async.} =
  client.buf.add frm  # Add the frame to the buffer
  client.bufSignal.trigger()  # Fire buffer changed signal
  if client.buf.len > 64 * 1024:
    # Wait for the buffer to be flushed if size > 64KB.
    await client.bufDrainSignal.waitFor()
```

The buffer can grow larger than 64KB, but it's still memory-bounded. This is checked after adding to the buffer to ensure frames are being added in the correct `send` call order. The buffer's max size is `64KB+number_of_streams*max_frame_size`.

This optimization resulted in a 2x speedup, but more importantly, it sparked a series of optimizations that were only possible because of it, resulting in a 5x speedup. nim-hyperx can currently handle ~245K requests/s running in a single instance, and +1M requests/s running 8 instances on my budget laptop.

This isn’t just good for some silly benchmarks done on localhost, where there is no real I/O latency. It works better in the real world, and the longer the `socket.send` call takes to complete, the more frames are batched into the buffer.

## Profiling

At the time, it didn't occur to me to reduce the number of syscalls and send bigger data chunks to make better use of TCP packets. I usually use perf, valgrind (callgrind), and strace for profiling, but focus on ways of making the calls go faster rather than decreasing their number. Lesson learned.

## Closing notes

Nagle's algorithm is good, but sometimes it's worth to disable it and take care of the batching in the application code. Just disabling it, like nim-hyperx did at first, may not be good enough. Write coalescing is a batching technique that helps make better use of the network and reduce `socket.send` calls.

I hope you enjoyed this article and found it useful. Until next time.
