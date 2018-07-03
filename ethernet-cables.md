---
title: "Some brief notes on making ethernet cables"
date: 2017-09-30T20:07:40+02:00
draft: false
---

# Some brief notes on making ethernet cables

*With thanks to the most excellent [NLNOG](https://www.nlnog.net/) for a lot
of mostly correct advice!*

Friends will tell you “crimping” ethernet cables is easy. Most of your friends
either have overly rosy memories, or have practiced on 200 cables before telling
you it is now easy. **It is not easy**.

This page is for if you have a few extra euros/dollars/coins to spare, and don’t
want to spend rare hours of concentrated effort to crimp an ethernet cable using
unfamiliar equipment and supplies. **I’ll also provide some moral support. It is
not just you, this stuff sucks.**

#### Rule 1: Try not to

If you can get away with it, do NOT crimp ethernet cables. Even professional
ones break, yours won’t be better. If at all possible, buy ready made cables.
Don’t crimp your own cables to save money if you value your time.

If you are still here, you likely need to run a cable through your house/office,
and you simply have to do that without connectors on there.

#### Rule 2: Get the good stuff

One of the easiest ways to mess this up is to reuse an old and bendy crimping
tool, or attempt to crimp the wrong kind of RJ45 plug on an incompatible cable.

<center>
![](https://cdn-images-1.medium.com/max/800/1*bJMgowfeIMnwunzbqcRJQw.png)
<span class="figcaption_hack">Just say no. All connectors on the right cost 3 euros, but you’ll need 7
attempts!</span>
</center>

The only thing lying around you can likely reuse is ethernet cable (Cat5/Cat6).
Disregard any (bendy) crimping tools or connectors you might have from previous
attempts! *Also, please stay away from
‘*[CCA’](http://www.belden.com/blog/datacenters/CCA-Cable-5-Reasons-To-Stay-Away.cfm)*
cables.*

#### Get the cable from A to B

This is usually the easy bit, except when it isn’t. Make sure you have something
like this “cable puller”.

<center>
![](https://cdn-images-1.medium.com/max/600/0*Byx0kc_AEL2ThVYv.jpg)
<br/>
<span class="figcaption_hack">“Trekveer” / cable puller</span>
</center>

If you are in luck, there will be an existing (guide) wire in your tube. Hook it
up **really** well to the cable puller, and then pull the cable puller through
the tubes using the original wire. **But do keep it thin and flexible, it has to
take corners!**

When the cable puller wire comes out on the other end, remove guide wire & hook
up your UTP cable to the cable puller. Use tape and all means available to make
a super solid connection that will NOT break within the tube, but again, do keep
it thin and flexible.

Now pull the cable puller and your UTP cable back. This may require far more
force than seems reasonable (it may help if someone else pushes the cable from
the other side while you pull).

#### Now the heretic bit

Longer ethernet cables are frequently [solid and not
stranded](http://www.scpcat5e.com/solid-vs-stranded-category-cables-ezp-80).
This means that the individual wires in there are actually solid, which in turn
means that most connectors you can crimp on there don’t work **that** well (or
at all!)

**So don’t use them! Also, forget about the crimping tool as well.**

My advice: use a large and ugly keystone connector. It is meant for solid or
stranded cables, and far easier to use:

<center>
![](https://cdn-images-1.medium.com/max/800/1*g6gUwCB0U9i1JKJD_25C9Q.png)
<br/>
“Tool-less keystone connector”, 3 euros 50
</center>

How this works is that you have a lot of room to put the individual wires in the
clearly marked holes. Typically they show two colour schemes called ‘A’ and ‘B’
and you need to pick one and use it on **both** ends. Put in cables, double
check if everything is right, close up module & done. *Note: don’t needlessly
untwist too much UTP, it will hurt signal quality.*

<center>
{{< figure src="https://cdn-images-1.medium.com/max/800/1*-338SefQS4bur-272Y_eyA.png" caption="Another keystone connector. Note the ‘A’ or ‘B’ wiring schemes" >}}
</center>

> Note: some of these connectors are truly tool-less, others tell you to use a
> ‘punchdown tool’ (3 euro 50). This does look like a useful thing to have and
allows you to buy cheaper keystone modules. Make sure you know what you bought
and if it requires a tool. **A screwdriver is not a punchdown tool (see Rule
2!).**

Of course the downside is that you now have a socket, and not a connector:

<center>
{{< figure src="https://cdn-images-1.medium.com/max/800/1*i1eHKJ_Jhy9MGtKW05owBw.png" caption="Keystone RJ25, 2 euro 50" >}}
</center>

So you’ll need to get patch cables to get to your actual devices.

<center>
![](https://cdn-images-1.medium.com/max/800/0*GzXcfwJe0RV55B_5.jpg)
[https://www.cableorganizer.com/learning-center/how-to/how-to-wire-keystone-jack.htm](https://www.cableorganizer.com/learning-center/how-to/how-to-wire-keystone-jack.htm)
</center>

#### Closing remarks

I just spent an epic two weeks learning about the wrong kind of connectors,
“easy” connectors that weren’t and that even the “right” connector isn’t that
great for solid cable. I then ordered the keystone stuff and it worked at the
first attempt.

I hope the above has been useful for you! And if your friends persist in telling
you crimping is so easy… **ask them to do it for you**!

