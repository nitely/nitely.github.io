---
layout: post
title:  "HTTP/2 in-depth server design"
date: 2024-10-23 17:31:00 -0300
---

This is a design doc for [nim-hyperx](https://github.com/nitely/nim-hyperx), an HTTP/2 server. It may be useful for HTTP/2 implementers and the curious.

This is not an overview of the HTTP/2 protocol. I won't go over frame types, stream states, flow-control, or the spec in general.

## Axioms

- Frames must be read and process as soon as they arrive.
- Streams must not block processing frames.
- Streams must not block each other. Not reading a stream in time must not block another stream, including the main stream.
- The main stream is a special stream, and it must process frames before delivering subsequent frames to other streams. Since this can be blocking, it must do it as fast as possible, and inpendently of other streams.
- No historical stream data must be kept at all. A fully closed stream does not exist. There is no distinction between a closed stream, or one never created.
- If a stream exists, it's opened in some way, either locally, remotely, or awaiting to be closed. This means it counts as an open stream for the open streams limit.

## Data flow

- Frames receiver: reads frames from socket, and puts them in the stream frame dispatcher queue.
- Stream frame dispatcher: takes frames from the queue, and dispatches them to the streams.
- Stream frame receiver: there is one per stream. Received data frames are added into a stream buffer. The buffer is consumed by user calling `recv`. The rest of frames are process here.

These are independent async tasks. Meaning they are spawn on connection creation, and keep running independently of user code, until the connection ends.

## Components

### Frames receiver

Frames are continously been read from socket. This does all kind of frame checks: valid frame size, valid payload value, valid padding size. Continuation frames are received until the end, the stream of continuations may not have an end, so it must be limited to some arbitrary total size (not given in the spec).

A failed check will throw a connection error. Usually a protocol error, or a frame size error.

Read frames are put into the `Stream frames dispatcher` queue. The queue is async, and once it's full, this task will block until a frame is consumed. This works as a back-pressure mechanism. Frames cannot be received much faster than they are being processed.

### Stream frame dispatcher

Frames are taken from the queue, and dispatched to their stream. Main stream frames are process here. This is because settings need to be applied before consuming following frames.

First time stream frames will open the stream. The stream is put into a queue of "received" streams, the server awaits and processes these streams. This involves calling a user provided callback. The callback usually send/recv headers, and data. It checks opening the stream won't exceed the max concurrent streams limit.

If the stream cannot be opened (older `sid` for example), and it does not exist, it assumes the stream is in closed state.

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

### Components annex

All components are "tasks" that run independently of what the user code does. So the user cannot block the receiving of frames. Granted they could stop consuming streams, so at some point *data* frames will stop arriving because of control-flow limits. ~~They could also block the async/await event loop for that matter; cough, cough.~~

## API

The 2 main user API are `recv` and `send`.

`recv` reads from the stream buffer. Not calling `recv` in time will buffer up to "control-flow window size" of data. The default is 64KB as per the spec. The utilized window is shared among streams, so the sum of all buffers size cannot go over the limit.

`send` creates a frame of the provided data. It tries to send as much data as the control-flow window allows. If the data is too big to fit in a single frame, the data is split into multiple frames. If there is no room in the window, it awaits for a window changed signal, until there is some room. Returns once the whole data is sent.
