---
layout: post
title:  "Regex literals optimization (or how to cheat on benchmarks)"
date: 2020-08-12 00:00:00 -0300
---

The regex literals optimization avoids running the regex engine on parts of the input text that cannot possibly ever match the regex.

An example of a regex this can be applied to is `\w+@\w+\.\w+`, where the algorithm *quickly* finds the first `@`, then matches `\w+` backwards to find the start of the match, and then matches `\w+\.\w+` forward to find the end of the match. It then finds the second `@`, starting from the end of the previous match, and so on. This is a fairly naive (and incorrect) implementation, but it gives the idea of how it works.

It's not a general optimization as it does not work on every regex, but when it does, it can greatly improve the matching speed.

I've recently implemented it in my pet project [nim-regex](https://github.com/nitely/nim-regex/pull/68), an NFA based regex engine that runs in (super)linear time. The results show it's around ~100x faster than before in some benchmarks. It's up to ~35x faster than PCRE when the optimization kicks in. The tests are based on [mariomka/regex-benchmark](https://github.com/mariomka/regex-benchmark).

## Literals Optimization

Since nim-regex has to guarantee linear time, I'll describe optimizations that are guarantee to take linear time. We must also ensure the matches are not overlapped.

I think the best way to understand how the current nim-regex implementation works is by example. However, I'll describe some parts of the algorithm that may be useful to grasp first:

  * We pick a literal that is `memchr`'ed to skip parts of the text.
  * The prefix is the regex part before the literal; none of the
    characters or symbols within the prefix can match the literal.
  * The prefix is run backwards to find the start of the match.
  * A full `find_all/search/scan` is run from the start of the match.
    until a character that cannot be matched is found (safe break point)
    or the end is reached.
  * Go to step one and repeat from the last scanned char. Make the prefix
    match until the previous last scanned char.

There are two important constraints to picking a literal:

  * `none of the characters or symbols within the prefix can match the literal`, why? consider the regex: `\d\w+x`, and the input text: `xxxxxxxxxxx`; this would take quadratic time, as the prefix will match until the start of the string every time.
  * The literal cannot be part of a repetition, nor it can be part of an alternation. For example: `(abc)*def` the first lit candidate is `d`, since `(abc)*` may or may not be part of the match. Same thing for alternations.

Here's some pseudo code of the main algorithm:

{% highlight nim %}
func findAll(text: string, regex: Regex, start: int): int =
  var matches: Matches = @[]
  var i = start
  var limit = start
  while i < text.len:
    limit = i  # rather pointless since the literal is a delimiter
    i = memchr(text, regex.lit, i)
    if i == -1:
      return -1
    var litIdx = i
    i = matchPrefix(text, regex, i, limit)
    if i == -1:
      i = litIdx+1
    else:
      i = findSome(matches, text, regex, i)
      if i == -1:
        return -1
      if matches.len > 0:
        return i  # this is used as "start" to resume the matching
{% endhighlight %}

A given character may be consumed only twice, once by the backward prefix match, and a second time by the forward scan. Hence the algorithm runs in linear time.

I may describe how `matchPrefix` and `findSome` work, how to construct the reversed NFA (hint: it's not just reversing the edges/transitions, the order matters), and how to pick the literal in a future article. The nim-regex code does have some explanations, though.

## Other optimizations

There are other possible optimizations:

  * Picking a literal —even if the prefix matches it— should take linear time as long as the prefix is bounded (i.e: does not contain repetitions).
  * Picking a literal within a "one or more" repetition / repetition group should be possible, since `(abc)+` matches the same as `abc(abc)*`.
  * It's almost always better to pick the last literal within the first literal sequence, since that way we always try to match as many literals as possible early on, and potentially fail early. We want to keep the prefix regex as short as possible, so the picking a lit in the first sequence is best.
  * Alternations can be optimized in some cases. PCRE seems to use `memchr` or similar for up to two alternation terms. A DFA could be used to quickly match candidates instead of `memchr`, as that's a more general solution.
