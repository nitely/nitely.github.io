---
layout: post
title:  "HTTP/2 in-depth server design"
date: 2024-10-24 20:51:00 -0300
---

This is a high-level description of [nim-hyperx](https://github.com/nitely/nim-hyperx), an HTTP/2 server and client. It may be useful for HTTP/2 implementers and the curious.

This is not an overview of the HTTP/2 protocol. I won't go over frame types, stream states, [flow-control](https://nitely.github.io/2024/08/23/http-2-flow-control-dead-lock.html), nor [the spec](https://datatracker.ietf.org/doc/html/rfc9113) in general.

This is also not a guide or a how-to for building a server/client, as I believe that would require a book rather than a blog post. However, the [core](https://github.com/nitely/nim-hyperx/blob/master/src/hyperx/clientserver.nim) of nim-hyperx is ~1K LoC and can be read alongside this post.

Note that this was written in hindsight, after the implementation.

## Axioms

- Frames must be read and processed as soon as they arrive.
- Streams must not block the processing of frames.
- Streams must not block each other. Failing to read a stream in time must not block another stream, including the main stream.
- The main stream is a special stream and must process frames before subsequent frames are delivered to other streams. Since this can be blocking, it must be done as quickly as possible and independently of other streams.
- No historical stream information must be kept. There is no distinction between a closed stream and one that was never created.
- If a stream exists, it is considered alive in some way—either locally, remotely, or awaiting closure. This means it counts toward the limit of max concurrent streams.

## Data flow

```
[Frame receiver] -> [Stream frame dispatcher] -> [Stream frame receiver(s)]
```

- Frame receiver: Reads frames from the socket and places them in the stream frame dispatcher queue.
- Stream frame dispatcher: Takes frames from the queue and dispatches them to the streams.
- Stream frame receiver: There is one per stream. Received data frames are added to a stream buffer, which is consumed when the user calls `recv`. Other types of frames are processed here.

These are independent asynchronous tasks that run concurrently, meaning the user cannot block the receiving of frames. If the user stops consuming streams, *data frames* will eventually stop arriving due to flow-control limits, while other types of frames will continue to be processed. (Of course, the user could block the async/await event loop—cough, cough.)

Data communication is typically handled through asynchronous bounded queues. The queue provides backpressure: once it is full, the task will await until a frame is consumed before putting the new frame. This ensures frames are not received significantly faster than they are processed. The queue also serves as a buffer, allowing one frame to be processed while another is being received. This is also helpful during small spikes of received frames.

## Components

### Frame receiver

Frames are continuously read from the socket. This does all kind of frame checks, such as verifying frame size, payload value, and padding size. Continuation frames are received until the end, but the stream of continuations may be infinite, so it must be limited to an arbitrary total size (not specified in the spec).

If a check fails, a connection error will be thrown, typically a protocol error or frame size error.

Read frames are put into the `Stream frames dispatcher` queue.

### Stream frame dispatcher

Frames are taken from the queue and dispatched to their respective streams. Main stream frames are processed here because settings must be applied before processing subsequent frames.

The first time a stream frame is received, the stream is created. A check ensures that creating the stream does not exceed the maximum concurrent streams limit. The stream is then added to a *queue of created streams*, which the server processes. Server processing involves calling a user-provided callback, typically to send/receive headers and data.

If a stream cannot be created (e.g., due to an older `sid`) and does not exist, it is assumed the stream is in a closed state (i.e., a stream that existed and has been closed).

Header frames are decoded here as they must be decoded in the order they are received, so this cannot be done concurrently by individual streams. The frame payload is replaced by the decoded payload for practical purposes.

For data frames, the payload size is added to both the main flow-control window and the stream window. A check ensures that adding the payload size won’t exceed the window size.

Frames are put into an asynchronous value that must be consumed before the next frame is taken. Previously, this was managed with a queue per stream, but handling RST frames became complex since the remaining queue messages needed to be processed. Additionally, the queue allowed for the reception of invalid frames, which was less restrictive than the spec. Using an asynchronous value allows setting a value and waiting for it to be consumed before continuing.

### Stream frame receiver

There is one receiver per stream. It takes frames from its own frame queue and handles the stream state transitions.

The receiver processes the following types of frames:

- RST: Closes the stream.
- Window Update: Updates the stream's flow-control window, allowing more data to be sent, and emits a *window changed* signal.
- Headers: Validates the headers and stores them in the stream state.
- Data: Adds the data to a stream buffer, which is consumed when the user calls `recv`.

When the user calls `recv` for data/headers, the receiver may not have received any data or header frames yet. A common asynchronous pattern is to use a signal/event that can be awaited, which is set when there is data to consume. Alternatively, a queue of frames can be used.

## Stream State

Stream state is updated upon receiving and sending frames. There is a single stream state per stream. The transition from one state to the next depends on whether the frame is being received or sent. For example, if a header with the end-of-stream flag is received on a stream in the open state, the state becomes `half-closed-remote`. However, if it is sent instead, the state becomes `half-closed-local`.

## Stream cancelation

Upon stream cancellation (RST frame sent), the stream must remain alive until the peer has received the cancellation frame. Since there is no ACK for RST frames, a PING frame is sent right after. Once this PING frame is ACK'ed, it indicates that the peer must have received the cancellation frame, and the stream can then be effectively closed. Some frames may still be received during this [in-between state](https://nitely.github.io/2024/08/20/http-2-the-missing-state.html).

Note that the ACK'ed PING payload contains the same data as the sent PING. This data can include the stream identifier that sent it, enabling the main stream to notify the relevant stream through an asynchronous event once ACK'ed.

## API

### Recv

This function reads from the stream buffer and returns the data read. The stream and client flow-control windows are decreased by the amount of data read from the buffer.

A window update is sent once the amount of consumed data exceeds half the window size. This avoids sending a window update for every data frame consumed. The default window size is 64KB as per the spec. Increasing the window to 256KB has shown better performance in benchmarks testing throughput.

Failing to call `recv` in time will result in buffering up to the *control-flow window size* of data, with the default being 64KB as per the spec. The utilized window is shared among streams, so the total size of all buffers combined cannot exceed this limit.

### Send

This function creates a frame from the provided data and sends it. It attempts to send as much data as the control-flow window allows. If the data is too large to fit in a single frame, it is split into multiple frames. If there is no room in the window, the function waits for a window change signal until there is available space. It returns once all the data has been sent.

## Closing notes

My motivation to write this is mainly so I can come up with a better design in the process. Thinking about the current state of a code base, and putting it into words tends to bring new ideas to improve it. I also hope it can be useful to the reader.

Until next time.
