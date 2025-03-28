---
layout: post
title:  "Regex literals optimization"
date: 2020-11-30 17:30:00 -0300
---

The regex literals optimization avoids running the regex engine on parts of the input text that cannot possibly ever match the regex.

An example of a regex this can be applied to is `\w+@\w+\.\w+`, where the algorithm *quickly* finds the first `@`, then matches `\w+` backwards to find the start of the match, and then matches `\w+\.\w+` forward to find the end of the match. It then finds the second `@`, starting from the end of the previous match, and so on. This is a fairly naive (and incorrect) implementation, but it gives the idea of how it works.

I've recently implemented it in my pet project [nim-regex](https://github.com/nitely/nim-regex/pull/68), an NFA based regex engine that runs in (super)linear time. The results show it's around ~100x faster than before in some benchmarks. It's up to ~60x faster than PCRE when the optimization kicks in. The tests are based on [mariomka/regex-benchmark](https://github.com/mariomka/regex-benchmark).

This is not to be confused with *Chivers' String Prefix Optimization*.

## Literals Optimization

Since nim-regex has to guarantee linear time, I'll describe optimizations that are guaranteed to take linear time. We must also ensure the matches are not overlapped.

Here's a high-level description of the algorithm:

  * We pick a literal that is `memchr`'ed to skip parts of the text.
  * The prefix is the regex part before the literal; none of the
    characters or symbols within the prefix must match the literal.
  * The prefix is ran backwards to find the start of the match.
  * A full scan is ran from the start of the match
    until a character that cannot be matched is found (safe break point)
    or the end is reached. The scan tries to start the match at every character (NFAs can do this in linear time).
  * Go to step one and repeat from the last scanned char. Make the prefix
    match until the previous last scanned char.

There are two important constraints to picking a literal:

  * *"none of the characters or symbols within the prefix must match the literal"*, why? consider the regex: `\d\w+x`, and the input text: `xxxxxxxxxxx`; this would take quadratic time, as the prefix will match until the start of the string every time. What about matching the prefix until the previous last scanned char? we'd need to match past that sometimes, ex: regex: `\d\w+x`, and text: `1xxx`. If we add this constraint, the literal becomes a delimiter, and these cases are solved.
  * The literal cannot be part of a repetition, nor it can be part of an alternation. For example: `(abc)*def` the first literal candidate is `d`, since `(abc)*` may or may not be part of the match. Same thing for alternations.

These constraints can be relaxed in some cases, see the *"Other optimizations"* section.

Here's the main algorithm in [Nim](https://nim-lang.org/):

```nim
func findAll(
  matches: var Matches,
  text: string,
  regex: Regex,
  start: int
): int =
  let limit = start
  var i = start
  while i < text.len:
    i = memchr(text, regex.lit, i)
    if i == -1:
      return -1
    let litIdx = i
    i = matchPrefix(text, regex, i, limit)
    if i == -1:
      i = litIdx+1
    else:
      i = findSome(matches, text, regex, i)
      if i == -1:
        return -1
      if matches.len > 0:
        return i  # this is used as "start" to resume the matching
  return -1
```

A given character may be consumed only twice, once by the backward prefix match, and a second time by the forward scan. Hence the algorithm runs in linear time.

I may describe how `matchPrefix` and `findSome` work, how to construct the reversed NFA in the right order, and how to pick the literal in a future article. The nim-regex code contains descriptions of the algorithms, though.

## Benchmarks

The [benchmarks](https://github.com/nitely/nim-regex/tree/master/bench) regexes are based on [mariomka/regex-benchmark](https://github.com/mariomka/regex-benchmark). The only difference is the regexes are pre-compiled, so just the matching is tested. The results show nim-regex is ~63x faster than PCRE in the email test, and ~2x faster in the URI and IP tests.

Why is nim-regex so fast in the email case? The regex engine doesn't run as often. There are orders of magnitud more IP/URI candidates than email candidates (`@` chars within the text) to match. In the former case the time is dominated by the regex engine, while in the latter case it's dominated by searching the char literal.

```
==================================================
GlobalBenchmark       relative  time/iter  iters/s
==================================================
GlobalBenchmark                  294.86ps    3.39G
==================================================
bench.nim             relative  time/iter  iters/s
==================================================
pcre_email                        21.76ms    45.96
nim_regex_email       3247.14%   670.02us    1.49K
nim_regex_email_macro 6335.93%   343.38us    2.91K
pcre_uri                          22.15ms    45.14
nim_regex_uri           92.82%    23.87ms    41.90
nim_regex_uri_macro    256.29%     8.64ms   115.68
pcre_ip                            5.73ms   174.58
nim_regex_ip            88.70%     6.46ms   154.84
nim_regex_ip_macro     214.75%     2.67ms   374.91
```

> Note Nim's PCRE is at the top of the mariomka/regex-benchmark. I ran those benchmarks, and IIRC nim-regex was just a bit faster, mainly because the non-macro regex engine is slower (see the above results), and the regex compilation is also tested.

## Other optimizations

Here are other possible optimizations:

  * Picking a literal —even if the prefix matches it— should take linear time as long as the prefix is bounded (i.e: does not contain repetitions), ex: `\d\wx`.
  * Picking a literal within a "one or more" repetition/repetition group should be possible, since `(abc)+` matches the same as `abc(abc)*`.
  * It's better to pick the last literal within the first literal sequence, since that way we always try to match as many literals as possible early on, and potentially fail early. We want to keep the prefix regex as short as possible. We want the prefix to be bounded if possible. So, this sounds like a good heuristic.
  * Alternations can be optimized this very same way in some cases, ex: `bar|baz`, since both alternations have `ba` in common, either `b` or `a` can be picked as the literal.
  * Alternations can be optimized in other cases. PCRE seems to use `memchr` or similar for up to two alternation terms. A DFA could be used to quickly match candidates instead of `memchr`, as that's a more general solution.
  * Literals substring optimization: pick a literal delimiter, and then grab its surrounding literals. The substring can be searched using `memmem` instead of `memchr`. This also has the advantage of easily supporting multi-byte utf-8 characters (unicode).

## Conclusion

Literals optimization is not a general optimization as it does not work on every regex, but when it does, it can greatly improve the matching speed.

Can a backtracker like PCRE implement this? PCRE in particular already has some sort of similar optimization, but it's not as good/fast as this one. Backtrackers cannot implement this as described here exactly, but they can do something similar that requires backtracking. If they provide a resumable `find` function, then probably yes.

Hopefully, more regex engines will implement these sort of optimizations, so there are more compelling alternatives to backtrackers such as PCRE.
