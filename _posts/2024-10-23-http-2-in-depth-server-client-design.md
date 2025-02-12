---
layout: post
title:  "HTTP/2 in-depth server design"
date: 2024-10-24 20:51:00 -0300
---

This is a high-level description of [nim-hyperx](https://github.com/nitely/nim-hyperx), an HTTP/2 server & client. It may be useful for HTTP/2 implementers and the curious.

The [core](https://github.com/nitely/nim-hyperx/blob/master/src/hyperx/clientserver.nim) of nim-hyperx is ~1K LoC and can be read alongside this post.

This is not an overview of the HTTP/2 protocol. I won't go over frame types, stream states, [flow-control](https://nitely.github.io/2024/08/23/http-2-flow-control-dead-lock.html), nor [the spec](https://datatracker.ietf.org/doc/html/rfc9113) in general.

## Axioms

- Frames must be read and processed as soon as they arrive.
- Streams must not block the processing of frames.
- Streams must not block each other. Failing to read a stream in time must not block any other stream.
- The main stream must process frames before subsequent frames are delivered to other streams.
- No historical stream information must be kept. There is no distinction between a closed stream and one that was never created.
- If a stream exists, it is considered alive in some way—either locally, remotely, or awaiting closure. This means it counts toward the limit of max concurrent streams.

## Lightweight Threads

Most (all?) HTTP/2 servers & clients are implemented on top of user-level threads. In nim-hyperx, non-blocking async/await is used as the concurrency model. The design described in this article can be applied to other models, such as coroutines/fibers.

In theory one could use native threads alone, but note a single connection can open up to 100 streams, and the streams must be able to run independently of each other, meaning probably in their own thread, which could be too expensive and so scale poorly. User-level threads, on the other hand, are lightweight in resource usage, and they are ideal to implement HTTP/2 high concurrency.

## Using queues/channels for communication

Asynchronous units—futures, coroutines, fibers, etc—such as socket reader, the main stream, and each stream middleware can communicate using queues/channels. This means, the socket reader will read a frame, put it in a queue and go back to reading the next frame. The main stream will consume from the queue, process the frame and send a response (ex: settings ack, ping ack). A stream will consume from a queue, process the frame and put the data into a queue for the user to consume.

In nim-hyperx, all communication between these units are done through unbuffered channels. This means sending through a channel will block until another unit receives the value. Using buffered channels—or a bounded queue—showed no performance improvement in nim-hyperx; and it created funny race conditions: remember main stream frames must be processed before subsequent frames are delivered to other streams? consider what happens when a stream has unprocessed "old" frames in the channel but a newer main stream frame gets processed before them.

One good thing about async/await is that futures are cooperative/non-preemptive; they need to explicitly yield to give control to the next future callback. This means the main stream can receive a frame, process it, and yield control when either doing I/O (sending an ack) or waiting on the channel for the next frame. Asynchronous preemptive schedulers may required additional synchronization mechanisms, or just process the main stream in the same coroutine/fiber as the socket reader, but likely send the ack's asynchronously so sending does not block receiving new frames.

## Server.run()

The user must supply a callback that gets called on a stream open event. When a frame for a new stream is received, a new stream resource is created and put into a channel. There is a future which only job is receiving on this channel and creating a future for the user-callback. The user-callback gets passed the stream resource, which can be used to receive headers & data, and send headers & data!. These headers/data are received asynchronously by receiving on a channel. Note we are talking about regular headers & data, and not frames. Err... this future has an additional job: create a future to process the frames for that stream, and put the data payload and the decoded headers into the user-callback channel (these are two distinct channels, btw)—this is the so called stream middleware.

Did you notice headers need to be decoded? this is because plain-text headers—as seen in HTTP/1—are encoded using HPACK, which is a compression algorithm (mostly just a static table mapping for common headers, a dynamic table cache that is usually misused due to cache trashing, and Huffman code). Accordingly, headers need to be encoded when sending.

There is more! a future for processing main stream frames asynchronously must be created; processed frames includes pings, settings, control-flow window updates, and Go-Away. *Go-Aways* are sent when the client/server is closing the connection, sometimes gracefully (AKA the `NO_ERROR` code)—a server will refuse to process new frames after this, but it will keep processing active streams until they end—, sometimes due to an error—this can be a protocol error when sending an unexpected frame, or a malformed frame, a header compression error, a control-flow violation, etc—. *Pings* are... pings! for every ping, an ack must be sent as response. Some tend to believe pings can only be sent on the main stream, so they cannot be use to ping a stream to check if it's alive, this is true, but not entirely: a ping contains a payload, and the exact same received payload must be included in the ack, this means they can be used for anything, including adding streams IDs to *ping streams!* and making these streams ping back. In nim-hyperx, pings are used after sending a stream RST frame, so once the ping is ack'ed we know for sure the peer received the RST. *Settings* are connection wide settings such as frame max size, max concurrent streams, etc. *Control-flow window updates* are changes to the connection wide window. The window starts with a given capacity which is consumed when sending data, and it's refilled when receiving the window updates.

It is worth noting that flow-control is crucial to allow the user read from the stream at its own pace, avoiding overwhelming the server/client. Only up to control-flow window size of data can be received on a given stream (and connection). Once this window is exhausted no more data will be received. When the user reads data from the stream, a window update is sent to the peer to let them know they can send more data (equal to the amount read). The same applies for the sender; there are two control-flow windows, one for receiving, and one for sending. When sending, we can send as much as the available amount in the window allows, and wait for a window update to send more data. This is hidden from the user; the user may want to send more data than the full window allows, that's ok, the data gets sent in multiple frames if needed.

It'd be wasteful to send a new window update for every little data read, given the window size is usually much greater than a single frame. So, a window update is sent once a given amount of the window size has been consumed. In nim-hyperx, this is half the window size.

The stream middleware will receive frames; for data frames, it puts the data into a buffer that will later be read by the user. In nim-hyperx, a channel is used to notify there is data to read from the buffer. The user call to receive data will wait for it if there is no data in the buffer.

## Recap

There is one single frames receiver: it reads frames from the socket, it dispatches it to the open streams; if the stream is not open, it puts the stream into a channel. There is a single just-opened stream reader, which creates a future/coroutine/fiber for the user-callback, so the user can process the stream as they see fit—this is minimally reading headers, data, and sending headers, data. There is one stream middleware *per* stream, they read frames, process them into decoded headers, data, and put them into buffers, for the user to read. There is one main stream future to read and process main stream frames.

Note user-callbacks are for servers. What about clients? these are simpler, as it's the user opening the streams in their own future/coroutine/fiber.

## BONUS

### Stream State

Stream state is updated upon receiving and sending frames. There is a single stream state per stream. The transition from one state to the next depends on whether the frame is being received or sent. For example, if a header with the end-of-stream flag is received on a stream in the open state, the state becomes `half-closed-remote`. However, if it is sent instead, the state becomes `half-closed-local`.

### Stream cancelation

Upon stream cancelation (RST frame sent), the stream must remain alive until the peer has received the cancelation frame. Since there is no ACK for RST frames, a PING frame is sent right after. Once this PING frame is ACK'ed, it indicates that the peer must have received the cancelation frame, and the stream can then be effectively closed. Some frames may still be received during this [in-between state](https://nitely.github.io/2024/08/20/http-2-the-missing-state.html).

Note that the ACK'ed PING payload contains the same data as the sent PING. This data can include the stream identifier that sent it, enabling the main stream to notify the relevant stream through an asynchronous event once ACK'ed.

## Closing notes

I hope you enjoyed this article and found it useful. I may update it from time to time. Until next time.

> Last updated on 2025-02-12.
