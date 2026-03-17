---
title: "Multiprocessing TSV repair"
date: 2026-03-17
layout: single
---

A few days ago I wrote about
[optimizing a TSV repair script]({% post_url 2026-03-12-tsv-repair %}) that took
large TSV files with unquoted newline characters like this:

```
id	name	comment	score
0	charlie	normal	20
1	alice	this is a
multiline comment	10
2	bob	normal	8
```

And turned them into valid TSV files like this:

```
id	name	comment	score
0	charlie	normal	20
1	alice	this is a multiline comment	10
2	bob	normal	8
```

I initially ignored multiprocessing as being pesky, complicated, and unlikely to
yield good results. But after returning to it, I found that not only is it a
tractable problem, but it could potentially have real performance benefits.

I've written a compatible multiprocessing TSV repair script that has taken the
best end-to-end time from 5.78s down to 3.04s (though the average is somewhat
worse).

# What I missed
In the last post, I discounted the multiprocessing approach for the following
reason:

> We can't align on a row boundary without knowing how many tab characters have
> come before the current `seek` position, and we can't do that without indexing
> all of the tabs in the file. That's what we're already doing in the basic
> implementation of the TSV repair script, so I don't see a way for
> multiprocessing to be faster than sequential processing.

I didn't consider that you could _also parallelize the tab indexing_.

This is the basis of the _two-pass_ approach to distributed parsing of
delimited files. The first pass uses workers to gather statistics about the
file. Then, the master uses those statistics to re-evaluate the naive ranges it
assigned to the workers. Finally, the master tells the workers what
modifications they need to make to their assigned ranges to work with only
complete records, and the workers go to work. For a complete description, see
[https://badrish.net/papers/dp-sigmod19.pdf](https://badrish.net/papers/dp-sigmod19.pdf).

For valid delimited files, where newline characters are allowed only if they're
quoted, the real question for each worker is whether or not their initial
position is in a quoted field or not. In the paper linked above, the authors use
a speculative approach to decide whether or not a worker is beginning in a
quoted field or not. It's a nifty technique, and if you're interested I
recommend reading through the paper. The short version is that each worker
"sniffs" the first megabyte or so of data in their chunk and makes an educated
guess about whether or not they started in a quoted field. They no longer
communicate back with the master, they just proceed with their guess. If they
encounter an error, the whole things falls back to the two-pass approach.

In my case, there are no quotes, so I had to stay with the two-pass approach and
adapt it to the malformed TSVs I'm dealing with.

But before I get to the new script, I wanted to mention one more interesting
feature of these TSV files that I didn't notice before.

# Inherent ambiguity
While working on the multiprocessor script, I realized that I had missed an
ambiguity lurking in these malformed TSV files. Given a file like the following,
there's no "correct" interpretation:

```
one,two,three
four
five,six,seven
```

This could be interpreted in one of two ways:

```
# Version 1
one,two,three\nfour
five,six,seven

# Version 2
one,two,three
four\nfive,six,seven
```

The problem is that when the final field contains a newline character, we can't
say whether it's a record delimiter or an unquoted newline.

It's computationally easier to use a non-greedy approach, which produces
interpretation Version 2. You read lines and stop reading as soon as you have
accumulated the correct number of field delimiters (tabs, commas). During
processing of the above ambiguous snippet, the processor first reads the line
`one,two,three` and finds it complete. Then, it starts building the next record
with `four`, which it joins with the next line, after which join it finds that
this record is now complete.

Foruntunately this rule is as easy to follow for a sequential processor as for a
parallel processor.

# Adapting the two-phase parallel parser
To make this work, I had to make the definition of a row a little more strict.
The parallel processor can no longer tolerate lines that have too many tabs in
them -- it's assumed that every record is composed of the same number of fields.

With that assumption, it becomes a given that the total number of tabs in the
file is a multiple of the number of tabs in the header, i.e. `n_fields - 1`. So,
I split the file evenly into chunks and assign each worker a chunk. The workers
count the number of tabs in their chunks and report back to the master process.
The master process then figures out how many tabs each worker needs to _skip_ in
order to land on a record boundary. The workers then treat the rest of their
range as a normal TSV file, stopping once they've read their last full record
that started before the end of their chunk. So workers often read past the end
of their assigned byte range in the file, and the calculations performed by the
master process ensure that the next worker down the line knows how far the
previous worker had to over-read.

The implementation is a minefield of boundary issues and off-by-one errors, but
in the end I got this working ok and reasonably efficiently. The full
implementation is available [on GitHub](https://github.com/charlie-gallagher/tsv-repair/blob/main/repair_bytes_buffered_read_write_multiprocessing_two_pass.py).

# Performance
As I said in the last post, for best performance, you should leave the files in
separate pieces, and that's exactly what I did for the performance benchmarks. I
did write an optional extension that recombines the files, and I used that to
confirm that the "repaired" version of various files matched a known-good
processor.

The performance is unstable, but generally very good. Performance seems to
depend on how busy the system is with other, more "important" work.

Still, it's sometimes exceptional. The best recorded time so far was 3.04
seconds, almost a 2x improvement on the previous best time. Here's a smattering
of results, with a normal sequential run thrown in the middle.

```
2026-03-17 15:35:40	repair_bytes_buffered_read_write_multiprocessing_two_pass	3.394725
2026-03-17 15:35:57	repair_bytes_buffered_read_write_multiprocessing_two_pass	3.986272
2026-03-17 15:36:08	repair_bytes_buffered_read_write	6.599493
2026-03-17 15:36:18	repair_bytes_buffered_read_write_multiprocessing_two_pass	9.798484
2026-03-17 15:36:35	repair_bytes_buffered_read_write_multiprocessing_two_pass	7.341621
2026-03-17 15:37:02	repair_bytes_buffered_read_write_multiprocessing_two_pass	3.043876
```

The trimmed mean is 4.9 seconds.

When I turn on the feature that re-combines the files at the end, performance
dips to more like 12 seconds per run, more or less as expected.

# Comments
This was a serious bump in complexity and peskiness, but I'm thrilled with the
performance (most of the time). I think there might be a bit more performance to
be squeezed out of it. The I/O could most likely be faster when I'm scanning
forward to skip tabs, but I'm plenty happy with this implementation, and I'm
satisfied that the cost of scanning forward is bounded by the number of fields,
not the number of rows.

Profiling becomes tricky with multiprocessing, so I skipped it for these runs.
If you're a profiler junky, I'll gladly accept any PRs.

Ultimately I wouldn't recommend this for production, because it depends so
heavily on there being a correct number of tabs. Any misalignment and the
quality goes out the window, or you'd have to write a guard that makes the
master process fall over if the number of tabs is wrong. But fun to get working
anyway!

