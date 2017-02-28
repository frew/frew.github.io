---
layout: post
title:  "Two Choices About Deployments"
date:   2017-02-28 13:19:00
---

I was reading Dan McKinley’s excellent [blog post](https://blog.skyliner.io/you-cant-have-a-rollback-button-83e914f420d9) about the lack of a reverse gear in your Internet deployment dumptruck, and he makes some great points about limiting the unexpected impact of changes. Deployment systems are an area that I’ve always been adjacent to but never had a chance to dig into deeply, so I wanted to put some thoughts to paper in hopes of clarifying them. It seems like there a couple of key choices here that determine what your production infrastructure looks like, all focused on balancing convenience in the common case with intentionality and ease-of-debugging in the abnormal case.

### Choice 1: Does the change in application behavior happen when you physically deploy a commit or when you update a feature flag?

This is essentially a choice of whether to make the change in behavior implicit in which commit of your application the load balancer sends a request to or whether to make the change in behavior explicit in the code path a request follows in your application. Dan’s post makes a compelling argument that due to shared state (e.g. in databases and caches), the application developer often needs to reason about the interaction of these changes with shared state. Application developers are used to reasoning about interactions between code paths but not about interactions between different revisions of code in a repository. So gating behavior via code paths (i.e. feature flagging) lets them employ that experience to reason about code versions.

![Flags are hard work but can pay off](/img/flags.jpg)

Additional benefits include:

* Feature flags are independent (i.e. you can deploy a revision with changes to feature flag A, B, and C and enable A and C but not B), decreasing the need for a global deploy lock or Git hijinks to isolate changes under deployment.
* They are also easier to test since test frameworks can easily enable/disable feature flags as part of tests.
* Also, it’s much easier to create a system that quickly enables/disables feature flags than it is to quickly deploy new repository revisions, although some blue/green deploy systems come close.

Downsides of feature flagging include:

* Feature flag driven deployment requires the additional changes to the code, as well as maintaining or removing the flag code after the feature has matured.
* It also requires an investment in tooling. You need a mechanism to manage feature flags outside of your existing physical deployment infrastructure.
* You also need tooling to debug issues associated with a particular feature flag (e.g. at the very least you need to tie a request back to its feature flag set for debugging).

To some extent this is mitigated by a smaller set of tooling needs around physical deployments, but unless application developers are fanatically committed to feature flagging all changes (probably past the reasonable cost/benefit tradeoff area of the spectrum), there are still going to be places where the physical deployment matters as well.

### Choice 2: What invariants do you establish about the ways in which code can be deployed?

Even with explicit feature flagging, there are still lots of ways that different code paths can interact unexpectedly.

* A database can have an invalid value due to a historically loose validation logic in the non-feature-flagged code path or an old code path that no longer exists. (forward compatibility)
* A database can have a new value in a feature flagged code path that a particular non-feature flagged code path doesn’t handle properly. (backward compatibility)
* A quick fix (e.g. fix.py in Dan’s example) could have not been replicated to the old feature flag path. (code skew)
* A user can get into a bad state due to a feature flagged path that isn’t resolved by turning off the feature. This is the feature flag version of Dan’s blog post example. (mangled state)

Like concurrency generally, a helpful way you can attack this is by declaring invariants and trying to enforce them, either technically or organizationally. Some invariants that I’ve found useful or interesting in the past:

**Sticky feature flags:** We turn feature flags on and off for a particular user rather than a particular request. This decreases the number of times the old code path interacts with the new one.

**One-way sticky feature flags:** A particular user can only become more featureful (i.e. you don’t turn feature flags off for a given user once they’re on). This removes the common case of backward compatibility issues and requires that application developers fix forward mangled state problems.

**Feature flag testing regime:** I’ve seen both randomized enabling/disabling of feature flags and deployment regimes that require test coverage under both branches of the feature flag before deployment. I tend to be more excited about the latter since one of the main points of feature flagging is to force developers to think about interactions.

**Distinguish between feature flags and larger functionality flags/off switches:** Dan's post calls switches that disable whole portions of functionality "off switches". It's useful to distinguish between feature flags that gate a new deployment change and off switches or functionality flags that turn off large but non-essential portions of the app. It's easier to reason and test the latter over time, and can be safer as a roll-back option.

I’m curious to know what other deployment challenges folks have encountered, invariants folks have found helpful in reasoning about deploys, or other ways of thinking about these problems that have proven useful.

