# Reinventing Regex TNFA and TDFA

Tl;dr: This article describes a DFA based regex engine that supports sub-matches extractions. There is a [document](https://github.com/nitely/nitely.github.io/releases/download/0.0/jan_2020_dfa_submatches_extraction.pdf)[0] providing most of the interesting algorithms described here.

The construction algorithm creates a NFA (without ε-transitions) and a DFA that are able to extract sub-matches and assert empty matches (start/end of text, and word boundaries). The matching algorithm extracts full parse trees, not just a single match for each capture group (as most regex engines do).

The algorithms outlined here are named nNFA and nDFA to distinguish them from NFA/DFA and implementations supporting sub-matches.

## nNFA

We start by constructing a ε-NFA using Thompson's construction[1], this can be done in linear time. First parse the regular expression and create the ε-NFA states. Then transform the expression to Reverse Polish Notation by using a variation of the Shunting-yard algorithm. This allows to convert the sequence of states into a ε-NFA in linear time without recursion. The ε-NFA construction is well explained by Russ Cox[1]. We keep all of the transitions, including the capturing groups/sub-matches in their own states.

We can simulate (run) the ε-NFA in linear time O(N * M) where N is the lenght of the matching string and M is the lenght of the regex, as described by Russ Cox[1]. While doing this we go through the ε-NFA doing a Breadth-First Search (BFS), traversing the branches in parallel. When the simulation completes, we are left with one or more accepted states, the first accepted state is the winner branch and the one we care about (assuming left-to-right PCRE sub-matching). It's important to realize we can keep track of what is being matched and captured on each of the branches at all times, hence creating a BFS tree that happens to be a prefix-tree. The accepted state will hold the leaf of the path we care about. Traversing the prefix-tree from the leaf to the root will give us the (reversed) captured text boundaries.

The trick is to add a new node to the prefix-tree each time we find a start/end sub-match transition. This will contain the group ID/number, the matching string boundary/index we are at, and a reference or pointer to the parent node for the current branch.

The prefix-tree holds the complete history of captured sub-matches, while most regex engines only return the last capture for each sub-match, we can return all of the captures. The space complexity is O(N * M) as there is at most one sub-match tree branch per NFA (simulated) branch and input character. However we can keep the start/end *bounderies* of each capture instead of the characters, thus the size is usually fairly small. This also means we can potentially return all possible valid sub-matches, not just the first to match, but the left-most, right-most, longest, shortest, and all in between.

The correctness of this algorithm lays in the fact that we construct an exhaustive prefix-tree based on all of the paths the NFA took. Thus, the resulting tree must contain the branch that matched the string. This algorithm can be considered a variation of "Thompson's construction" and "Thompson's NFA simulation" algorithms. The main modifications are the addition of the open/close capturing group states to the ε-NFA at compile time and the construction of the prefix-tree at runtime.

The ε-NFA can be converted into a NFA by removing the ε-transitions. This avoids some recursion, and improves the simulation speed. However, transitions containing sub-matches and empty matches must be stored to look them up while simulating the NFA. This conversion is outlined in the nDFA section.

I've written two implementations of this. The first one a few years ago in Python[2], and a more recent and complete one in Nim[3].


## nDFA

DFA's merge (ε-)NFA's parallel states into single states, while constructing an automata that accepts the same input. This removes all the ε-transitions such as sub-matches, empty matches, repetitions, and alternations. However, we can keep a set/subset of the ε-NFA transitions and use them to contruct the sub-matches prefix-tree while executing the DFA.

The nDFA construction is as follows:

* Linearize the expression such as each letter in the expression E is unique in the expression E'.
* Convert the expression E' to ε-NFA using Thompson's construction.
* Convert the ε-NFA to NFA removing the ε-transitions T by computing the ε-closure on each state. Construct a set T' containing ε-transitions for each state and their ε-closure. Keep ε-transitions of sub-matches and empty matches, and merge them together if they are adjacent.
* Convert the NFA to DFA using Powerset construction[3], and (optionally) minimize it.
* Execute the DFA on a given input text, and construct the sub-matches prefix-tree following the T' transitions for each state of the DFA.
* Construct the matched text boundaries set by traversing the prefix-tree from the accepted state/branch to the root.

The meat of the algorithm is the ε-transitions removal and storage of interesting states, and the matching modification to construct the prefix-tree. The rest are known unmodified algorithms.

The NFA construction takes linear time in the number of states and transitions. The DFA construction takes exponential time[5]. The DFA execution takes O(N * M) in the lenght of the input text and the number of NFA transitions. However, it takes linear time if the regex does not contain sub-matches and empty matches, as the NFA transitions can be omitted.

The DFA supports all of the NFA features. Since we map DFA states to NFA transitions, we are implicitly simulating the NFA, so the solution is as correct as the NFA simulation. Unlike some DFA implementations supporting sub-matches, we don't need to handle edge cases, it's a general solution.

There's an alternative matching algorithm that simulates both DFA and NFA at the same time, using the DFA to match the input text, and the NFA to match the DFA and construct the prefix tree.

There's an implementation of this algorithm called nregex[6] that's written in the Nim programming language. Beware, it's not very optimized, however it already shows promising results, as the classical DFA matching is faster than PCRE in some cases. Nim allows powerful macros that will be used to generate optimized code and remove most of the current bottlenecks.


[0] https://github.com/nitely/nitely.github.io/releases/download/0.0/jan_2020_dfa_submatches_extraction.pdf
[1] https://swtch.com/~rsc/regexp/regexp1.html
[2] https://github.com/nitely/regexy
[3] https://github.com/nitely/nim-regex
[4] https://en.wikipedia.org/wiki/Powerset_construction
[5] https://en.wikipedia.org/wiki/Powerset_construction#Complexity
[6] https://github.com/nitely/nregex
