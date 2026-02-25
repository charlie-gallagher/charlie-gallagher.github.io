---
layout: single
title: "More consistent hashing, or learning math for the advanced in age"
date: 2026-02-25
---

I wrote my last post about consistent hashing, which has really stuck with me.
The problem statement is so simple but the solution seems at a glance to be
counter-intuitive. Sure, hashing both the file key and the node name is easy to
say and implement. But that's not enough for me to believe that it's sufficient
to solve the problem of data partitioning, nor that it's required. Isn't there
any simpler version? An alternative?

Consistent hashing solves a generic problem -- place keys on nodes, retrieve
keys from the correct node. At least, that's the apparent problem. But after
reading the paper that suggested consistent hashing, I've started to appreciate
the subtlety of the solution.

The paper describes the following desirable properties:

- Balance. Distribute items evenly across buckets.
- Monotonicity. If a new bucket is added, items may move into the new bucket,
  but no item moves between two buckets that were already there.
- Spread. The degree to which clients disagree about where a data item belongs
  when they don't agree about which buckets are available.
- Load. Given some set of clients with different knowledge about what buckets
  are available, the max load at any particular bucket.

Three out of the four properties concern the behavior of the system when storage
nodes are unstable. Monotonicity minimizes disruptions when nodes are added or
removed from the cluster. Spread and load define how storage nodes are treated
when clients have incomplete information about them. The other property,
balance, is the one I originally thought was interesting about consistent
hashing.

Lots of functions could have balance, any even partitioning of the key space
will work. The other criteria are harder to achieve.

Some modeling will help explore the problem space. Data items could be described
as a set of data items or a sequence of data items. If we model it as a
sequence, we can assign items to buckets using, for example, round robin
algorithms, but those are unavailable if we consider data items an unordered
set. In any case, an algorithm that depends on a specific sequence of data items
is more difficult to use in a distributed, imperfect-information system. A good
solution will minimize state, and so if we can, we should use an algorithm that
works on a set.

We could say that there is some set _K_ of all possible data keys, and that our
system must handle some reasonably-sized subset of _K_. So, given any subset of
_K_, our solution must satisfy the balance, monotonicity, spread, and load
properties. If you know the domain well, you could restrict this more and claim,
for example, that keys are expected to be alphabetically evenly distributed. If
you can't make assumptions about the distribution of keys, you have to assume
keys can be any subset of _K_. Prefixes stop working because you can always find
a subset of _K_ such that all elements have the same n-length prefix for any n.

Once you lay out your criteria, you try to satisfy them. A regular hash scheme
where you hash keys modulo the number of storage nodes has excellent balance but
poor monotonicity, spread, and load. It is a bad scheme under imperfect
conditions. On the other hand, partitioning on prefix (with some adjustments)
has excellent monotonicity, spread, and load, but the keys will most likely not
be evenly distributed.

The traditional consistent hash scheme (hash the keys, hash the storage node
names, assign keys to successors) actually doesn't have great spread or load.
It's too likely that some node will land close to another one, causing one node
to shoulder the burden. You have to use _virtual nodes_, which add some
complexity to the implementation but have nicer properties.

The paper uses the abstraction of a view, _V_, to describe what clients know
about storage nodes. The abstraction makes sense if clients themselves route
requests based on the data key, but does it work when clients just send
everything to a load balancer that forwards requests to a non-busy storage node
for further routing? I think it works. A view could also represent what
different storage nodes know about each other, and since we have some control
over what the different storage nodes know (via gossip) this changes the game a
bit. We can, like Chord, structure the views intentionally to guarantee certain
routing properties. You could still call each node's perspective a _view_ and
evaluate it on the criteria from the original paper. You could also reframe it
as a routing table with certain properties, though. Is that more or less useful?
It's certainly more specific, and we need to be specific to gain a performance
edge. But it's less generic, and we might be missing out on a "greater truth."

The paper leaves out a few practical criteria. Clients need to be able to
deterministically place and search for data items on particular nodes. The
function needs to be efficient. If routing is necessary[^1], nodes in the system
need to be able to route a request to the correct bucket efficiently.

[^1]: Routing was not necessary in the original paper, because the paper was concerned with cache nodes. Cache misses are a bit wasteful but not catastrophic. It's enough to minimize them, not make them impossible.


---

I let this discussion get a bit mathematical to match the tone of the paper,
but also because math has been on my mind lately. My math skills have atrophied
since I left college. I haven't even done a good proof in years.

But I've been reading comp-sci papers lately, and so I've started to turn that
part of my brain back on. After reflecting on it the past few weeks, I've come
to think that programmers and mathematicians are in a very similar business. The
history of math, just like prorgamming, has been a history of discovering which
abstractions, notions, and notations have useful properties for solving classes
of problems.

I like theory ok, but perfect mathematical reasoning is a burden in programming.
If you hold yourself to the standard of mathemetical proof, then you'll spend
forever modeling your system only to find that your proof is stretching for
pages and pages, and it still doesn't account for important things like
differences in computer architecture and maintenance cost. In practical systems,
you approximate and experiment. Software systems are [filled with magic
numbers](https://oxide-and-friends.transistor.fm/episodes/grown-up-zfs-data-corruption-bug)
found by experimentation.

Even still, I regret not being able to take more math classes in college,
because the mathematical thought process is so similar to the programmer's. You
find the abstraction with the right level of power for solving the problem
you're interested in. You wouldn't use ffmpeg to edit videos for the same reason
you wouldn't teach children basic algebra using group theory. The programmer
eventually learns that ffmpeg underlies everything, and I guess one way of
looking at work in mathematics is the process of discovering the lowest level
components that explain how the higher level components work. It's reverse
engineering Nature.

I've thought about starting to learn math again, at least to struggle with
problems every now and then. But how do you learn math at this point? What math
do you learn? I spent some time thinking about what you get out of a college
math degree.

There are a couple parts. There are the capabilities to reframe problems,
abstract things, unify, generalize, and simplify. Then the ability to write good
proofs and communicate with other mathematicians. And then there are the
subjects, the tradition, and the important results. The universal processes of
all math fields -- abstracting, reframing, and so on -- are my favorite parts. I
use those skills every day in programming and systems design. Proofs aren't too
bad. But I'm lacking majorly in the tradition.

I have that in common with another group of math enthusiasts, child prodigies.
How do they cope? In _The Man Who Only Loved Numbers_ about Paul Erdos, Paul
Hoffman says,

> Perfect numbers and friendly numbers are among the areas of mathematics in
> which child prodigies tend to show their stuff. Like chess and music, such
> areas do not require much technical expertise. No child prodigies exist among
> historians or legal scholars, because years are needed to master those
> disciplines. A child can learn the rules of chess in a few minutes, and native
> ability takes over from there. So it is with areas of mathematics like these,
> which are aspects of elementary number theory (the study of the integers),
> graph theory, and combinatorics (problems involving the counting and
> classifying of objects). You can easily explain prime numbers, perfect
> numbers, and friendly numbers to a child, and he or she can start playing
> around with them and exploring their properties. Many areas of mathematics,
> however, require technical expertise which is acquired over years of
> assimilating definitions and previous results.
>
> p. 48

And just as I was feeling better about the possibility that I could consider
myself "kinda good at math" without knowing the integral of 1/x, I saw on HN
yesterday a paper about [Terence Tao, age 7](https://news.ycombinator.com/item?id=47123689)[^2].
Sure enough, he was good at everything. "He has a prodigious long-term memory
for mathematical definitions, proofs and ideas with which has become
acquainted," writes the author of the paper M. A. Ken Clements. He mentions in
another part of the paper, "Not only did he have an astounding grasp
of algebraic definitions, for someone who was still seven years old, but I was
amazed at how he used sophisticated mathematical language freely." He was the
whole package at 7 years old. Widely read, good at math communication, and a
natural problem solver. The paper is a great read.

[^2]: I'd skip the comment section. It's mostly people flexing their own child prodigiousness and one thread about eugenics ("this proves to me that biological intelligence hasn't nearly reached its peak. If we select for pure intelligence, biological brains can get much smarter").

As an aside, I also learned how to evaluate a continued fraction from Terence in
this paper, and I'm wigging out over how programmerly the answer is. Here's the
whole section:

![Terence Tao solves a continued fraction at age 7]({{ "/assets/images/2026-math-and-comp-sci/ttao-continued-fraction.png" | relative_url }})

It's a recursive problem, so you introduce x as a recursive definition of the
continued fraction. Then it looks just like a quadratic as Terence's mom
suggested. Multiply both sides by x, rearrange, and solve for the positive
root. Of course it wouldn't work in a program since it recurses infinitely, but
it has the same feel as any recursive function.

Child prodigies get the problems and the intuition for mathematical constructs.
But I think I have a leg up in terms of appreciating the [experience of
mathematics](https://en.wikipedia.org/wiki/The_Mathematical_Experience). The
experience of math is the mind-altering effort to grasp the importance of a
pattern and what's required to state the problem in "simplest" terms. The
tradition of math on the other hand is the set of constructs, patterns, results,
and abstractions that have been found to be substantial to the history of
mathematics, both theoretical and applied. One isn't too useful without the
other. Sure I can think really hard about some problem, but without the
tradition of math, I can't build on any existing work. Like a programmer who
doesn't know what libraries are out there, and so has to build everything from
scratch.

I wish the author of the paper on Terence Tao had asked him what he thought
mathematicians did. He knew quite a few of them. Did he already appreciate that
math isn't just given in books, but thought, rethought, argued about, and
revised? (He probably did.)

---

Like in any tradition, it's hard to jump into mathematics. The most important
results of the last few decades are built on subleties, arguments, tradeoffs,
and alternatives that the casual student will never see. In my econometrics
class, we used matrices a bit for linear regressions. After a short introduction
about what matrices represent and how they work, we started on some definitions
like how matrix multiplication is defined. It was elusive to me at the time why
you would (a) have a non-commutative definition, and (b) do multiplication by
lining up row and column values pairwise. Who decided this, and why? But sure
enough, doing this led to the results we needed, so I put the questions in the
back pocket for later.

We trade off power for understanding, just as you don't need to really know what
a socket is if you take the usage examples for granted. But I think it's a
mistake to leave it there, to never encourage a student to figure out something
for themselves. Trying for a few days to derive some "obvious" idea is the only
way to appreciate the difficulty of coming up with good math abstractions. I
wasn't gifted in math, but I also wasn't encouraged to try to understand _why_
math is being taught one way or another. And I think that was a scholastic miss
on the curriculum's part.

One of my favorite essays from Jon Bentley's _Programming Pearls_ (technically
this one is from _More Programming Pearls_) is about writing your own profiler,
and in the end he writes one for Awk. It seemed sheerly impossible to me to
write a profiler, but I realized I had never even considered the problem space.
_Why not write a profiler?_ And why not try to find out for yourself what are
the properties of a connected vs partitioned graph of storage nodes? Why not try
to think up a new API for a threading library to rival pthreads? A bad idea can
be discussed and expanded on, but you can't do anything with no idea at all.

These days I'm pulled towards low-level programming, myself. I like the history
and the forgotten alternatives, the influential ideas that faded and yet
contributed to modern programming. But I also never forget that I my job is to
build effective software even if can't take the time to appreciate some tech
that I find a bit mysterious and interesting. I just have to file it away until
I have some spare time.

