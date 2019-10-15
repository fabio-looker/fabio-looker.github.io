---
layout: post
title: LAMS Customization Updates
categories: Data
---

I'm excited to share some big updates that I've made to our open-source LookML linter, LAMS! In one line - LAMS now supports way more customization!

This article will dive into the why and how of the update. If you just want to see the updated customization docs, go [here](https://looker-open-source.github.io/look-at-me-sideways/customizing-lams). 

Also, if you'll be at JOIN, why not attend my presentation with Carl Anderson of WW about [Extending LookML Development](https://events.bizzabo.com/215120/agenda/session/139043)? :)

# How come?

Before we jump into it, some quick background. LAMS, or Look At Me Sideways, is an open-source LookML style guide and linter that I co-author. If you want to learn the basics, check out [this article](https://discourse.looker.com/t/introducing-lams-a-lookml-style-guide-and-linter/10603).

As it pertains to today's update, however, it's worth noting that LAMS was originally ONLY a style guide, and then had a linter built around it. What this means is that from day 1, the linter was super prescriptive and opinionated, telling everyone what they should be doing, walking around like it owned the place... :eye_roll:

In all seriousness, that pretty much was LAMS' original goal: To *prevent* problems by helping LookML developers do things a different way. A way that was based on our experience in the Customer Success Engineering team, and that would not be immediately (or ever) obvious to new LookML developers.

At the same time though, I saw that a couple people opted to write their own linters because they were looking for something different. They didn't want to adopt our best practice, nor did they want to make their own statement on what best practice *is* in any sort of broad cross-organization way... rather, they just wanted to express lightweight rules, some of which are relatively organization-specific. For example, fields with a "value_format: usd" should have a label ending in "(USD)".

# Ok, so what?

Two is a pattern, so I figured there must be more people with the need to express simple, quick, business-specific LookML rules. And, I saw an opportunity for this need to be met within the framework that LAMS is already providing. It just needed a bit more... finesse.

## First, opting out of prescribed rules

Although it has been possible to opt out of rules from day 1, it wasn't particularly clear in the docs, which I have learned through speaking with several prospective/actual LAMS users. So, I've added a new section in the rule customization docs that talks about this in more detail.

## Second, the legacy Javascript approach

Also from day 1, we offered the ability to specify custom rules via Javascript. It was a great feature for telling an extensibility story, but it had some pain points:

1. The Javascript had to be hosted externally. Not only was this inconvenient, but from a governance standpoint, it would be best if the rules were in the same git project as the rest of the model.
2. The evaluation of the Javascript was done by the same runtime that was running LAMS itself, which put custom rules in a murky security context[1]. Essentially, using the feature meant anyone with access to the LookML project could also run code on your CI server. 
3. Finally, and perhaps most importantly, the custom rules were not pleasant to write. You were expected to export a function which would be fed the entire LookML project, so you would inevitably have to have 30 lines of boilerplate JS for iterating through the project just to apply the simplest possible of rules.

It may not come as a surprise, but I don't think any customers have actually used custom JS rules. I am glad we included them, but felt no guilt marking them as a legacy feature (which is just a console message if you do use them). And with that, we get to the biggest part of today's article...

## Finally, the new in-project expression-based rules

So, I set out to build a new custom rules system. The first thing I did was to borrow some inspiration from [Carl Anderson's Linter](https://github.com/ww-tech/lookml-tools/blob/master/README_DEVELOPER.md) and his approach to custom rules, which by declaring different rule "types" greatly reduces boilerplate. Taking it to a bit more abstract level, I decided that "dimension-level", "explore-level", etc. were all just matching a shape of LookML construct, so instead of enumerating a few specific types, I would leverage a general-purpose data-shape matching solution - JSONpath.

Now, if you want to match dimensions, rather than write 12 lines of boilerplate javascript loops, you just declare your rule as `match: "dimension.*"`. Easy! Even better, not only did this hit on pain point #3, unwieldy rules, but also on pain point #1, because if rules are more succint, they can more readily be put inline in your LookML project.

The one missing piece of the puzzle was how to specify the logic of the rule itself. To start, I did try to see if someone had recently built an _actually_ safe approach to evaluating untrusted Javascript. This is something I end up wanting every couple of years, and invariably the answer is no. This time around was no different[2].

However, what I eventually found and was satisfied with was [Liyad](https://github.com/shellyln/liyad). It's a bit of a left-field choice, so I'll go ahead and acknowledge some downsides before going into why I ultimately picked it:

### Cons:

- It seems to be super obscure, with only 10 github stars
- It has a high bus-factor risk, with only 1 contributor
- It is not yet v1
- It uses an unfamiliar syntax (to most), being based on LISP
	
### Pros:

- It is theoretically capable of being sandboxed, which is not something that could be said of any current JS-expression library
- Separation of syntax and semantics leads to flexbility down the line
	- Since it is a LISP-based language, there is a clear separation of syntax and semantics. This is a huge bonus for handling any security issues that come up in the future since either Liyad or LAMS can operate on the semantics of the language without having to worry about how this affects the syntax.
	- As an example of the above, I submitted an issue to the maintainer because their method for property access allowed expressions to access some unsafe methods on objects' prototypes. The fix came through very quickly, with very minimal code changes, and with very targetted protections, still allowing a lot of intended JS prototype functionality. 
	- In addition, we can extend the language with custom written functions without changing the syntax.
- While it is unfamiliar to most, certainly some people will be familiar with a LISP-based language
- It handled lots of my strawman rules well enough, including boolean logic, string matching, and sequences of steps/assignment
- It is lightweight, having 0 dependencies and being <100k itself

# See the docs, try it out!

If you'd like to try it out, there are examples to try out in the new [customizing LAMS docs](https://looker-open-source.github.io/look-at-me-sideways/customizing-lams).

# Bonus: What's next?

Building the new expression-based custom rules was a lot of fun, since it involved researching lots of possible languages/approaches. However, in the end, the implementation was pretty quick. That said, there are lots of opportunities of where to go next with it:

- Web-based sandbox to try out custom rules, which would let you to enter some LookML, a JSON Path, and a Liyad expression and see the output. This is definitely doable since everything is already in Javascript, just a matter of setting up some build pipelines since modules intended for Node usually don't export themselves in the nicest way for browsers.

- Better generalization of error levels, logging, & reporting, so you can get a more wholistic picture of your custom rules. I rather like what Carl has done with [his linter](https://github.com/ww-tech/lookml-tools/blob/master/README_LINTER.md), making it log to BigQuery. But before LAMS does something like that, I'd like to get our output more structured. For example, what exactly is the difference between our errors and our warnings, and why?

- Extending the language with some more functions than the ones that come out of the box. This will take some hands-on time with the current iteration to see what feels missing.

# Footnotes

1. JS custom rules were behind a defaulted-off command-line option so CI admins would've had to opt in to this unusual security configuration.
2. JS sandboxing (or the lack thereof) could be an article in and of itself. In addition to questionable approaches using Node's VM module, there were a few libraries that approached the problem with their own JS parsers, but all safety guarantees were effectively at parse time, meaning you had to settle for a watered-down subset of Javascript, which kind of defeats the point.