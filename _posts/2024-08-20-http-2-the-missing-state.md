---
layout: post
title:  "HTTP/2 the missing state"
date: 2024-08-20 18:49:00 -0300
---

There is a missing state in the HTTP/2 spec. Lets look at the *closed* state carefully:

> An endpoint that sends a RST_STREAM frame on a stream that is in the "open" or "half-closed (local)" state could receive any type of frame. The peer might have sent or enqueued for sending these frames before processing the RST_STREAM frame. An endpoint MUST minimally process and then discard any frames it receives in this state.

The actual *closed* state does not allow this. So, while this paragraph is in the *closed* state, it implies a hidden *almost-closed* state. This new state is the same as the *half-closed (local)* state for receiving, and the same as the *closed* state for sending.

Sending an RST in *half-closed (local)* state transitions to this *almost-closed* state. 

The *reserved-remote/local* states require similar care, but given push-promise has been largely disabled by server/clients, I got away with not implementing these states trasitions.

A stream in *closed* state must not keep a stream resource around, while one that is in this *almost-closed* state must, and it must count as an active stream, until the peer has received the RST frame. Otherwise, we would be open to a resource exhaustion attack. We can make sure they received the RST by sending a PING after, and waiting for the response.

Yes, yes, low effort kind of post, but I wanted to write about it, since it's probably one of the most confusing parts of the spec, and yet I may have got it wrong.

Related: https://github.com/nitely/nim-hyperx/pull/23
