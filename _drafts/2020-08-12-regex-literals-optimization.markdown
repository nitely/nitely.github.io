---
layout: post
title:  "Regex literals optimization (or how to cheat on benchmarks)"
date: 2020-08-12 00:00:00 -0300
---

The regex literals optimization avoids running the regex engine on parts of the input text that cannot possibly ever match the regex.

It's not a general optimization as it does not work on every regex.

An example of a regex this can be applied to is `\w+@\w+\.\w+`, where the algorithm *quickly* finds the first `@`, then matches `\w+` backwards to find the start of the match, and then matches `\w+\.\w+` forward to find the end of the match. It then finds the second `@`, starting from the end of the previous match, and so on. This is a fairly naive (and incorrect) implementation, but it gives the idea of how it works.

I've recently implemented it in my pet project [nim-regex](https://github.com/nitely/nim-regex/pull/68), an NFA based regex engine that runs in (super)linear time. The results show it's around ~100x faster than before in some benchmarks. It's up to ~30x faster than PCRE when the optimization kicks in, and a bit slower otherwise. The tests are based on [mariomka/regex-benchmark](https://github.com/mariomka/regex-benchmark).

## Literals Optimization


