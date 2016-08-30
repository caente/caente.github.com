---
layout: post
title: "A far fetched analogy"
description: ""
category: 
tags: []
---
{% include JB/setup %}
I was thinking in a cute analogy between the dichotomies of mutability/immutability, and intuition/conscious thinking.

## mutability/immutability

When you mutate state, e.g. when you keep updating an object that holds any meaningful state; you need to “trust” that state. You will write logic that will "act" based on what it's known in that instant, not on how it got there.

With immutability you *need* to write logic based on the past. There is no such thing as latest state, your code needs to handle all the information up to this instant.

I'm not sure if my take about immutability only makes sense for the use cases I'm dealing with these days. So let me clarify with an example.

Imagine that you want to write a system that schedule meetings. Imagine that it is possible to "magically" extract all relevant information from the emails, and thus a there is always a `List[Email]` associated with any given meeting, which holds all the relevant data people sent in a structured form.

To schedule a meeting you basically need a time and a place right? The negotiation will be mainly about those (there could be some back and forth regarding participants, but let's ignore that). 

#### solution with mutability
We could update a `Meeting` object every time a participant proposes a time or a location. That object would be saved on a database. Somehow it will need to keep track who has accepted what times and locations.

Our system will need to "speak" -- i.e. propose times and locations to others, based on the latest state of the meeting. 

The biggest issue with that approach is that it needs to "assume" too much, e.g. :
1 - There is a time `Meeting` that no one has accepted.
2 - A participant sends an email proposing a location, should we propose the time? Or did we already proposed the time and if we do it again we would be spamming the person?

#### solution with immutability
Now consider that we don't care about the latest state of the meeting. We only care about the information that they sent and we sent. Every time the system needs to make a decision will be based on what we have said and what they have said, the above uncertainty instantly disappears.

## intuition/conscious thinking

Intuition is the stuff you "know", even if you don't remember you do or you don't even realize you know them, but there they are. It's great when making snap decisions, since you don't need to know, in that instant, _why_ are you making that decision. 

Intuition is not great to analyze the world nor for making decisions if you have the time to think. It's basically a bunch of patterns your brain has seen, and they might look alike to a new situation, and if it doesn't completely work, your brain will fill the blanks for you, it'll will _guess_. And it will feel *right*. In other words you are using the latest state of the `Meeting`, you are hoping that your assumptions are enough, that you are not facing an "edge case".

If you try to use intuition for everything, you will be doing lots of wrong assumptions -- the earth is obviously flat, tourists are idiots, people with a different color are sub-humans, etc.

Conscious thinking is about pondering context, you wouldn't accept what you think you know something, you will want to understand why things are the way they are, before committing to anything.

Obviously there is a point where you need to stop and trust your brain, but you can also make a conscious decision about that too. 

The point is, you don't want to be at the mercy of whatever your brain picked up without asking you. Remember you cannot control what you learn. Learning is not a fully conscious activity, as the ad/propaganda industry has proved.

## Duality

Hopefully at this point is obvious what I'm trying to say:

immutability <-> conscious thinking

mutability <-> intuition

It is far fetched, and it only makes sense under the lens of this piece. And the conclusion is hardly surprising:

You usually should prefer immutability/conscious thinking, since it will allow you to find a solution closer to your real problem -- i.e. less assumptions. But there are moments when you do want to use mutability/intuition. As far as I have seen, _in my life_, performance is the only valid reason for it.


