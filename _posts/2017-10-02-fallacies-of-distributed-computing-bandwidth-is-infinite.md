---
layout: post
title: Fallacies of distributed computing&#58; Bandwidth is infinite
---

> Ignorance of bandwidth limits on the part of traffic senders can result in bottlenecks over frequency-multiplexed media. <br>
> &mdash; L Peter Deutsch ([Source](https://en.wikipedia.org/wiki/Fallacies_of_distributed_computing))

It may seem obvious but how many developers do you know who wouldn’t give a second thought to the amount of bandwidth their systems use.

5 years ago you may have dug deep to get a laptop with sufficient resources to run a few virtual machines.
Year on year we’re seeing bigger and faster memory and bigger and faster disks at more affordable prices.

So why isn’t bandwidth keeping up with the trend? Bandwidth hasn’t grown nearly as quickly.

It’s incredibly common to see network architectures using 1 gigabit eithernet which seems plenty, right?

How many engineers really understand what their 1 gigabit network is really capable of?

Let’s start with the misnomer that 1 gigabit is anything close to 1 gigabyte.
It’s a common misinterpretation and the shared use of `giga` is almost certainly to blame. 

### In fact, 1 gigabit is equal to 125 megabytes. ###

Doesn’t sound quite as plentiful anymore does it?

Let’s see what happens when we introduce a protocol to our network. It would stand to reason that any layer of abstraction would eat into our bandwidth.

At TCP level can we expect to see headers wrap our packets.
These headers allow for reliable transportation of data across our of network.
Speaking of reliability, we should acknowledge [packet loss](https://en.wikipedia.org/wiki/Packet_loss) happens.
[Sliding window](https://en.wikipedia.org/wiki/Sliding_window_protocol) limits the rate at which data is exchanged between sender and receiver and [exponential back off](https://en.wikipedia.org/wiki/Exponential_backoff) are all working against you to limit the usefulness of your bandwidth.

The amount of useful remaining bandwidth is estimated to be around **40%**.
That's approximately **50 megabytes per second** of our plentiful 1 gigabit network.

You can imagine that once we begin to add HTTP, HTTPS or encryption into the mix we being to see our bandwidth go from 40% to 30% to 20% and that’s before we begin to talk about inefficient text based serialization such as XML and HTML.

We all want to build scalable systems; nobody wants to be victim to their own successes.
Let’s all acknowledge that the pipes won’t be getting bigger any time soon and begin having sensible design discussions about bandwidth.

Death to the monolith!

&mdash; Dan