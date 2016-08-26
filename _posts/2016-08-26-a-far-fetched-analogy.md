---
layout: post
title: "A far fetched analogy"
description: ""
category: 
tags: []
---
{% include JB/setup %}
I was thinking in a cute analogy between the dichotomies mutability/immutability and intuition/conscious thinking.


## mutability/immutability

When you mutate your state, e.g. when you keep updating an object that holds any meaningful state; you need to “trust” that state. In the code, you will write logic that will "act" based on what is known in that instant, not on how it got there.

With immutability you *need* to write logic based on the past. There is not such thing as latest state, your code needs to handle all the information up to this instant.

I'm not sure if my claim about immutability is only true for my own experience. So let me clarify with an example.

Imagine that you want to write a system that schedule meetings. Imagine that some system can "magically" extract all relevant information from the emails, and thus a there is always a `List[Email]` associated with any given meeting.

To schedule a meeting you basically need a time and a place right? The negotiation will be mainly about those (there could be some back and forth regarding participants, but let's ignore that). 

### solution with mutability
We could update a `Meeting` object every time a participant proposes a time or a location. That object would be saved on a database. Somehow it will need to keep track who has accepted what times and locations.

Our system will need to "speak" -- i.e. propose times and locations to others, based on the latest state of the meeting. 

The biggest issue with that approach is that it needs to "assume" too much, e.g. :
1 - There is a time `Meeting` that no one has accepted.
2 - A participant sends an email proposing a location, should we propose the time? or did we already proposed the time and if we do it again we would be spamming the person?

### solution with immutability
Now consider that we don't care about the latest state of the meeting. We only care about the information that they sent and we sent. Every time the system needs to make a decision will be based on what we have said and what they have said, the above uncertainty instantly disappears.

## intuition/conscious thinking

Intuition is the stuff you "know", even if you don't remember you know them or you don't even realize you know them, but there they are. It's great when making snap decisions, you don't need to know, in that instant, _why_ are you making that decision. It is not great to analyze the world nor making really important decisions. Since it is basically a bunch of patterns your brain has seen, and they might look alike to a new situation, and if it doesn't completely work, your brain will fill the blanks for you, it'll will _guess_. And it will feel *right*.

If you try to use intuition for everything, you will be doing lots of wrong assumptions -- the earth is obviously flat, tourists are idiots, people with a different color are sub-humans, etc.

Concious thinking is about pondering context, you wouldn't conform with what you think you know about something, you will want to understand why things are the way they are, before committing to anything.

Obviously there is a point where you need to stop and trust your brain, but you can also make a conscious decision about that too. 

The point is, you don't want to be at the mercy of whatever your brain picked out there. Remember you cannot control what you learn. Learning is not a fully concious activity, as the ad/propaganda industry has proved.

## Duality

Hopefully at this point is obvious what I'm trying to say:

immutability <-> conscious thinking
mutability <-> intuition

It is far fetched, and it only makes sense under the lens of this piece. And the conclusion is hardly surprising:

You usually should prefer immutability/conscious thinking, since it will allow you to find a solution closer to your real problem -- i.e. less assumptions. But there are moments when you do want to use mutability/intuition. As far as I have seen, _in my life_, performance is the only valid reason for it.



