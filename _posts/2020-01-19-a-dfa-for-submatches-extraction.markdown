---
layout: post
title:  "A DFA for submatches extraction"
date: 2020-01-19 08:10:00 -0300
excerpt: "Finite Automata is commonly used to efficiently match a Regular Expression (RE) to a given textinput. There are RE engines for submatch extraction based on Non-deterministic Finite Automata (NFA). These algorithms usually return a single match for each submatch, instead of the history of submatches (full parse tree). An NFA can be converted to a Deterministic Finite Automata (DFA) to improve the runtime matching performance. This article describes an algorithm based on DFA that extracts full parse trees from text."
---

Tl;dr: This article describes a DFA based regex engine that supports submatches extraction. There is a [document](https://nitely.github.io/assets/jan_2020_dfa_submatches_extraction.pdf) providing most of the interesting algorithms described here. There's a reference implementation called [nregex](https://github.com/nitely/nregex) written in [Nim](https://nim-lang.org/).

The construction algorithm creates a NFA (without ε-transitions) and a DFA that are able to extract submatches and assert empty matches (start/end of text, and word boundaries). The matching algorithm extracts full parse trees, not just a single match for each capture group.

The algorithms outlined here are named nNFA and nDFA to distinguish them from NFA/DFA and implementations supporting submatches.

## nNFA

We start by constructing a ε-NFA using Thompson's construction[1], this can be done in linear time. First parse the regular expression and create the ε-NFA states. Then transform the expression to Reverse Polish Notation (RPN) by using a variation of the Shunting-yard algorithm. The RPN to ε-NFA conversion is a simple non-recursive algorithm. The ε-NFA construction is well explained by Russ Cox[1]. We keep all of the transitions, including the capture groups/submatches in their own states.

We can simulate/run the ε-NFA in superlinear time O(N * M) where N is the length of the string to match, and M is the length of the regex, as described by Russ Cox[1]. While doing this we go through the ε-NFA doing a Breadth-First Search (BFS), traversing the branches in parallel. When the simulation completes, we are left with one or more accepted states, the first accepted state is the winner branch and the one we care about (assuming left-to-right PCRE sub-matching). It's important to realize we can keep track of what is being matched and captured on each of the branches at all times, hence creating a BFS tree that happens to be a prefix-tree. The accepted state will hold the leaf of the path we care about. Traversing the prefix-tree from leaf to root will give us the (reversed) captured text boundaries.

The trick is to add a new node to the prefix-tree each time we find a start/end submatch transition. This will contain the group ID/number, the matching string boundary/index we are at, and a reference or pointer to the parent node for the current branch.

The prefix-tree holds the complete history of captured submatches, while most regex engines only return the last capture for each submatch, we can return all of the captures. The space complexity is O(N * M) as there is at most one submatch tree branch per (simulated) NFA branch and input character. However we can keep the start/end *bounderies* of each capture instead of each character, thus the size is usually fairly small. This also means we can potentially return all possible valid submatches, not just the first to match, but the left-most, right-most, longest, shortest, and everything in between.

The correctness of this algorithm lays in the fact that we construct an exhaustive prefix-tree based on all of the paths the NFA took. Thus, the resulting tree *must* contain the branch that matched the string.

The ε-NFA can be converted to an NFA by removing the ε-transitions. This avoids some recursion, and improves the simulation speed. However, transitions containing sub-matches and empty matches must be stored to look them up while simulating the NFA. This conversion is outlined in the nDFA section.

I've written two implementations of this. The first one a few years ago in Python[2], and a more recent and complete one in Nim[3].


## nDFA

DFAs merge (ε-)NFAs parallel states into single states, while constructing an automata that accepts the same input. The classical conversion removes all the ε-transitions such as submatches, empty matches/zero-match assertions, repetitions, and alternations. However, we can keep a set/subset of the NFA transitions, and use them to contruct the submatches prefix-tree while executing the DFA.

The nDFA construction is as follows:

* Linearize the expression such as each letter in the expression `E` is unique in the expression `E'`.
* Convert the expression `E'` to `ε-NFA` using Thompson's construction.
* Convert the `ε-NFA` to `NFA` removing the ε-transitions `T` by computing the ε-closure on each state.
  Construct a set `T'` containing transitions for each state and their ε-closure. Keep ε-transitions of submatches and empty matches, and merge them together if they are adjacent.
* Convert the `NFA` to `DFA` using Powerset construction[3], and (optionally) minimize it.
* Run the `DFA` on a given input text, and construct the submatches prefix-tree following the `T'` transitions for each state of the `DFA`. Test the zero-match assertions as well.
* Construct the matched text boundaries set by traversing the prefix-tree from the first accepted state/branch to the root. If there is no accepted state a zero-match assertion failed to match.

The meat of the algorithm is the ε-transitions removal and storage of interesting transitions, and the matching modification to construct the prefix-tree. The rest are known unmodified algorithms.

The NFA construction takes linear time in the number of states and transitions. The DFA construction takes exponential time[5]. The DFA execution takes O(N * M) in the length of the input text and the number of NFA transitions. However, it takes linear time if the regex does not contain submatches and empty matches, as the NFA transitions can be skipped.

The DFA supports all of the NFA features. Since we map DFA states to NFA transitions, we are implicitly simulating the NFA, so the solution is as correct as the NFA simulation. Unlike some DFA implementations supporting submatches, we don't need to handle edge cases, it's a general solution.

There's an implementation of this algorithm called nregex[6] that's written in the Nim programming language. Beware, it's not very optimized, however it already shows promising results, as the classical DFA matching is faster than PCRE in some cases. Nim has powerful macros that are used to implement some optimizations, for example replacing dynamic data structures by case/switch statements and unrolling some loops.

## Benchmarks

The following benchmarks show nregex[6] is up to 22 times faster than PCRE. However, when the RE contains capture groups, PCRE is about 4 times faster than nregex.

|  | relative | time/iter | iters/s | regex | text
| --- | --- | --- | --- | --- | ---
CPU | | 294.85ps | 3.39G
PCRE | | 1.10ms | 912.11 | ^\w\*sol\w\*$ | (a\*100000)sol(b\*100000)
nregex | 739.52% | 148.25us | 6.75K
PCRE | | 174.87ns | 5.72M | ^[0-9]+-[0-9]+-[0-9]+$ | 650-253-0001
nregex | 2280.84% | 7.67ns | 130.43M
PCRE | | 179.23ns | 5.58M | ^[0-9]+..+$ | 650-253-0001
nregex | 1447.15% | 12.38ns | 80.74M


[0]: https://nitely.github.io/assets/jan_2020_dfa_submatches_extraction.pdf
[1]: https://swtch.com/~rsc/regexp/regexp1.html
[2]: https://github.com/nitely/regexy
[3]: https://github.com/nitely/nim-regex
[4]: https://en.wikipedia.org/wiki/Powerset_construction
[5]: https://en.wikipedia.org/wiki/Powerset_construction#Complexity
[6]: https://github.com/nitely/nregex
[7]: https://nim-lang.org/
