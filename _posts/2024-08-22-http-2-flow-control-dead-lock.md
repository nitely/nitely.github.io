---
layout: post
title:  "HTTP/2 flow control deadlock"
date: 2024-08-22 23:04:00 -0300
---

This is a high-level description of HTTP/2 flow control, potential deadlocks, delays, and how to prevent them. It may be useful for HTTP/2 implementers and the curious.

HTTP/2 allows full-duplex communication over a single stream, along with stream multiplexing. This means a peer can send and receive data simultaneously on a single stream and do the same on multiple streams at the same time.

Each peer has its own flow control and chooses the amount of data it is willing to consume at a time; this is the window size. If the sender transmits more data than the receiver’s window size allows or has processed, a protocol error is returned, and the connection is terminated.

The receiver will send window updates, informing the sender how much data it has processed since the last update. The sender will add this back to the window of data allowed to send.

There is a connection window, and each stream has its own window based on it. The connection window is shared among streams. Window updates are sent for both the connection and individual streams. This way, a sender knows which streams they can continue sending data on and ensures they do not exceed the overall connection window.

> Endpoints MUST read and process HTTP/2 frames from the TCP receive buffer as soon as data is available. Failure to read promptly could lead to a deadlock when critical frames, such as WINDOW_UPDATE, are not read and acted upon. [5.2.2. Appropriate Use of Flow Control](https://www.rfc-editor.org/rfc/rfc9113.html#section-5.2.2).

A deadlock occurs if the window update frames are not read and processed. At some point, the window size will be consumed, and a window update is needed to make progress. To avoid this, streams must never be allowed to block the receiving and processing of frames.

A delay occurs when the window update is sent after processing the received data. This delay is the time it takes for the application to process the data. We cannot send a window update as soon as data is received without processing it first, as that would defeat the purpose of flow control.

The solution is to process all frames asynchronously, storing the data frames in an internal stream buffer. This way, all frames can be processed as soon as they arrive.

The buffer size is capped by the stream window size, and since there is a connection window size, all of the stream buffers combined cannot use more memory than this.

The application will consume from this buffer and send a window update as soon as it does. This way, the program will only use up to twice the window size of memory: the data in the buffer and the data being processed.

This has the additional benefit of processing data frames while more data is being transferred, which can potentially improve performance.

Note: Only data frames are subject to flow control. The rest of the frames should be handled with other back-pressure mechanisms, such as bounded queues, to avoid resource exhaustion.

This is what [nim-hyperx](https://github.com/nitely/nim-hyperx) does at the time of writing.

## Bonus

At the application level, a deadlock can be caused by both peers sending data on a single stream at the same time without receiving it. The general way around this is for the application to always receive and send data simultaneously.

Some RPC protocols define how the message exchange works to avoid this kind of deadlock. For instance, a sender could transmit a stream of messages and then wait for a single message in reply. Alternatively, a sender could send a stream of messages and then wait for a stream of messages in return. Both peers could also send streams of messages in a ping-pong manner. A message protocol might involve sending a message size followed by the message body.

This is what gRPC does.
