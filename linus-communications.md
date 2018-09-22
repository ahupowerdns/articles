---
title: "On Linus Torvalds, technical & corporate communications"
date: 2018-09-18T10:24:42+02:00
draft: false
---

Linus Torvalds has long been one of my heroes.  The invention of Linux & the
subsequent development of Git were technical and organizational miracles. 
You could fill a book simply by quoting examples of Linus dissecting
technical problems to their components and making it obvious what should
happen.

However, over the past decade, Linus’ communication style has degraded from
‘[Finnish style
robust](https://www.linkedin.com/pulse/management-perkele-just-do-ashley-cooper)’ to needlessly hurtful screeds, tearing into people
who did not deserve that.  For many of us this was painful to see.  “Why,
Linus, why?”

It now appears that several unnamed community members have breached Linus’
emotional hull and [caused him to see the
light](https://lkml.org/lkml/2018/9/16/167).  This is no mean feat, and
let us all hope Linus comes out as a better person.  We could all use his
insights, especially if delivered in a way that does not hurt.

In [expressing my
joy](https://twitter.com/PowerDNS_Bert/status/1041424097945837574) at the news of Linus seeking help quite a number of
people opined that we should not be “snowflakes”, and that this “Finnish
army style communications” were what made Linux great.  One of the 
[printable responses](https://lkml.org/lkml/2018/9/16/198):

> “No.  Just no.  You're so successful because you're one of few people who don't waste time beating around the bush.  You call a spade a spade instead of polite "professional" bullshit”

And I think this touches on some of the confusion among people that support Linus’ historical combative communication style. They conflate swearing at people with being honest or “straightforward”. And I can see where their confusion is coming from.

Three communication styles
--------------------------

> “Hey, we have this moveFile(Fileptr) function, but it needs a flag to not
> move symlinks.  I didn’t want to change the API, but I note that all
> pointers are divisible by 2.  So I propose we use
> ‘moveFile((Fileptr)((char*)f +1))’ to indicate that if the file is a symlink,
> it should not be moved.  moveFile can then & the pointer to get rid of the
> 1”.

If you are a C or C++ programmer of any merit, your brain should explode at this suggestion. Here is the “Linus Torvalds” response to this suggestion:

> “What the f*ck is wrong with you? Your employer has been releasing shit products for years, I wonder if this is how that happened. This is like the worst possible API abuse ever and the world will burn down because of this crap. Go away and don’t waste my time with this f*cked up shit”.

First the positive.  This response is actually what everybody is thinking. 
It is the unvarnished truth.  But that is the only thing this reply has
going for it.  The author may have had many reasons for proposing this
excremental idea and we may not have been aware of all the constraints they
were working under.  We can safely assume that this person will now become
an ex-contributor to our project though.

Contrast this with the ‘modern corporate HR-compliant response’. Bear with me:

> “Hi John,
> 
> I hope this email finds you well!
>
> Thanks for your suggestion. moveFile() is indeed an important part of our project, and I’m happy to see that you want to improve it. I’ve thought about your suggestion, which is clever in its own way, but I do wonder why you went down that avenue.
>
> The new calling syntax is somewhat unusual (at least to me, perhaps it is idiomatic in other places?) and may lead to surprises. I’m also not entirely sure if we might not one day get unaligned pointers on some platforms. We might get hard to debug problems from that, but I’m not sure, what do you think?
>
> Finally, I wonder how hard it would be to simply extend the moveFile API with a flag ‘noSymlink’, would that be too difficult? It may be there are reasons I’m not aware of, but from where I am sitting, it looks like we could just add that and recompile, but I could be wrong.
>
> Please let me know your thoughts and we can discuss.
>
> --  
> 🖨️ Please think about the environment before printing this email!”

Wow that was quite a read.  Not only did it take a long time to write, it is
also a complicated thing to read.  The hidden meaning is “no way we are
going to do this”, but it spends five paragraphs saying that.  You could
definitely read this reply wrong and actually go discuss the merits of your
odd-pointer idea.  I mean, it said this technique was “clever in its own
way”!  Quite some time has already been wasted on this email and it is
pretty likely far more will be wasted in further discussion.

This communication style is almost mandatory in many megacorporations, and it does kill technical progress. 

Here’s the “hey we’re all engineers here”-style response:

> “Um well - we could solve it this way but it looks fragile and not very obvious to the caller. Is there a good reason we can’t simply add a flag to moveFile?”

In my world, this answer is near perfect for a team that already knows each
other.  It communicates there is a problem with the suggestion, but leaves
open that there might be a good reason for doing it like this.  This took 20
seconds to write, and for a typical technical reader, there is no problem
interpreting this response.

Note that in many corporate environments, this email message is already
worryingly direct and might upset some non-technical people.  You would not
do this in the marketing department and have a decent career prospective,
for example. 

If a team does not yet have experience communicating so compactly, slightly more words may be required, or even a private reply that gives some more context that we appreciate the effort, but that technically speaking this won’t fly.

The problem
-----------

The problem now is that Linus suggest he wants to move to the “hey we’re all
engineers” style of communicating, still making points concisely, but now
without swearing or intimidating anyone.  However, many people have
conflated “not insulting people” with mandatory pages long beating around
the bush corporate drivel.  And we definitely don’t need that.

I’m charitably going to assume that the people supporting Linus’ legacy
communication style are actually worried we’ll soon all have to couch our
words in layer upon layer of corporate drivel.

But the good news is that we don’t have to.  It is entirely possible to
communicate concisely and directly without insulting people or making them
angry, and I recommend that any engineering organization strives to talk to
each other that way.

And if you have any thoughts, please [let](mailto:bert.hubert@powerdns.com)
[me](https://twitter.com/PowerDNS_Bert) know :-)

Note: my earlier post ‘[Talking to technical people](https://ds9a.nl/articles/posts/talking-to-technical-people/)’ 
discusses some similar themes.


<script>
/* dear reader - this is not to track you, but I need some metrics to know
 * how many people read my posts. What you are seeing is a small check to
 * determine if you are an actual visitor or a crawler. 
*/
function realityCheck()
{
	var num = 1*2*3*4*5*6*7*8*9*10;
	num = Math.sqrt(num);
	fetch('forreal.json?'+num+'&'+Math.random());
}
setTimeout(realityCheck, 1000 * 10);
</script>

