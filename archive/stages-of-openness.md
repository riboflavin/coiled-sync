---
title: Seven Stages of Open Software
description: This post lays out the different stages of openness in Open Source Software (OSS) and the benefits and costs of each.
blogpost: true
date: 
author: 
---

# Seven Stages of Open Software

This post lays out the different stages of openness in Open Source Software (OSS) and the benefits and costs of each.

### Motivation

"Open Source Software" is a hot term today.

As a result, people are reasonably encouraged to open up their software. This is great, but means that the term "open source" can get a bit confusing. Is Linux as open as TensorFlow? How about my personal project? Is that the same?

To help give depth to this topic, this post structures opening software into a sequence of stages of openness.

1. **Publicly visible source code:** *We uploaded our code to GitHub*
2. **Licensed for reuse:** *And let people use it for free*
3. **Accepting contributions:** *And if they submit a patch, we'll take the time to look at it, and work with them to merge it in*
4. **Open development:** *And when we work we'll make sure that all of our communication happens in the open as well, so that others can see what we're doing and why*
5. **Open decision making:** *And that communication will be open to the public, so that everyone can weigh in, vote, and determine what happens to the project*
6. **Multi-institution engagement:** *So much so that no single institution or individual has control over the project*
7. **Retirement:** *So now we can retire, and know that the software will live on forever*

To be clear, I'm not advocating that going deeper into this hierarchy is a good thing. Often it's more productive to stop somewhere around 3 to 5 (this is what most companies do). There are costs and benefits to every layer. We'll now go into each layer more deeply.

## 1 - Publicly visible source code

Technically all that "open sourcing" your code means is that you've made the source code publicly visible. This might be throwing a tarball up on some website, but today it mostly means pushing a repository to GitHub.

**Benefits**

This is great because your code is now auditable. For example if your code handles personal data or communications people can see what is happening internally to gain confidence that everything you're doing is secure and ethical. Or if your code runs some complex mathematical algorithm your users can inspect inside your black box and see why things behave the way that they do.

Opening up your code creates trust among your users.

**Costs**

But of course if you are trying to make money on this software then your competition can now gain insight.

## 2 - Licensed for reuse

Publishing your code doesn't actually legally enable anyone to use it. Code is still legally protected by copyright unless the author provides a license. For proprietary software you purchase this license. Often for open source software a license is distributed with the software which lays out the ways in which anyone can use it without ever asking you for permission.

**Benefits**

Yay, free things

**Costs**

You no longer control this software. If someone makes a bunch of money off it it then you can't complain. If someone uses it to build bombs then you also can't complain.

## 3 - Accepting Contributions

So you get an e-mail, and someone who you've never met has found and fixed a bug for you. Great, right?

**Benefits**

People do work for you for free. Assuming that you depend on this software this is great, you're getting free engineering time and a new perspective.

**Costs**

As this happens more and more it becomes a major time sink. You now have to educate these bug fixers, integrate all of their work, and make sure that things stay on the right track. Often you will have to reject contributions and handle the fallout of that.

These activities require a level of social engineering that you may not be familiar with or want.

## 4 - Open development

You spend enough time educating the people contributing to your code that you realize that it would be better if your internal team started having all of their conversations in the open as well. You push them away from in-person conversations (this is hard), out of the company Slack (this is harder), and force everyone to have all of your conversations in the open on public and searchable forums like GitHub or mailing lists.

**Benefits**

Yay transparency!

The community outside of your organization now has much more insight on why certain choices have been made. They really appreciate this and it builds a lot of trust around your product. Additionally, this attracts developers of a higher calibre to the project because they're now able to operate with the full context.

**Costs**

This is really hard to do. It's also pretty inefficient in the short term, especially if you already have good internal communication practices. Fully open and public communication is really time consuming to do correctly.

You'll also be fighting an uphill battle to get your teammates to move off of internal communication, particularly if the rest of the company is there.

## 5 - Open decision making

Now that you've opened development, you have attracted some developers who fully understand the project, and have thoughts on where it should go. You listen to them and make sure that the aggregate needs are met, and not just the needs of your company. Sometimes you disagree with them, and sometimes they win these disagreements. This ends up being mostly a good thing.

You give them commit rights, the ability to push releases, and generally the ability to change things that you might not agree with.

**Benefits**

You get additional developer power and perspective at a completely different level. This isn't just the occasional bugfix, this is people pouring their work-life into your project, and using it to do things that you never thought possible.

Other people in the community see that, and stop seeing your project as just a bit of corporate software, but as something that might be infrastructure some day.

**Costs**

You've given up a critical amount of control. Certainly, you still have the ability to make changes and make sure that the software meets your needs, but now you may have to negotiate from time to time.

## 6 - Multi-institution engagement

Those core maintainers grow to the point where they dwarf your original team of contributors. There are now multiple companies/universities/institutions that build and maintain this software. You've given up the majority of control in order to enable a large and diverse workforce.

**Benefits**

This thing has grown in maturity and scale beyond what you could have predicted

**Costs**

It may not grow in the way that you want. You no longer have control

## 7 - Retirement

You can die now. Your software will survive without you :)

## Conclusion

I wrote this in response to a variety of people saying "Oh, we open sourced our software"