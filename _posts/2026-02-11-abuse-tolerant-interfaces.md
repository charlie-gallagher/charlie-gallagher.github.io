---
layout: single
title: "Abuse-tolerant interfaces"
date: 2026-02-11
---

> A common approach in the industry for forming a performance oriented SLA is to
> describe it using average, median and expected variance. At Amazon we have
> found that these metrics are not good enough if the goal is to build a system
> where **all** customers have a good experience, rather than just the majority.
>
> "Dynamo: Amazon's highly available key-value store," DeCandia, et al.

The Dynamo paper has me thinking about kinds of customer and service. Amazon is
a company on the offense, by which I mean that there is no sort of traffic they
want to turn away. They succeed when customers gleefully fill their carts and
hammer as many orders as possible through the checkout during the holidays.
Their only concern is keeping up.

The Dynamo paper goes on:

> For example if extensive personalization techniques are used then customers
> with longer histories require more processing which impacts performance at the
> high-end of the distribution. An SLA stated in terms of mean or median reponse
> times will not address the performance of this important customer segment.

Those customers with enthusiastic shopping patterns are exactly the type of
customer that Amazon wants to court, and that drives their metrics away from
averages and towards extremes at the 99.9th percentile. At my own job at IXIS,
we've created a BI and data socialization platform, and the Dynamo paper has
me thinking that we are on the opposite side of some customer relationship
spectrum. Our platform works best when people use it _reasonably_.  If a power
user queries only for multiple years of data, that strains our resources and has
no incremental benefit to our bottom line.

Every company gets pricing stress, but some companies like IXIS, Snowflake, and
OpenAI have to worry about whether their pricing is secure against unusually
power-hungry power users. And that sucks, for everyone. I want people to
power-use our product without feeling like we're against them. At the least the
fiddly money problems we have should be transparent to the user. Just imagine if
YouTube made you pay a small fee if you watched too many videos today to cover
their compute costs serving you those videos.

Here are a few pricing models in this space, with companies I think represent
the model well:

- **Customer pays for compute.** AI tokens, most AWS services, Snowflake.
- **Ads pay for customer.** YouTube, Spotify.
- **Special services pay for freeloaders.** The freemium model, usually mixed
  with ads.
- **Good behavior by default.** BitTorrent's bartering system.
- **Good behavior rewarded.** Reddit, Stack Overflow.
- **Hard resource limits.** Google drive, GMail.
- **Throttling.** AT&T[^1], Tinder.
- **Abuse-tolerant interface.** Adobe Analytics Workspace, coupons.

[^1]: Seller beware, AT&T was sued by the FTC in 2014 for throttling data speeds for "unlimited" plan users after they used a certain amount of data.

As an alternative to "customer pays for compute," I've been interested in
abuse-tolerant interfaces, which you could describe as, "It's not impossible for
users to cost us a lot of money, but they'll find it's inconvenient to do so."
Coupons represent this par excellence. I mean physical, cut-em-out coupons.
They're abuse-tolerant because while it's possible to cut big stacks of coupons,
most people don't. There was that whole Extreme Couponing TV show about lengths
to which people went to clip the coupons. We're talking _days_ of labor between
collecting books, snipping, and organizing. But the savings were huge.

Coupons work in spite of their inconvenience. When you take the trouble, it
feels like a _steal_. They're a great time.

A few digital companies have figured out how to work this model into their
products. Adobe Analytics Workspace is one of those products in my opinion --
and if you aren't familiar, it's a BI tool for analyzing Adobe Analytics data.
The interface is composed drag-and-drop components like metrics, dimensions, and
segments. You build visualizations ranging from simple and customizable (tables)
to prefab (various flow charts and funnels), and they never limit you. You can
theoretically ask for millions of rows of data and you won't get rate limited.
Instead:

- The data is paged in on-demand.
- The interface is much friendlier to simple visualizations than monstrously
  large tables.
- Complex, nested breakdown tables must be created incrementally, which limits
  how much data you could sanely fit into a single table.

Now don't confuse me for Adobe Analytics' biggest fan or anything, but I've
never felt limited by the interface, although we've certainly pushed it to some
limits.

I think these sorts of abuse-tolerant interfaces are subtle and difficult to
execute well. But when you get it right, it's great for both the business and
the customer. Food for thought for those product owners out there who are
considering a "compute per request" pricing model.

