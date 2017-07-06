---
layout: post
title:  "Error handling in RxJava 2"
categories: rxjava rx
published: false
---

When using RxJava many thing you have to figure out on your own. One of these things is how to handle errors.

First instinct is to handle errors on onError() callback of Subscriber. This have some problems. RxJava is designed in the way that errors received in this callback are not expected errors but rather unexpected ones, like programming errors. (Find examples why it is designed this way)

First I tried to use onError() to handle expected exceptions and I'd invested quite an effort into it. But I encountered more and more problems on the way.

Handling errors in onError() causes problems in tests in RxJava 2.

Breadcrumbs. It's hard to understand what crashed if something crashed inside RxJava.

RxJava errors are RuntimeErrors, programming errors, unexpected errors. If you call is expected to throw some errors sometimes, like http requests are expected to have "no internet" errors, then you better use compound result with either return value or exception.

Throwing inside an observable expecting to "catch" that exception inside onError() method of the subscriber is not a valid option.
