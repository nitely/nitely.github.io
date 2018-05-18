---
layout: post
title:  "An efficient circular queue of strings"
date: 2018-05-16 19:23:47 -0300
---

I've been interested in all-things HTTP/2 for a while now and I've meaning to write my own implementation of the whole protocol. One of the first steps is to implement the [header compressor](https://github.com/nitely/nim-hpack), so we can go from a blob of bytes to the headers in plain text that we all know.

This article is about one of the parts needed to build the compressor: a queue that can store strings efficiently.

## Queue Requirements

The requirements for this queue are quite simple: it has to limit the length of all strings put together and it has to limit the number of strings. But it must also be able to be reused efficiently.

My first approach to *make it work* was to just use a queue that contains references to individual strings, since that's built-in in my language of choice. However the only way to make this efficient is to use something similar to an [arena in C++](https://en.wikipedia.org/wiki/Region-based_memory_management). But even then, it wouldn't be this efficient, more on this later.

I did some research but I didn't have any luck finding an algorithm that would meet all the requirements. Thankfully, the implementation is pretty straightforward.

## The Queue

{% highlight nim %}
type
  StrQueue = object
    ## An efficient queue of strings.
    ## - ``s``: a string storing all substrings
    ## - ``pos``: stores the index to the end of last substring
    ## - ``filled``: tracks how much of ``s`` is being used
    ## - ``b``: stores the boundaries of each substrings
    ## - ``head``: the head of ``b`` queue
    ## - ``tail``: the tail of ``b`` queue
    ## - ``i``: tracks how much of ``b`` is being used
    s: string
    pos: int
    filled: int
    b: seq[Slice[int]]
    head: int
    tail: int
    i: int
{% endhighlight %}

This code is written in [Nim](https://nim-lang.org/), but it should be easy
to read for those coming from any procedural language.

There are two important things here: a single string of substrings and a dynamic array that keeps track of the boundaries of each substring. Note that this queue can be easily reused by simply resetting the counters.

To add an element, we first remove elements from the tail/end of the queue until there's enough space to store the new substring. The tail may be in any index of the vector and it moves in a circular way when removing the elements. Then, the head/start of the queue is increased to point to the new element.
By doing *modulo* we wrap-around when the actual end of the vector is reached. Lastly, the substring is stored pretty much the same way as its boundaries.

Note that since this is a *circular queue* the tail/head terms are reversed from a regular queue. This is because the used portion of it resembles a snake going around in a circle and so the head is the end at which elements are added, or [so I've read...](https://softwareengineering.stackexchange.com/questions/144477/on-a-queue-which-end-is-the-head)

{% highlight nim %}
proc add(q: var StrQueue, x: string) =
  while q.len > 0 and x.len > q.left:
    discard q.pop()
  if x.len > q.s.len:
    raise newException(ValueError, "string to long")
  q.head = (q.head+1) mod q.b.len
  q.b[q.head] = q.pos .. q.pos+x.len-1
  q.i = min(q.b.len, q.i+1)
  for c in x:
    q.s[q.pos] = c
    q.pos = (q.pos+1) mod q.s.len
  inc(q.filled, x.len)
  assert q.filled <= q.s.len
{% endhighlight %}

To remove an element from the end of the queue, we take the element at the tail and then the tail is incremented, moving the tail to the previous element.

{% highlight nim %}
proc pop(q: var StrQueue): Slice[int] =
  assert q.i > 0
  result = q.b[q.tail]
  q.tail = (q.tail+1) mod q.b.len
  dec q.i
  dec(q.filled, result.b-result.a)
  assert q.filled >= 0
{% endhighlight %}

## Optimizations

Here there are some optimizations:

* Use arrays to avoid allocations, if the queue is small enough.
* Make the string and boundaries size a power of two and do `x & (y-1)` instead of modulo.
* Use `memcopy` to store the substrings.

## Benchmarks

[@IJzerbaard](https://www.reddit.com/r/programming/comments/8k2lny/an_efficient_circular_queue_of_strings/dz4eu67/) asked me to add some [benchmarks](https://gist.github.com/nitely/35b57b22aa6d360b6f53f4b01762208e). So, I made some for the `add` function which is the most expensive one. Results on my laptop, YMMV:

```
============================================================================
GlobalBenchmark                                 relative  time/iter  iters/s
============================================================================
GlobalBenchmark                                            334.15ps    2.99G
============================================================================
crap.nim                                        relative  time/iter  iters/s
============================================================================
addWithModulo                                                1.38ms   722.64
addWithAnd                                      1652.41%    83.75us   11.94K
addWithMemCopy                                  36495.54%     3.79us  263.73K
simpleStrCopy                                                3.39us  294.82K
strMemCopy                                       270.54%     1.25us  797.60K
noOp                                                         0.00fs      inf
```

The bitwise `AND` solution is around ~16x faster than the modulo solution. Memcopy solution is around ~365x faster than the modulo one, also it's ~3x slower than doing a memcopy and nothing else.

The slowest solution was about as fast as the regular queue of string references I used at first, so the fastest solution is an improvement.

## Conclusion

Compared to a regular queue, this one can be preallocated at the start of the program and be efficiently reused over and over again, it potentially avoids cache misses, and it avoids resorting to something like arenas.
