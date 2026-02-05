---
layout: "single"
title: "That darn ballistic-electron logic"
date: 2026-01-30
---


> The ntpd utility can synchronize time to a theoretical precision of about 232
> picoseconds. In practice, this limit is unattainable due to quantum limits on
> the clock speed of ballistic-electron logic.
>
> (https://docs.ntpsec.org/latest/ntpd.html)

# Have the time?
I always assumed I was using ntpd to keep time on my linux computer. But I was
only sort of right.

According to the Debian Wiki, since Debian 12, the default NTP client is
`systemd-timesyncd`. It uses SNTP (Simple Network Time Protocol), which
implements the client (no option to host a time server), and it sets the time
roughly by communicating with a single time server. There's no recourse if you
get a bad server, or "falseticker" in NTP parlance.

There are a few implementations of NTP to choose from. The systemd-timesyncd
daemon is a basic client suitable for keeping time. The original NTP reference
implementation is `ntpd`, which is still around, but is deprecated on Debian in
favor of the more security-focused [NTPSec](https://ntpsec.org). And then
[Chrony](https://chrony-project.org) is a newer implementation that is more
practical than `ntpd`. It looks like a darn fine timekeeper [by
comparison](https://chrony-project.org/faq.html#_how_does_chrony_compare_to_ntpd).

There are interesting things to say about each NTP tool (and their apparent
[controversies](https://www.linux-magazine.com/Online/Blogs/Off-the-Beat-Bruce-Byfield-s-Blog/NTPsec-The-Wrong-Fork-for-the-Wrong-Reasons)),
but if you're interested in NTP you can pick pretty equally among `ntpd`,
Chrony, and NTPSec. I've been playing with NTPSec for its debugging utilities
like out-of-the-box data visualizations using `ntpviz`.

---


Most computers have a real time clock (hardware) and a system clock (software).
On powerup or reboot, the system clock is set using the RTC. Normally, using a
command like `date` to set the date only updates the system clock; to update the
hardware clock, you'll need

```
hwclock --systohc
```

The hardware clock is battery-driven, which is how its time reading persists
across boots. But some parts of it are curiously system-dependent, for example
whether the hardware clock is set in UTC or local time.

> If your machine dual boots Windows and Linux, then you could have problems
> because Windows uses localtime for the hardware clock; while Linux and Debian
> use UTC for the hardware clock. In this case you have two choices. The first
> is to use localtime for the hardware clock, and set Debian to use localtime.
> The second is to use UTC for the hardware clock, and set Windows to use UTC.
>
> https://wiki.debian.org/DateTime


NTP implementations like Chrony and NTPSec don't directly interact with the RTC;
instead, they modify the system clock. They _tend_ to make use of a kernel
feature called "11-minute mode", where the system clock syncs to the hardware
clock every 11 minutes, but documentation on this is a bit scant. Some comments
in the [Chrony
docs](https://chrony-project.org/faq.html#_real_time_clock_issues).

Real time clocks are usually crystal oscillators with a frequency of 32.768 kHz
(hey there's that 16-bit signed integer number again), and since NTP doesn't
directly interact with them, so I'm not going to talk much more about them.

Software clocks on the other hand are crucial to the system. Every system that
NTP runs on must provide a time correction service. David Mills (who designed
NTP) also defined the `adjtime` syscall, intended to be portable. As far as I've
seen, it's POSIX standard. You might also see `adjtimex`, which is a
Linux-specific variant.

This describes how the system call works, from _The Design and Implementation of
the FreeBSD Operating System_:

> The `settimeofday` system call will result in time running backward on
> machines whose clocks were fast. Time running backward can confuse user
> programs (such as `make`) that expect time to invariably increase. To avoid
> this problem, the system provides the `adjtime` system call \[Mills, 1992\].
> The `adjtime` system call takes a time delta (either positive or negative) and
> changes the rate at which time advances by 10 percent, faster or slower, until
> the time has been corrected. The operating system does the speedup by
> incrementing the global time by 1100 microseconds for each tick and does the
> slowdown by incrementing the global time by 900 microseconds for each tick.
> Regardless, time increases monotonically, and user processes depending on the
> ordering of file-modification times are not affected. However, time changes
> that take tens of seconds to adjust will affect programs that are measuring
> time intervals by using repeated calls to gettimeofday

The Mills reference is to [RFC 1305](https://datatracker.ietf.org/doc/html/rfc1305).

Since I have TDAIOTFBSDOS open already, I can mention a few other things about
a typical POSIX software clock works. The system software clock is created
through an interrupt timer, and the system "increments its global time variable
by an amount equal to the number of microseconds per tick. For the PC, running
at 1000 ticks per second, each tick represents 1000 microseconds," (73). And if
you think 1000 interrupts per second is a lot of interruption, you're right. "To
reduce the interrupt load, the kernel computes the number of ticks in the future
at which an action may need to be taken. It then schedules the next clock
interrupt to occur at that time. Thus, clock interrupts typically occur much
less frequently than the 1000 ticks-per-second rate implies," (65-66).

I'd guess (and a brief conversation with ChatGPT seems to confirm) that modern
operating systems have heavily optimized this part of their timekeeping. After
all, who cares what the time is if no one's there to see it?


---

> There is not one way of measuring time more true than another; that which is
> generally adopted is only more _convenient_.
>
> Henri Poincaré

What would it take for _me_ to serve time to others? NTP servers listen on port
123 and usually work only over UDP, so I suppose the simple way to serve time is
to start ntpd in server mode, start listening, and configure someone to use you
ask their authority.

But if you really want to be seen, you have to join the
[pool](https://www.ntppool.org/en/).

> The pool.ntp.org project is a big virtual cluster of timeservers providing
> reliable, easy to use NTP service for millions of clients.
>
> The pool is being used by hundreds of millions of systems around the world.
> It's the default "time server" for most of the major Linux distributions and
> many networked appliances (see information for vendors).
>
> https://www.ntppool.org/en/

There's very clear documentation on [how to join the
pool](https://www.ntppool.org/en/join.html), too. The basic steps are:

- Get your own time from a known good source (_not_ the pool).
- Configure a stable IP address (trickier than you might think -- even if you
  set up port forwarding to get around DHCP issues, your ISP tends to rotate
your public IP address as it wants).
- Be willing to make a long-term commitment to the project.

I'll put "create a home time server" on my list of things to try, but joining
the pool would probably create too big a wave.

---

All this had me wondering, where does authority in timekeeping come from? Who
has the time? I'm only an hour's drive away from the [National Clock and Watch
Museum](https://museum.nawcc.org), and after visit, notbook in hand, a few dozen
hours of followup research, I'm still not sure.

What I have learned is this. We get our sense of time from the periodic
movements of the starry firmament -- the sun, the seasons, the stars. And our
bodies (along with most organisms on Earth) have built-in timers that encourage
us to do those activities that keep us alive. It's been a while since you ate,
the sun is down, which when you normally sleep, and so on. Biological clocks (as
anyone with a cat will tell you) can be extremely precise.

We also seem to be hardwired to prefer coordinated time. One of the earliest
forms of coordinated time is yawning, which we all know is evolutionarily
contagious.

At the moment my interest in timekeeping isn't how we developed precision
clocks, but how we managed to coordinate ourselves using those clocks. _Why_ we
coordinate ourselves using those clocks.

Early religion (who?) temporalized life with its emphasis on doing certain
religious acts multiple times a day. Some of the earliest interesting clock-like
devices we have are from monasteries that rang bells at specific times. (And the
word _clock_ is derived from the French word for bell.) This went on for a few
hundred years.

The next advance was periodic time. Time used to be more organic than it is
today. Hours were not equally sized, and the day was not split equally into 24
parts. But somewhere, at some time, Europeans made an intuitive leap from
continuous time devices like the clepsydra or the procession of different stars
and planets to discrete time -- time as ticks[^1]. In _Revolution in Time_, David
Landes considers this one of the great methodological leaps in western
civilization, coming seemingly out of no where.  It took other cultures another
500 years to begin using oscillating, periodic timekeepers.

Nearly as soon as clocks became more convenient and domestic, _punctuality_
became an important social cue.

[^1]: Escapements convert potential energy (like a falling weight suspended by a rope) into periodic motion (the ticking of a hand). There's no better visualization of the development of the clock than Bartosz Ciechanowski's [mechanical watch](https://ciechanow.ski/mechanical-watch/). And while these escapements were crude in the beginning, it took only a few breakthroughs until they were able to tell time within a few seconds per day.

---

I installed NTPSec on my Debian machine and left the configuration mostly as-is.

```
sudo apt-get update && sudo apt-get install ntpsec ntpsec-doc ntpsec-ntpviz
```

I made sure to enable statistics, because I'm really after visualizations. I
want to see the thing do stuff. Visualizations are generated using `ntpviz`,
which is scantily documented (this was helpful but ancient:
[ntpvis-intro](https://blog.ntpsec.org/2016/12/19/ntpviz-intro.html)). I found
enough to get me going. But having only just set up my daemon, there's still no
data to visualize. I took the chance to do some background work on the metrics.

Clocks are never perfectly in sync (even UTC is calculated as an average), and
the most important contributor to incorrect timekeeping is a difference in
oscillator frequencies. This is called frequency _skew_. If the "correct" time
is an oscillator at 1000 Hz, my local computer clock might be more like 1001 Hz
or 999 Hz. So even if I set my clock to the right time, I would gain or lose
some seconds every day.

Frequency skew is measured in parts per million, which is to say the number of
periods fast or slow per million oscillations. In the 1000 Hz example, 1001 Hz
would have a skew of 1 part in every thousand, or 1000 parts per million (ppm).
999 Hz has a skew of -1000 ppm.

Skew is also described in other ways. A human-friendly way to describe it is
"seconds gained or lost per day", or year, or whatever. This gives you the
number in practical terms. It's a bit tricky to translate between them, though,
considering the gap between oscillation frequency and length of a day.

Your skew might also vary over time, and this is called _drift_.

NTP corrects for skew (but not necessarily drift) as part of the protocol by
nudging the time and doing its best to predict changes. But skew is dynamic, and
it is affected by the quality of the hardware and things like changes in local
temperature (room temperature or computer temperature) and even humidity
(CLAIM). For ideal timekeeping, you'll want to keep your computer in a nice
climate-controlled vault with excellent heat sinking.

The offset is the estimated difference between my clock and the reference clock,
measured in milliseconds. I show roughly how this is calculated in NTP in 30
Seconds TODO: link. In short, it's calculated by estimating the latency between
you and the server and using that to guess what time the server received your
request. Then you compare your guess (based on local time + latency) to what
the server reported was the "actual" time it received the request, and use the
difference to work out how wrong your clock is.

Practically speaking, for general monitoring, you can use `ntpmon`. This is a
top-like tool for watching your NTP daemon interact with peers. The output looks
something like this:

```
     remote           refid      st t when poll reach   delay   offset   jitter
 0.debian.pool.n .POOL.          16 p    -   64    0   0.0000   0.0000   0.0001
 1.debian.pool.n .POOL.          16 p    -   64    0   0.0000   0.0000   0.0001
 2.debian.pool.n .POOL.          16 p    -   64    0   0.0000   0.0000   0.0001
 3.debian.pool.n .POOL.          16 p    -   64    0   0.0000   0.0000   0.0001
-ip74-208-14-149 192.58.120.8     2 u  598 1024  377  41.7645   0.9319   1.5714
-144.202.66.214. 162.159.200.1    4 u  834 1024  377  45.3224   1.1844   1.2529
*nyc2.us.ntp.li  17.253.2.37      2 u  564 1024  377  10.2534  -0.8732   0.8516
+ntp-62b.lbl.gov 128.3.133.141    2 u  748 1024  377  73.6187  -0.3344   1.0547
+time.cloudflare 10.102.8.4       3 u  212 1024  377   8.4884   0.0523   0.9331
 192-184-140-112 .PHC0.           1 u  66h 1024    0  85.8202   5.3690   0.0000
+ntp.nyc.icanbwe 69.180.17.124    2 u  639 1024  377  11.8765  -0.1314   1.1134

ntpd ntpsec-1.2.2                             Updated: 2026-02-04T08:17:40 (32)

 lstint avgint rstr r m v  count    score   drop rport remote address
      0   1284    0 . 6 2    321    1.217      0 51529 localhost
    212   1054   c0 . 4 4    127    0.050      0   123 time.cloudflare.com
    564   1079   c0 . 4 4    123    0.050      0   123 nyc2.us.ntp.li
    598   1058   c0 . 4 4    126    0.050      0   123 ip74-208-14-149.pbiaas.com
    639   1066   c0 . 4 4    125    0.050      0   123 ntp.nyc.icanbwell.com
    748   1055   c0 . 4 4    126    0.050      0   123 ntp-62b.lbl.gov
    834   1066   c0 . 4 4    125    0.050      0   123 144.202.66.214 (144.202.66.214.vultruser
```

I'll describe peer metrics in a second. For now, the second table (starting with
`lstint`) is the MRU list (most recently used). Here are the stats it reports.

- `lstint` Interval (s) between receipt of most recent packet from this address
  and completion of the retrieval of the MRU list by ntpq.
- `avgint` Average interval (s) between packets from this address.
- `rstr` Restriction flags.
- `r` Rate control indicator.
- `m` Packet mode
- `v` Packet version number.
- `count` Packets received
- `score` Packets per second (averaged with exponential decay)
- `drop` Packets dropped
- `rport` Source port of last packet received
- `remote address` The remote host name

There are commands you can use to change the output, like `d` for detailed mode.

For a snapshot, you can use `ntpq`, a helpful tool for inspecting the daemon. It
has an interactive mode and a one-shot mode. This queries peers in the one-shot
mode.

```
$ ntpq --peers --units
     remote                                   refid      st t when poll reach   delay   offset   jitter
=======================================================================================================
 0.debian.pool.ntp.org                   .POOL.          16 p    -   64    0      0ns      0ns    119ns
 1.debian.pool.ntp.org                   .POOL.          16 p    -   64    0      0ns      0ns    119ns
 2.debian.pool.ntp.org                   .POOL.          16 p    -   64    0      0ns      0ns    119ns
 3.debian.pool.ntp.org                   .POOL.          16 p    -   64    0      0ns      0ns    119ns
-ip74-208-14-149.pbiaas.com              192.58.120.8     2 u  671 1024  377 41.765ms 931.92us 1.5714ms
-144.202.66.214.vultrusercontent.com     162.159.200.1    4 u  907 1024  377 45.322ms 1.1844ms 1.2529ms
*nyc2.us.ntp.li                          17.253.2.37      2 u  637 1024  377 10.253ms -873.2us 851.60us
+ntp-62b.lbl.gov                         128.3.133.141    2 u  821 1024  377 73.619ms -334.4us 1.0547ms
+time.cloudflare.com                     10.102.8.4       3 u  285 1024  377 8.4884ms 52.298us 933.07us
 192-184-140-112.fiber.dynamic.sonic.net .PHC0.           1 u  66h 1024    0 85.820ms 5.3690ms      0ns
+ntp.nyc.icanbwell.com                   69.180.17.124    2 u  712 1024  377 11.877ms -131.4us 1.1134ms
```

Here's how this table is interpreted (from the `ntpmon` man page):

- `tally` (symbol next to remote) One of `space`: not valid, x, ., -: discarded
  for various reasons, +: included by the combine algorithm, \#: backup, \*:
system peer, 0: PPS peer. Basically, look for the `*` and any `+` signs to see
who you're listening to right now.
- `remote` The host name of the time server.
- `refid` The RefID identifies the specific upstream time source a server is
  using. In other words, it names the reference clock (stratum 0 or 1), even if
this server is just repeating what that reference clock says.
- `st` NTP stratum
- `t` Type. u: unicast or manycase, l: local, s: symmetric (peer), server, B:
  broadcast server.
- `when` sec/min/hr since last received packet.
- `poll` Poll interval in log2 seconds
- `reach` Octal triplet. Represents the last 8 attempts to reach the server.
  `377` is binary `11111111`, which means all 8 attempts reached the server.A
value like `326` is binary `11010110`, meaning out of the last 8 attempts, the
3rd, 5th, and 8th attempts failed.
- `delay` Roundtrip delay
- `offset` Offset of server relative to this host.
- `jitter` Jitter is random noise relative to the standard timescale. (Get
  better definition.)

For more complete definitions, see `man ntpmon`.

Many of these (especially in the MRU list) are technical and mostly of interest
to those already experienced with NTP. I'm not, so I've focused on a few of the
more interesting metrics: tally, reach, delay, offset, and jitter. These are the
same metrics that `ntpviz` reports on.

---

> There is a law of error that may be stated as follows: small errors do not
> matter until large errors are removed. So with the history of time
> measurement: each improvement in the performance of clocks and watches posed a
> new challenge by bringing to the fore problems that had previously been
> relatively small enough to be neglected.
> 
> _Revolution in Time_ p. 114

For a long time, agreement between clocks didn't matter. Citizens of the US in
the 19th century had timekeepers, but they set them using "apparent solar time",
or time estimated by when the sun is highest in the sky. There were also
sundials that told slightly better solar time, and astronomers could keep time
better still by looking at the movements of the planets and stars. (But who had
an astronomer in those days?) Besides the sun, you had church bells and tower
clocks. Ye old tower clock in most cases was set by sundial, and "none too
accurately" in the words of the clock museum. Religious clocks were more of a
suggestion of the time.

Coordination wasn't important in the US until the railroads. When you're
coordinating a few hundred trains in and out of stations, timekeeping becomes
quite important. For most of the 19th century, each railroad company had its own
timekeeping system and standards for accuracy. In the middle of the century,
there were 144 official time zones in North America alone.

How do you get 144 time zones? Any move along a line of latitude (i.e. east or
west) causes the sun's apparent apex to move. When the sun is highest in the sky
in Pennsylvania, it's still rising in Colorado. Your sundial would in that case
create infinite time zones for each variation in longitude. The railroads
"solved" this by using a standard time for each major city they stopped in. You
got a sort of "average solar time" for this stretch of railroad.

The railways kept different time, though, and this led to accidents and
fatalities that finally motivated the US to move to _standard time_ based on
only 3 main timezones, the same basic ones we use today.

If you're like me, you might get nervous thinking about the logistics of
suddenly changing the time, but while the topic of changing from "God's time" to
an official time was controversial, the actual change seems to have gone well.
There was a [day of two
noons](https://guides.loc.gov/this-month-in-business-history/november/day-of-two-noons)
on November 18, 1883, and that was it. Incidentally, it wasn't until the 1918
Standard Time Act that Congress actually acknowledged the new time scheme, but
by then it was well known.

---

After a day or two, I checked back in on my NTP stats to see what I'd collected.
For my distribution, the data collects in `/var/log/ntpsec/`. Running `ntpviz`
on this folder will generate an HTML report with all of the default data
visualizations.

```
nptviz -d /var/log/ntpsec/
open ntpgraphs/index.html
```

The interesting graph for me is the first one, which plots clock offset (ms,
left axis) and frequency skew (ppm, right axis). My clock is slow, pretty
consistently, by about 7ppm. That is, over 1 million oscillations, my clock will
read 7 units less than the authority. As long as this is consistent, that's ok.

\[FIRST GRAPH\]

At some point on Jan 31, I suddenly found myself 4ms ahead of the reference
clock, and the ensuing correction was a bit too big. But the last day or two has
been very stable.

The next graph shows "RMS time jitter" (RMS=root mean square), or in other words
"how fast the local clock offset is changing." The tip under the graph says that
0 is ideal, but it doesn't give me a sense of whether my clock with a 90% range
of 0.528 is any good. It seems spiky.

\[SECOND GRAPH\]

And a third graph shows RMS _frequency_ jitter, similar metric but for my
oscillator's consistency.

\[THIRD GRAPH\]

Skipping down a bit, there's a fun correlation graph between local temperature
and the frequency offset. My computer apparently measures temperature in two
different places (one consistently warmer than the other). You can see how
sudden changes in temperature correlate closely with changes in the frequency
offset. The spikes are caused by the space heater in my office.

---

More story here.

---


> ntpd does most computations in 64-bit floating point arithmetic and does
> relatively clumsy 64-bit fixed point operations only when necessary to
> preserve the ultimate precision, about 232 picoseconds. While the ultimate
> precision is not achievable with ordinary workstations and networks of today,
> it may be required with future gigahertz CPU clocks and gigabit LANs.
>
> https://linux.die.net/man/8/ntpd

Debian has recently switched to using `ntpsec` as its primary NTP server
(CLAIM), so a quick:

```
sudo apt-get update && sudo apt-get install ntpsec ntpsec-doc ntpsec-ntpviz
```

was enough to get started. That's good because one of the main reasons I'm
trying this out is to use `ntpviz`.

```
ntpq -p -ddd
```

Or if you're inside the program, you can use the commands documented in the man
page I think. Don't worry about reading the spec, it's useless.

See linux laptop (`~/Documents/notes/ntp`). Tomorrow, I'll see about the ntpviz
side of things. I also still have to put together the narrative around why
`ntpsec` isn't standard. What's sntp (simple ntp)? It was funny that there was
controversy around the ntpsec folks. But I'll probably skip it for the article.

# Clock museum
We were at the clock museum today, a few notes:

- Got good photos of the atomic clocks, or at least some component of the atomic
  clocks.
- Lots of good stories. Railroad motivated standard timekeeping, otherwise
  people mostly used "apparent solar time" and sundials to set their clocks.
- Sam mentioned that there was a time outage at the Colorado NIST standard
  cesium clock. It's true! https://www.npr.org/2025/12/21/nx-s1-5651317/colorado-us-official-time-microseconds-nist-clocks
- Got two books, Revolution in Time (anti clock) and _What is time?_, which I'm
  reading now, since the mechanical part is shorter.
- Sam gave me good tips on two technical, casual writers: Bob Pease and Jim
  Williams. I have a Jim Williams book on pdf in our whatsapp. "You fix
everything!"
- Jerrold R. Zacharias is the "father of the atomic clock."
- Nov 18, 1883 -- the day with 2 noons
- 1918 standard time act
- uniform time act of 1966
- Ball time
- Clocks stored in vaults. Main challenges are environmental conditions
  (temperature, pressure, humidity, contaminants) and physical imperfections
(inconsistent manufacturing, rough surfaces, parts that wear).
- Electric horology, Alexander Bain.
- How were chronometers set? Still need an answer. "Any clock used as a
  navigational tool could not err more than a few seconds a day." (poster)
- "Up until 1883, the majority of Americans obeyed _apparent solar time_, or
  time told by the sun. Even though many people owned some type of mechanical
timekeeper (which kept _mean time_), these clocks and watches were set by a
sundial or the position of the sun itself." (poster)
- Potential structure: history of timekeeping cut with my own adventure in ntp
  (since ntp by itself isn't very fun).
- Hours were more organic before periodic timekeepers; but asian technologists
  invented precise water clocks that obeyed the variable hour lengths.
- Transportation (especially at speed) and broadcasting (radio, TV) depended on
  relatively accurate clocks.
- Development of cesium atomic clocks: https://www.nist.gov/pml/time-and-frequency-division/popular-links/walk-through-time/walk-through-time-atomic-age-time
  - NIST-F1 is accurate to 30 billionths of a second per year
- Difference between UTC and GMT https://www.nist.gov/pml/time-and-frequency-division/popular-links/walk-through-time/walk-through-time-world-time-scales


Compare style:

- We coordinate time to avoid train accidents.
- The railroad coordinates time to avoid accidents.

Principle: use clear subjects and verbs. Avoid "we (as a society, as engineers,
etc.)".

- I finished setting up the `ntp.conf` file, and I've started collecting stats.
- The `ntp.conf` file is now set up, and stats have started collecting.

Principle: if you're not a key player, let the objects occupy the subject.

Poor writing, in my opinion: https://guides.loc.gov/this-month-in-business-history/november/day-of-two-noons
Abusive use of the passive voice. Great bibliography, though.

TODO: Clock frobbing? Sometimes it is diagnostically interesting to perturb your clock to watch how ntpd responds and makes corrections. This option does that. https://docs.ntpsec.org/latest/ntpfrob.html 


- https://linux.die.net/sag/hw-sw-clocks.html
- https://wiki.debian.org/DateTime
- https://ntpsec.org
- https://wiki.archlinux.org/title/Systemd-timesyncd


