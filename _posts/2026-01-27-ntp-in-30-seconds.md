---
layout: single
title: "NTP in 30 seconds"
date: 2026-01-27
# toc: true
# toc_sticky: true
---

# What time is it?

>  There is a metrological fable that has been retold many times. It concerns
>  an eccentric retired sea captain who lived in the hills overlooking Zanzibar
>  City and fired a ceremonial cannon and raised the ensign at exactly noon each
>  day. He knew it was noon from his chronometer which he took pains to
>  accurately set whenever he passed the watchmaker’s window in town. The
>  watchmaker knew his clocks were accurate because he checked them daily when
>  that punctilious captain on the hill fired his cannon at noon exactly.
>
> Account of the _Zanzibar Effect_ in "Failures of the global measurement
> system. Part 2: institutions, instruments and strategy." Gary Price.

> Ballyhough railway station has two clocks which disagree by some six minutes.
> When one helpful Englishman pointed the fact out to a porter, his reply was
> "Faith, sir, if they was to tell the same time, why would we be having two of
> them?"
>
> The Five Clocks, Martin Joos

Two servers have clocks with the same frequency, but they don't agree on what
time it is. Server A says it's 10:00, Server B says it's 10:05. Server B has a
direct line to an atomic clock receiver, so it's more accurate.

Once Server A knows that Server B has the right time, it needs to synchronize
with it; Server A needs to discover that it's actually 10:05, not 10:00. It
could ask, but by the time the question made a round trip through the network,
the answer would already be out of date. So, Server A needs to estimate the
network latency between itself and Server B.

We can estimate latency without having synchronized clocks. Since latency is the
amount of time spent in the network, we only need to estimate the amount of time
_not_ spent in the network. Server A knows the round-trip time of its "What time
is it?" request, so it asks Server B to communicate how long it held the
message. `RTT - B_time = time_on_network`, which means the link latency is
`(RTT - B_time) / 2`.

Now we know the link latency, and Server A just needs to know the timestamp when
Server B received its "What time is it?" request and it can work out the offset.
And that's it.

## Example
Suppose Server A says it's 10:00:00 and Server B says it's 10:05:00, like above.
As a shortcut, instead of telling Server A both the duration and the timestamp
when it received the request, Server B includes the receipt timestamp and the
exit timestamp in its message back to A.

```
              ┌────────────What time is it?───────────┐       
              │                                       │       
              │                                       │       
              │10:00:00                               │10:05:01
       ┌──────┼───────┐                       ┌───────▼──────┐
       │              │                       │              │
       │              │                       │              │
       │ Server A     │                       │ Server B     │
       │              │                       │              │
       └──────────────┘                       └───────┬──────┘
              ▲ 10:00:03                              │10:05:02
              │                                       │       
              │         ┌──────────────────┐          │       
              │         │                  │          │       
              │         │  Recv: 10:05:01  │──────────┘       
              └─────────┼  Ret:  10:05:02  │                  
                        │                  │                  
                        └──────────────────┘                  
```

Server B held the packet for `10:05:02 - 10:05:01 = 1s` and the RTT was
`10:00:03 - 10:00:00 = 3s`, which gives a link latency of `(3 - 1) / 2 = 1s`.
And Server A now knows that Server B received the message at 10:05:01, and since
it sent the request at 10:00:00 and there's 1 second of network delay, that
means Server B's 10:05:01 is the same as Server A's `10:00:00 + 1s = 10:00:01`,
and Server A is exactly `10:05:01 - 10:00:01 = 5m` too slow.

In practice, a little algebra takes this from several steps to just one.

```
offset
  = ((B1 - A1) + (B2 - A2)) / 2
  = ((10:05:01 - 10:00:00) + (10:05:02 - 10:00:03)) / 2
  = (5:01 + 4:59) / 2
  = 5 minutes
```

## Notes
NTP involves a lot more than this algebra trick, but I think it's a very neat
trick. In addition to this formula, NTP solves for unreliable networks --
latencies are estimated by repeatedly sampling the latency; a single round trip
is not enough. Also, you can't just apply an offset to a clock. Many
applications and protocols assume that time only goes forwards, so if your clock
is ahead of the reference, care is taken. Instead of hopping back in time,
the system clock is slowed down slightly until the entire offset has been made
up for. When you're behind the reference, the standard protocol is to modestly
speed up the clock until you're caught up.

In extreme cases (like being off by 30 minutes) some hopping is used to avoid
the algorithm taking too long.

None of this so far addresses how servers agree on who has the authority and who
will sync to whose clock. That's determined by consulting a hierarchy of
authority based on how many hops you are from an atomic clock. Time sources are
stratified. The atomic clocks belong to Stratum 1, and anyone who gets their
time directly from an atmoic clock is in Stratum 2, and so on. Part of the NTP
exchange is comparing stratum and selecting the authority base on whose number
is lower. The "loser" sets their stratum to `s_authority + 1`.

And you might then ask, are all atomic clocks created equal? Where does that
authority come from? How is UTC defined? And for that, I don't have space on
this blog. Let's just be grateful that thanks to NTP, we don't have to worry
about it.

## Postscript
If you run Linux and want to see exactly what your `ntpd` daemon is up to, try
out `ntpviz` from the NTPsec project. Details here: https://blog.ntpsec.org/2016/12/19/ntpviz-intro.html


