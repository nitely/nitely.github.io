---
layout: post
title:  "HTTP/2 flow control deadlock"
date: 2024-08-22 23:04:00 -0300
---

This is a high-level description of HTTP/2 flow control, potential deadlocks, and delays, and how to prevent them. It may be useful for HTTP/2 implementers and the curious.

Each peer has its own flow control and chooses the amount of data it is willing to consume at a time; this is the window size. If the sender sends more data than the receiver's window size allows or has processed, then a protocol error is returned, and the connection is terminated.

The receiver will send window updates telling the sender how much data it has processed since the last window update. This indicates how much data the sender is allowed to send.

HTTP/2 allows full-duplex communication over a single stream, along with stream multiplexing. This means a peer can send and receive data simultaneously on a single stream and do the same on multiple streams at the same time.

A delay occurs when the window update is sent after processing the received data, depending on how long that data takes to be processed by the application.

Albeit, I think it's at best a delay that can cause a single stream to slow down all streams, this is described as a deadlock in the spec:

> Endpoints MUST read and process HTTP/2 frames from the TCP receive buffer as soon as data is available. Failure to read promptly could lead to a deadlock when critical frames, such as WINDOW_UPDATE, are not read and acted upon.

The solution is simple. Process all frames asynchronously. Store the data frames in an internal stream buffer. This way all frames can be processed as soon as they arrive.

The buffer size is capped by the stream window size, and since there is a connection window size, all of the stream buffers put together cannot use more memory than the connection window size.

The application will consume from this buffer and send a window update as soon as it does. This way, the program will only use up to two times the window size of memory. The one that is being received, and the one that is being processed.

This has the additional benefit of processing data frames while more data is being transfer. Which can potentially improve performance.

Note, only data frames are subject to flow control. The rest of the frames should be handled with other back-pressure mechanisms, such as bounded queues, to avoid resource exhaustion.

## Bonus

At the application level, a deadlock can be caused by both peers sending data on a single stream at the same time, wihtout receiving it. The way around it in general is for the application to always receive and send data simultaneously.

However, two peers can agree to do a ping-pong where they wait to receive/send complete messages defined by some higher-level protocol. A message protocol could be protobuf, for example.

Some RPC protocols define how the message exchange works to avoid this kind of deadlock. A sender could send a stream of messages and then wait for a single message as a reply. Both could send a stream of messages in a ping-pong way. A sender could send a stream of messages where the last frame contains the end of stream flag and then wait for a stream of messages as a reply. An RPC that works like this is gRPC, for example.
