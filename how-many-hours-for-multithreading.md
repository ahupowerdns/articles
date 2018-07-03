---
title: "How Many Hours for Multithreading the Server?"
date: 2014-09-02T20:25:58+02:00
draft: false
---

### How many hours for multithreading the server? Or: dealing with overly detailed project planning

Many developers, me included, dread that moment. Someone sits down with you and wants to know how many "hours" each step of the project will take.  And of course the thing is, if you are doing something that has been done many times before, you might be able to provide a detailed estimate. People that build houses work like this, but they still often get it wrong. And typically, what we are doing is new - otherwise people'd be using an off the shelf solution!  

Even very good developers struggle to estimate how long individual steps of a project may take, even if over the years they have developed the ability to give a decent **\*whole*** project estimation.  This post is intended for those developers that CAN commit to a final deadline, but balk at the "spreadsheet hours exercise".  

I'm not knocking serious project management - you NEED to know where a project is, you NEED to keep track of dependencies.  Good project managers (they know who they are), are your partners in delivering great results.  This post is not about them, however.  Back to our software developer getting asked for estimates rounded to the closest 15 minutes.  

Everything inside us screams to tell this fool of a project manager "I'll let you know when it is done!". This however runs counter to all their beliefs, so soon your project is split up in a hundred small steps with innocuous names like 'make server multithreaded', 'add coherent content cache', 'implement localization'.  

And now this guy is sitting next to you and asking how much hours the "implementation of ACLs" will take.  "Come on, we are both professionals, I don't care what the number is, but you must give me an estimate!"  

Engineers that we are, we take a copy out of Scotty's playbook:  

> [![](https://lh3.googleusercontent.com/proxy/DBPHci5h0S_XhHajWhWOwEX9tRszqhNEka9L3Za7BOcUUFwq2_XodGvv0FIanCmTDw_GTgptosrVQ5Ddz0AwP-jLHvnpTwJJBSxNzadfPP5bjzFouY4wPY4k8ykPPmLimm8TLkyMzsY37hbYGBTkOWIpEnAoMxchMNrcs13ZfJoIjN6K6ZZfBauy=s0-d)](http://imgc.allpostersimages.com/images/P-473-488-90/61/6189/3J41100Z/posters/star-trek-the-original-series-scotty.jpg)"**Geordi La Forge**: Yeah, well, I told the Captain I'd have this analysis done in an hour.  
> **Scotty**: How long will it really take?  
> **Geordi La Forge**: An hour!  
> **Scotty**: Oh, you didn't tell him how long it would *really* take, did ya?  
> **Geordi La Forge**: Well, of course I did.  
> **Scotty**: Oh, laddie. You've got a lot to learn if you want people to think of you as a miracle worker!"

> [http://www.youtube.com/watch?v=8xRqXYsksFg](https://www.youtube.com/watch?v=8xRqXYsksFg)  

So we provide a massively padded estimate. 160 hours for implementing the ACLs. Dilbert also has some engineering overestimation guidance here: http://dilbert.com/strips/comic/1993-05-02/  

Soon the project guy adds up all our numbers and comes up with 2 years of work.  'But you told us it could be done in 3 months!', and you probably did.  So this doesn't work, and we end up with a compacted and painful schedule that is incredibly detailed and bogus. It rots the soul.  

Sound familiar? While we all know doing project management as outlined above is bunk (even though [Agile](http://agilemanifesto.org/) is not all that it is cracked up to be, it is massively more right than "waterfall"), you still get these people asking you for detailed estimates.  

And everything screams in us to tell the guy he's a fool, that things don't work that way, but it still doesn't help.  Anger management is a challenge here.  

This stuff has been holding me back for years. A [fellow developer](http://twitter.com/hacktobeer) I spoke with last week shares one trick I employ to avoid this mess: finish the project well BEFORE someone with a spreadsheet shows up to ask how long it will take! However, this scales badly, and doesn't allow you to charge serious money for your work ('he finished it before we started!').  

Then, I saw The Light. It might not do me any commercial good to share this trick, but I think it is too good to keep to myself.  

Here goes. WHY is someone asking you for such detailed estimates? Do they really care if it takes you 1 hour to do the ACLs and 120 hours to do the multi-threaded architecture or the other way around? Often the person asking you for estimates doesn't even know what these things are!  

No, a major reason why they are asking this stuff is to give you lots of rope to hang yourself with.  For them, it is a big covering of asses exercise.  Because if you go faster than you scheduled, it is your issue ('this guy has been overbilling us!'). If you go slower, it is DEFINITELY your issue.  If part of the project was easier, but another part was harder, it is still your problem ('he missed the deadline for the layout demo but he mumbled something about the ACLs being ready!').  

**Another major reason is that by having YOU be very detailed, it saves THEM from having to make any tough choices**. You provide loads of detail, they only have to watch if you are sticking to your prediction. And since you most likely won't, they don't have to live up to THEIR commitments. And for  many large organizations, this is key: **they have a far harder time committing to things than you!**   

A final very important reason is that asking for so much detail is a sign of insecurity. They don't know how you do your magic, they hate to rely on it, so they try to quantify it all. This gives folks a semblance of control over your work. And who can blame them for wanting that?  

Summarizing, they have very good reasons to force you to make detailed estimates that you can't possibly deliver on exactly. This works for them.  

Here's what's been working for me lately. If people show up wanting to do project planning, there is common ground: In what order will things happen, and where do we need the other party to deliver something? Work that out together. This builds trust and understanding.  

The customer (or whoever needs your work) must spend time explaining what they want.  (In the past, we might've said that they would need to provide detailed specifications and requirements, but the dark secret is, no customer that NEEDS you is in a position to provide specifications that are good enough. If they could do that, they probably could finish the project themselves!  So you need serious time from them to together pin down what they want).  

Once you are working, after a while you'll need input, logos and icons from their marketing (for example), and when integrated, an ok on layout, usability etc.  They have to commit people to sign off on that.  Similarly for integration with their database etc.  And eventually, the time comes for testing and rollout.  Typically there are many 'wait points' where 'now the other party has to do something'.  

So the common ground first consists of identifying these 'wait points'. And the very first wait point is at the start of the project: provide lots of time with the people that can tell us what they want!  (or, if they think they can do it, provide the very detailed specifications and requirements).  

**<u>And this is where you turn the tables on the people with the spreadsheets.</u>**  

Force THEM to commit to specific times and deliverables! If there are unanswered questions on the project (operating system?  virtualized or not?  database to use?), block on that.  "I can't plan unless I know what the work is!". List ALL the things you need to know.  

If they can't answer that on the spot (and a project manager typically won't be able to), ask them for an exact date when they CAN provide the answers.  

**Turn those tables!**  

Quite quickly you'll find that the customer has as much problems as you do with committing exactly to when they'll have something done!  "The children of the cobbler go bare feet".  

Then move on to the other 'hard wait points' where you need them, and see if they can commit to those. Can marketing deliver content 1st of February, can we schedule an ok of the UI on the 25th of February, 9AM?  And a go-live meeting on April 2nd?  Typically, you'll find that this is hard for them to do.  

If you've done your job right, after a while it will be well established and agreed that committing to specific results at specific times is very hard. Frustration is in the air.  

And now you whip out your gift - under these tired and battered circumstances, allow that it doesn't actually matter that much to you when the UI decision is "shall we put it somewhere in week 12 or 13? Let me know when you pin it down".  

If done well, this "gift" will be accepted with a lot of enthusiasm, and if anyone asks how long the ACLs will take to implement, ask when they'll let you know which database they'll be using. This restores the balance. The upshot is that you now only have a few deadlines left, the moments where THEY have to do something with your project, you have to be ready for them.  

Of course, in reality things will rarely play out exactly as above, but the take home message is: **turn the tables on folks asking you for detailed estimates you can't provide. Because in the end, most large organizations have a far harder time committing to things than you do**.  

Don't end up supplying padded numbers, just make sure you are asking as much detail from them as they from you. This will level the playing field, allowing you to 'sell' your flexibility in return for them getting off your  back in the meantime!  

Finally, I'd like to reiterate that the story above only applies to serious developers that ARE able to commit to actual deadlines like 'UI demo'.  If you have a hard time divining the size of a project, turning the tables won't help you! Work on that first - or get your customer to commit to Agile (but not the 'half-assed Agile' as found on http://www.halfarsedagilemanifesto.org/ )  

Also, from what I hear of serious project managers, they have little love already for spreadsheets full of micro-deadlines. A good project manager can however be your partner in making sure both you and your customer meet the expectations in delivering a working product!  

Good luck, and let me know if this works for you!