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
I always assumed I was using ntpd to keep time on my linux computer. And I was
sort of right.

According to the Debian Wiki, since Debian 12, the default NTP client is
`systemd-timesyncd`. It uses SNTP (Simple Network Time Protocol), which only
implements the client (no option to host a time server), and it sets the time
roughly by communicating with a single time server.

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



TODO: Clock frobbing? Sometimes it is diagnostically interesting to perturb your clock to watch how ntpd responds and makes corrections. This option does that. https://docs.ntpsec.org/latest/ntpfrob.html 


- https://linux.die.net/sag/hw-sw-clocks.html
- https://wiki.debian.org/DateTime
- https://ntpsec.org
- https://wiki.archlinux.org/title/Systemd-timesyncd


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

---

All this had me wondering, where does authority in timekeeping come from? Who
has the time? Fortunately, I'm only an hour's drive away from the [National
Clock and Watch Museum](https://museum.nawcc.org), and after a few dozen brief
hours of research, I'm still not satisfied.

What I have learned is this. We get our sense of time from the periodic
movements of the starry firmament -- the sun, the seasons, the stars. And our
bodies (along with most organisms on Earth) have built-in timers that encourage
us to do those activities that keep us alive. It's been a while since you ate,
the sun is down, which when you normally sleep, and so on. Biological clocks (as
anyone with a cat will tell you) can be extremely precise.

Further, one of the earliest forms of coordinated time has to be _yawning_. I
yawn, you yawn, and we all get tired together.

Religion further temporalized life with its emphasis on doing certain religious
acts multiple times a day. Some of the earliest interesting clock-like devices
we have are from monasteries that rang bells at specific times. (The work
_clock_ is derived from the French word for bell.)

Time used to be more organic than it is today. Hours were not equally sized, and
the day was not split equally into 24 parts. But somewhere, at some time, we
made an intuitive leap from continuous time devices like the clepsydra or the
procession of different stars and planets to discrete time -- time as ticks. In
_Revolution in Time_, David Landes considers this one of the great
methodological leaps in western civilization, coming seemingly out of no where.
It took eastern cultures another 500 years to begin using oscillating, periodic
timekeepers.

_Escapements_ convert potential energy (like a falling weight suspended by a
rope) into periodic motion (the ticking of a hand). There's no better
visualization of the development of the clock than Bartosz Ciechanowski's
[mechanical watch](https://ciechanow.ski/mechanical-watch/). And while these
escapements were crude in the beginning, it took only a few breakthroughs until
they were able to tell time within a few seconds per day.

---

I installed NTPSec on my Debian machine and left the configuration mostly as-is.

```
sudo apt-get update && sudo apt-get install ntpsec ntpsec-doc ntpsec-ntpviz
```

I made sure to enable statistics, because what I'm really after are
visualizations. I want to see the thing do stuff. This is done using `ntpviz`,
which is scantily documented (this was helpful but ancient:
https://blog.ntpsec.org/2016/12/19/ntpviz-intro.html). I found enough to get me
going.

There are a few things to visualize with NTP. Clocks are never perfectly in
sync (even UTC is calculated as an average), and the most important contributor
to incorrect timekeeping is a difference in oscillator frequencies. This is
called frequency _skew_. If the "correct" time is an oscillator at 1000 Hz, my
local computer clock might be more like 1001 or 999 Hz (or worse). So even if I
set my clock to the right time, in this fictinal scenario I would lose N seconds
every day (TODO: calculate).

Your skew might also vary over time, and this is called _drift_. NTP must
account for both of these.

For reasons I don't understand, frequency skew is usually measured in parts per
million, which is to say the number of periods fast or slow per million
oscillations. In the 1000 Hz example, 1001 Hz would have a skew of 1 part in
every thousand, or 1000 parts per million (ppm). 999 Hz has a skew of -1000 ppm.

But skew is also described in other ways. The most human-friendly (in my
opinion) is "seconds gained or lost per day", or year, or whatever. This gives
you the number in practical terms. It's a bit tricky to translate between them,
though, considering the gap between oscillation frequency and length of a day.

NTP corrects for skew (but not necessarily drift) as part of the protocol by
nudging the time and doing its best to predict changes. But skew is dynamic, and
it is affected by the quality of the hardware and things like changes in local
temperature (room temperature or computer temperature) and even humidity
(CLAIM). For ideal timekeeping, you'll want to keep your computer in a nice
climate-controlled room with excellent heat sinking.

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

How do you get 144 time zones? Any move along a line of latitude (ie east or
west) causes the sun's apparent apex to move. When the sun is highest in the sky
in Pennsylvania, it's already setting in Colorado. Your sundial would in that
case create _infinite_ time zones for each variation in longitude. The railroads
"solved" this by using a standard time for each major city they stopped in. You
got a sort of "solar mean time" for this stretch of railroad.

The railways kept different time, though, leading to accidents and fatalities
that finally motivated the US to move to _standard time_.

For those of you who've worked on large, coordinated projects, you might get
sweaty thinking about the logistics of suddenly changing the time, and you'd be
right -- there was a [day of two
noons](https://guides.loc.gov/this-month-in-business-history/november/day-of-two-noons)
on November 18, 1883. Incidentally, it wasn't until the 1918 Standard Time Act
that Congress actually acknowledged this.

---

After a day or two, I checked back in on my NTP stats to see what I'd collected.
For my distribution, the data collects in `/var/log/ntpsec/`. Running `ntpviz` o
this folder will generate an HTML report with all of the default data
visualizations.

```
nptviz -d /var/log/ntpsec/
open ntpgraphs/index.html
```



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

