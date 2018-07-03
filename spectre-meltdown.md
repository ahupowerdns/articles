---
title: "Spectre & Meltdown: tapping into the CPU's subconscious thoughts"
date: 2018-01-06T10:34:05+01:00
draft: false
---
Comments are very welcome on bert.hubert@powerdns.com or
[@PowerDNS_Bert](https://twitter.com/PowerDNS_Bert). *Update: several
constructive remarks have been used to improve this text. Thanks!*

In this post I will attempt to fully explain the Spectre and Meltdown
vulnerabilities in an accessible way.  I decided to write it up after I
realised it took me more than a day to figure it out, even though I've been
doing security related stuff on CPUs for 20 years.

You will find many explanations of Meltdown elsewhere.  This document is as
far as I know the only one so far that makes the hardest part of Spectre
accessible.

After reading the story below, the most excellent and detailed
[Google-provided
explanations](https://googleprojectzero.blogspot.nl/2018/01/reading-privileged-memory-with-side.html)
should start to make sense.  The
[Spectre](https://spectreattack.com/spectre.pdf) and
[Meltdown](https://meltdownattack.com/meltdown.pdf) papers are also great
sources of information. 

Nothing I write here is original.  The researchers involved deserve huge
credit for their work.  My explanation here is a small tribute to their
massive achievement.

Please contact me if you find any mistakes in this post, but do realise that
this post is a tradeoff between 'accessible' and 'comprehensive'.  In other
words, some shortcuts have been taken.

Out-of-order execution
----------------------
To a remarkable extent, computer programs are like kitchen recipes. The CPU
can then be seen as a machine that executes those recipes.

Over the years, it has proven hard to make the clock speeds of CPUs go up
any further.  A few gigahertz is the practical limit.  To improve speeds, we
can't make individual things go a lot faster, much like you can't whip cream
any faster by upping the voltage on your food processor.

When faced with a recipe like this:

1. Seed and dice a pumpkin, chop into cubes
2. Cook for 5 minutes in melted butter
3. Blend cubes in food processor
4. Put chicken in a [Dutch oven](https://en.wikipedia.org/wiki/Dutch_oven)  at 180 degrees for 3 hours
5. Make chicken broth
6. Add broth to blender until pumpkin soup reaches desired consistency

A wise cook immediately spots that the preparation of the chicken can fully
run in parallel with the pumpkin related work.  And in fact, we'll begin by
making the chicken & dice the pumpkin while the chicken is in the oven.

CPUs do similar things for us to improve processing speeds.  While on the
surface a CPU executes code sequentially, if it spots that there is work
'down the road' that could already be performed, it will do so in the
background and in parallel.  **However, and this is key, it will not tell us
about this**.

From the perspective of an executing computer program, what the CPU delivers
is like pure magic. All of a sudden there is a whole bit of code that takes
no further time to execute: the results fall right out of the sky. 

Compared to the recipe above, it is like after preparing the pumpkin.. 
suddenly a fully cooked chicken appears. It was prepared in the background,
and we didn't see it happening.

The CPU's subconscious work
---------------------------
As computer programs execute, CPUs continually race ahead to perform this
"speculative" or "out-of-order" execution".  Sometimes it works and it turns
out the CPU can meaningfully get ahead, but frequently this turns out not to
be the case.

Compared to the recipe above, let's say there was a step 'now drip some
pumpkin blend over the chicken', it would all have broken down: suddenly you
can't cook the chicken before the pumpkin is done. The speculation failed.

To make this not be a problem for a CPU, the promise is that all this
speculative execution leaves no trace.  The assumption is that if it failed,
it was like the attempt was never made.  All effects are rolled back.

**This assumption, which turns out to be false, is key to understanding
Meltdown/Spectre**.

The cache
---------
Regular RAM memory is a lot slower than a CPU, so it makes sense to have a
cache for frequently used memory. This is highly effective, to the point
that a modern CPU will actually not use memory directly at all. It will
always go through the cache. If a bit of memory was already found in the
cache, accessing it will go hugely faster than if that memory first had to
be retrieved from actual RAM.

But what about the promise that speculative execution left no trace? What if
some code that was executed already had to access memory, surely this would
end up in the cache? Isn't that a trace?

**Turns out that this is indeed the case** - the CPU has no alternative.  To
(speculatively) execute code that needs information from RAM, it has to go
through the cache.

**The upshot of this is that we have gained a facility to figure out what the
CPU has been doing behind our backs, things we were never supposed to see.**

Forbidden memory
----------------
An advertisement running on a webpage should not have access to the rest of
the computer. Otherwise a single rogue ad could sniff your WiFi password &
share it with the world. And beyond that, an ad may need access to the page
on which it is displayed, but it should not be able to look into other
browser tabs.

In short, we use programming and CPU features to restrict what data running code
can read, in order to protect our secrets.

**Key to both Meltdown and Spectre is that we trick the CPU into speculatively
executing code that does read forbidden memory**.

Meltdown
--------
The Meltdown bug is easiest to explain.  A computer program must have no
access to operating system (OS) memory.  In this OS memory reside WiFi keys,
passwords, and we don't want our web browser to be able to see (or sell)
those things.  To do so, the operating system instructs the CPU to forbid
access to kernel memory.  The CPU should then stop any program that attempts
to read such memory.

It turns out that the CPUs found in most computers do enforce that check.. 
but not during speculative execution.  They only perform the check before
the results of that access are released to the 'regular' program.  In other
words, the subconscious part of the CPU can see things we can't, but it will
never tell us.

But as shown above, speculative execution does end up putting things in the
cache. And we can measure if some memory is present in the cache or not.

An ad could exploit this by attempting to run the following program:

1. Draw ad on the screen
2. Read and write a ton of memory so the cache is full of other stuff
3. Read the first letter of the WiFi password from OS memory
4. If that first letter is an 'S', read the first pixel of the ad from memory

This program can not normally execute. Step 3 is forbidden by the CPU, which
is why normally an ad can run safely without stealing your passwords.
However, if the CPU has the Meltdown bug, it may speculatively execute steps
3 and 4 as well. It would not release the results to us however.

So this program fails to execute.. but now the ad runs a second program:

1. Start stopwatch
2. Read the first pixel of the ad
3. Calculate how long this took

If we now find that the read of the first pixel was super fast, we know that
pixel was in the cache. And the only way this happens is if the CPU
speculatively concluded the WiFi password started with an S.

Even though this example program only read one letter, it is obvious how
this technique could be extended.

As far as I can tell right now, of popular CPUs in computers, only those
from AMD appear not to be sensitive to Meltdown.  Some phones and tablets are
sensitive as well.  Meltdown is a clear mistake, but one made by many CPU
vendors.

AMD CPUs appear to do the wise thing and not continue speculative execution
once 'forbidden' memory has been touched. Many ARM chips do not speculate at
all.

The fix for Meltdown introduces overhead, but does work: make sure that
computer programs have no way to address kernel memory at all.  The overhead
comes from the switching that has to be performed any time a program
executes a system call to the operating system.

Further details can be found in the [Meltdown
paper](https://meltdownattack.com/meltdown.pdf).

Spectre
=======
Spectre is a related issue that is harder to exploit but builds on similar
techniques.  And it is far more pernicious.  As far as we can tell ALL
modern CPUs are affected.

The first variant is simplest. In the Meltdown example we modeled an
advertisement trying to read the WiFi password. However, not only don't we
want ads to be able to do that, we also don't want them to take a look at
other browsing tabs (which might have your banking details on them).

This protection is implemented in the browser, and it might look something
like this:

```
char accessMemory(position)
{
   if(position >= end)
       throw error();
   return memory[position];
}
```
In the 'if' statement, we make sure the ad can't access memory beyond the
end of its own memory. If it tries, that is an error.

However, remember that the CPU tries to speculatively go ahead. If we can
speculatively make it access memory[position], we could again leak data:

```
if(accessMemory(100000000)=='B')
	x=readPixel(1);
```

And then the first pixel would be in the cache, and like Meltdown above, we
could use a stopwatch to measure how long it takes to read the first pixel. 
If this was fast, we would have learned there was a 'B' at position
100000000.  This might be another tab in the browser with Banking details.

But how do we make this speculative execution happen? Clearly the first 'if'
statement protects us? Even speculative execution would stop there.

The trick is to make sure that the value of 'end' is NOT in the cache. The
CPU can then decide to speculatively already execute the rest of the code,
assuming that position will be < end.

With this trick, most embedded programming languages (as used in
advertisements, modules, plugins etc) can escape their secured
sandbox and see data they should not be able to see.

The good news is that such embedded environments can make it hard to perform
this trick, for example by removing sufficiently precise stopwatch
functionality, or adding special instructions to `accessMemory()` disabling
speculative execution.

The bad news is that as it stands, no CPU is able to change this behaviour.
Speculative execution is wired deeply into the fabric of modern processors
and will always leave traces. 

The most powerful part of Spectre
---------------------------------
This part builds on everything that was explained before, and is the hardest
bit to explain.

As described above, speculative execution reads ahead to already perform
work, in case we need the results later. The pumpkin soup recipe was fully
linear, and could be executed completely in parallel.

But let's say the recipe had an additional twist:

1. Cook the chicken for 3 hours in the Dutch oven
2. If the pumpkins cubes weren't very browned, leave chicken in oven a bit longer
3. Debone chicken, make broth etc.

To speculatively execute this, in step 2 we have to make an assumption about
the pumpkin cubes.  And we don't know that yet, so no parallel speedup for
us.  Bummer.

CPUs frequently have this problem, for example with statements like `if(pos
>= end)`. To enable speculative execution in the face of such unknowns, the
CPU keeps statistics. These statistics store where the `if` statement
went the last N times it was tried. **This allows speculative execution to
continue along the most likely path**.

The statistics are stored in the  Branch History Buffer (BHB).  As CPUs do
not have infinite memory, they can't store statistics for every `if`
statement out there.  What follows is rather amazing, and I still can't
quite believe it.

Since the BHB capacity is limited, it is not keyed to the full (absolute)
address of the `if` statement. Only a number of bits are used. Whenever a
CPU needs to know where an `if` statement jumped to, it consults the BHB
based on **part** of the address of that statement.

This might of course sometimes lead to a collision because many addresses in
memory will share a single position in the BHB.  CPU vendors are well aware
of this, but they have traditionally not worried about the problem.  The
reason for this is that the content of the BHB is only used for
**speculative** execution.

In other words, if there is a nonsense address in the BHB for a certain `if`
statement, some irrelevant code may be speculatively execute for a bit, code
that maybe belongs to a completely different computer program.  Might even
be random noise.

**Because of the assumption that speculative execution has no side effects and
is not visible, nobody cared**. You can see where this is going.

If you know the details of which address bits are used to consult the BHB,
you can craft collisions.  By polluting the BHB just right, you can make the
kernel or even the VM host speculatively execute some code.  The CPU will
quickly find out that that code is not actually needed, and not use the
result of the calculation.  But by that time, the damage has been done.

Because, as above, we can craft that speculatively executed code to move
certain things into the cache, which we can then later detect.  And voila,
we've been able to read memory not only outside our own process, but even
outside our own virtual machine.

The impact of this variant of Spectre is hard to overstate. Although it
requires a lot of preparation and knowledge, the power is unsurpassed. 

And as with the previous Spectre variant, this behaviour is wired deeply
into the core of CPUs. No quick fix will be forthcoming.

Instead, operating system vendors and compiler writers are currently working
furiously to insert countermeasures that disable branch prediction at
crucial points.

Full details can be found in the
[Spectre](https://spectreattack.com/spectre.pdf) paper.

Summarising
===========
Speculative execution has side-effects that include leaving stuff in the
cache that would otherwise not have been there. Speculative execution can
access forbidden (kernel) memory in some cases (Meltdown). In addition,
through Spectre, protective `if` statements may be temporarily ignored,
again leading to speculative access to forbidden memory.

Furthermore, by polluting the Branch History Buffer that is used in
speculative execution, a CPU can be fooled into speculatively executing
code, even in the VM host, that has observable cache side effects.

CPU, Operating System and compiler vendors are racing furiously to find
mitigations and workarounds with acceptable performance impact.

Further reading
===============
After this high level introduction, I can recommend the actual
[Spectre](https://spectreattack.com/spectre.pdf) and
[Meltdown](https://meltdownattack.com/meltdown.pdf) papers.

The full heroics of actually exploiting these vulnerabilities are described
in glorious detail on the [Google Project
Zero](https://googleprojectzero.blogspot.nl/2018/01/reading-privileged-memory-with-side.html) post.
