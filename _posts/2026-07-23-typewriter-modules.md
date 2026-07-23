---
title: Modularity and engineering in typewriters
layout: single
date: 2026-07-23
---

I have a small collection of typewriters that I started fiddling with during the
pandemic. Mostly I don't have to do very much to get them working again, but
occasionally there's a pesky problem or adjustment. I turn the machine upside
down and try to figure out why the ribbon holder isn't raising up high enough on
this Adler Tippa S, and I don't see anything. The thing is a mess of linkages,
screws, and posts. Even slowly pressing keys doesn't make it that much clearer
what's going on -- a dozen parts move in coordination.

A traditional manual typewriter has around 2,000 parts, which are used to
accomplish a complicated set of behaviors. It's a lot like software. And like a
large software project, it's helpful to start from the behaviors you want and
then work towards the implementation details.

# Functions of a typewriter
At its most fundamental, the duties of a typewriter are:

- Hold the paper to be typed on
- Allow the typist to imprint letters on the paper in series with correct
  spacing
- Hold consistent margins
- Advance consistently to the next line
- Allow for various convenient adjustments and features

Some of the implementation details have become standards, like doubling up
characters on each type bar to save space. Others are specific to the
manufacturer and model, like the escapements and their trips for advancing the
carriage by a consistent character width.

Every modern typewriter shares these features:

- A moving carriage attached to a spring-loaded drawband for advancing the
  characters on the paper.
- An escapement mechanism to advance the carriage consistently.
- A rubber platen and a set of rollers to hold the paper and provide a surface
  to smack characters into.
- A keyboard with doubled up characters (either lowercase/uppercase or a
  combination of number and symbol).
- An inked ribbon, which advances to provide a fresh surface for each letter.

In fact, most features of the typewriter have standard implementation patterns,
even though the details vary from the technician's standpoint. As in software
and architecture, the patterns repeat for economy, simplicity, and
servicibility. The user interface for all typewriters is largely consistent.

# Implementing a typewriter
A breakdown of features:

- Keystrokes
  - Smack platen with key face
  - Align all letters
  - Advance the carriage by one character
  - Raise ribbon based on color setting
  - Advance the ribbon
  - Halt keys at right margin stop
- Spacebar
  - Advance carriage
  - Leave ribbon, ribbon holder unchanged
- Backspace
  - Reverse of spacebar action
  - Restricted by margins
- Shift
  - Raise the carriage
  - Maintain all other settings
- Line feed
  - Roll the platen
  - Select line spacing
  - Allow free movement
- Paper holder
  - Allow easy paper feeding
  - Hold paper securely
  - Allow paper to be released
- Tabulator
  - Advance carriage to next tab stop
  - Set/clear tab stops
- Margins
  - Set, adjust margins
  - Margin release
  - Ring bell as right magin is approached
  - Halt carriage at margins
  - Halt keys at right margin
- Other
  - Color selector
  - Key tension adjustment

Many of these conflict with each other, such as advancing the ribbon for
keystrokes but leaving it unchanged for the spacebar. Others are biased, such as
halting keystrokes at the right margin but not at the left one. Inside each
typewriter, all of these functions are implemented through a complicated series
of linkages and levers. It's starting to occur to me that 2,000 pieces is even
economical.

The typewriter mechanic not only has to understand each function, but also the
traditional implementations of those functions, how they interact and which
tradeoffs each manufacturer made to keep the user happy. The manufacturers also
had to grapple with significant complexity, which they did using the same
techniques as software engineers: modularity, patterns, and when all else fails,
detailed documentation. Unfortunately, a lot of the service manuals are hard to
find now, and with them the "correct" procedures for servicing their machines.

The mechanic also works in an in-between space -- in the large and in the small,
typewriters are all the same. In the large, they all implement the same basic
features. In the small, they're all composed of levers, linkages, cams, springs,
and other traditional mechanical devices. It's the medium that bites you --
understanding how the designers made use of the mechanical "language" to
implement the standard interface.

As someone who came of age much after the time when software engineers had to
agonize over space and time restrictions, I find the whole problem of fitting
all this functionality into a desk or portable typewriter kind of fascinating.
The whole idea of mechanical engineering is so tangible compared to the
thought-stuff I work on. When you implement something well in software, it works
fine; when you do it in a typewriter, you get a satisfying 'clack' and you
produce words on a page. Something to be said for it.

If you're interested in seeing a traditional implementation, this video by
Animagraffs is pretty excellent:
[https://www.youtube.com/watch?v=yKpIwi1UUIk](https://www.youtube.com/watch?v=yKpIwi1UUIk)

