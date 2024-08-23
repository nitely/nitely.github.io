---
layout: post
title:  "HTTP/2 flow control dead lock"
date: 2024-08-22 18:49:00 -0300
---

This is a high level description of HTTP/2 flow control. Potential dead-locks, and delays. And how to prevent them. Useful for HTTP/2 implementators, and the curious.

Each peer has it's own flow control, and chooses the amount of data they are willing to consume at a time, this is the window size. If the sender sends more data than the receiver's window size allows, or it has processed, then a protocol error is returned, and the connection is terminated.

The receiver will send window updates telling the sender how much data it has processed since the last window update. This is how much data the sender is allow to send.

Back to the initial question:

> What happens when both peers want to send data at the same time but there is not enough credit in the flow control window?

HTTP/2 allows full-duplex communication over a single stream, and streams multiplexing. This means a peer can send and receive data at the same time on a single stream, and do the same on multiple streams all at the same time.

A dead-lock occurs when both peers want to send data at the same time, but there is not enough credit in the flow control window.

A less deadly, but unwanted delay occurs when the window update is sent after processing the received data, depending on how long that data takes to be processed by the application.

This is somewhat mentioned in the spec:

> Endpoints MUST read and process HTTP/2 frames from the TCP receive buffer as soon as data is available. Failure to read promptly could lead to a deadlock when critical frames, such as WINDOW_UPDATE, are not read and acted upon.

The delay solution is simple. Keep a buffer of received data, and process all frames asynchronously. This way receiving data is not tied to whatever the application does. The buffer size is capped by the window size, and since there is a connection window size, all of the stream-buffers put togheter cannot use more memory than the connection window size.

The application will consume from this buffer, and send a window update as soon as it does. This way the application will only used up to 2 times the window size of memory. A single streams may block waiting for the data to be processed by the application, but this won't block the rest of the streams. No way around this, otherwise, we would use unbounded memory.

As for the dead-lock, the way around in general is for the application to always receive and send data at the same time. However, two peers can agree to do a ping-pong where they wait to receive/send complete messaged defined by some higher level protocol. A message protocol could be protobuf for example.

Some RPC protocols define how the message exchange works to avoid dead-locks. A sender could send a stream of messages and then wait for a single message as reply. Both could send a stream of messages in a ping-pong way. A sender could send a stream of messages where the last frame contains the *end of stream* flag, and then wait for a stream of messages as reply. An RPC that works like this is gRPC for example.

As a side note, only data frames are subject to flow control. The rest of frames should be handled with other back-pressure mechanisms, such as bounded queues to avoid resource exhaustion.
