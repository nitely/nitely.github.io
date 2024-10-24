---
layout: post
title:  "HTTP/2 in-depth server design"
date: 2024-10-23 17:31:00 -0300
---

This is a design doc for [nim-hyperx](https://github.com/nitely/nim-hyperx), an HTTP/2 server and client. It may be useful for HTTP/2 implementers and the curious.

This is not an overview of the HTTP/2 protocol. I won't go over frame types, stream states, [flow-control](https://nitely.github.io/2024/08/23/http-2-flow-control-dead-lock.html), nor [the spec](https://datatracker.ietf.org/doc/html/rfc9113) in general.

## Axioms

- Frames must be read and process as soon as they arrive.
- Streams must not block processing frames.
- Streams must not block each other. Not reading a stream in time must not block another stream, including the main stream.
- The main stream is a special stream, and it must process frames before delivering subsequent frames to other streams. Since this can be blocking, it must do it as fast as possible, and inpendently of other streams.
- No historical stream data must be kept at all. There is no distinction between a closed stream and one never created.
- If a stream exists, it's alive in some way, either locally, remotely, or awaiting to be closed. This means it counts as an alive stream for the alive streams limit.

## Data flow

```
[Frames receiver] -> [Stream frame dispatcher] -> [Stream frame receiver(s)]
```

- Frames receiver: Reads frames from socket, and puts them in the stream frame dispatcher queue.
- Stream frame dispatcher: Takes frames from the queue, and dispatches them to the streams.
- Stream frame receiver: There is one per stream. Received data frames are added into a stream buffer. The buffer is consumed by user calling `recv`. The rest of frames are process here.

These are independent async tasks. They run independently of user code.

## Components

### Frames receiver

Frames are continously been read from socket. This does all kind of frame checks: valid frame size, valid payload value, valid padding size. Continuation frames are received until the end, the stream of continuations may not have an end, so it must be limited to some arbitrary total size (not given in the spec).

A failed check will throw a connection error. Usually a protocol error, or a frame size error.

Read frames are put into the `Stream frames dispatcher` queue. The queue is async, and once it's full, this task will block until a frame is consumed. This works as a back-pressure mechanism. Frames cannot be received much faster than they are being processed.

### Stream frame dispatcher

Frames are taken from the queue, and dispatched to their stream. Main stream frames are process here. This is because settings need to be applied before consuming following frames.

First time stream frames will create the stream. The stream is put into a queue of "received" streams, the server awaits and processes these streams. This involves calling a user provided callback. The callback usually send/recv headers, and data. It checks creating the stream won't exceed the max concurrent streams limit.

If the stream cannot be created (older `sid` for example), and it does not exist, it's assumed the stream is in closed state (i.e: a stream that existed, and got closed).

Header frames are decoded here. This is because headers must be decoded in the received order, so it cannot be done concurrently by the streams. The frame payload is replaced by the decoded payload for practical purposes.

Data frames payload size is added to the flow-control window, both the main window, and the stream window. It checks adding the payload size won't exceed the window size.

Frames are put into an async value that must be consumed before taking the next frame. This used to be a queue per stream, but it got complex when receiving RST frames, as the rest of queue messages need to be processed as well, and it was also less restrictive than the spec, as it allowed to keep receiving invalid frames. An async value allows to set a value and await for it to be consumed.

### Stream frame receiver

There is one receiver per stream. It takes frames from its own frames queue. it does the stream state transition.

It does the following processing of frames:

- RST: it closes the stream.
- Window Update: it updates the stream flow-control window.
- Headers: it validates the headers, and stores them into the stream state.
- Data: it adds the data into a stream buffer. The buffer is consumed by user calling `recv`.

When user calls `recv` data/headers, the receiver may not have received any data/header frame yet. A common async pattern is to use a signal/event that can be awaited to be set when there is data to consume. Alternatively, use a queue of frames.

### Components annex

All components are "tasks" that run independently of what the user code does. So the user cannot block the receiving of frames. Granted they could stop consuming streams, so at some point *data frames* will stop arriving because of control-flow limits. ~~They could also block the async/await event loop; cough, cough.~~

Data communication is often done through async bounded queues. These queues have the advantage of providing backpressure: once the queue is full, it must be awaited until an element is consumed to put one. In this case, upstream components need to wait for the frames to be processed before put a received frame. The queue also provides a buffer so a frame can be processed, while another frame is being received. It also helps when there are tiny spikes of received frames.

## Stream State

Stream state is updated on received and sent frames. There is a single stream-state per stream. The transition from one state to the next state depends on wheter the frame is being recv/sent. This because, for example: if a header with end of stream flag is received on a stream in open state, the state becomes half-closed-remote; but if it's sent instead, then it becomes half-closed-local.

## Stream cancelation

On stream cancelation, the stream needs to remain alive until the peer has received the cancelation frame. Since there is no ACK for these frames, a PING frame is sent right after, and once ACKed, the peer must have received the cancelation frame. The stream can be effectively closed then. Some frames can be received in this [in-between state](https://nitely.github.io/2024/08/20/http-2-the-missing-state.html).

## API

### Recv

This reads from the stream buffer, and returns the data read. The stream and client flow-control window are decreased by the amount of data read from the buffer.

A window update is sent once the amount of consumed window is above half the window size. This is to avoid sending a window update per every data frame consumed. The default window size is 64KB as per the spec. Increasing the window to 256KB has shown better performance benchmarks testing throughput.

Not calling `recv` in time will buffer up to "control-flow window size" of data. The default is 64KB as per the spec. The utilized window is shared among streams, so the sum of all buffers size cannot go over the limit.

### Send

This creates a frame of the provided data, and sends it. It tries to send as much data as the control-flow window allows. If the data is too big to fit in a single frame, the data is split into multiple frames. If there is no room in the window, it awaits for a window changed signal, until there is some room. Returns once the whole data is sent.

## Closing notes

My motivation to write this is mainly so I can come up with a better design in the process. Thinking about the current state of a code base, and putting it into words tends to bring new ideas to improve it. I also hope it can be useful to the reader.
