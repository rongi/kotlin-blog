---
layout: post
title:  "Make your RxJava crash logs more informative by adding breadcrumbs"
excerpt: "Once in a while, something inside RxJava guts crashes and spits out perfectly useless crash log.
This is about how to make it useful."
tags: rxjava rx
---

<img src="{{site.url}}{{site.baseurl}}/assets/hansel-and-gretel-breadcrumbs.jpg" alt="Drawing" style="width: 740px;"/>

Once in a while, one of your observables will go down in a blaze of glory and leave perfectly useless crash log behind. Something like this.

<!-- <img src="{{site.url}}{{site.baseurl}}/assets/rxjava-breadcrumbs-log.jpg" alt="Drawing" style="width: 740px;"/> -->

```java
java.lang.NullPointerException: The mapper function returned a null value.
    at io.reactivex.internal.functions.ObjectHelper.requireNonNull(ObjectHelper.java:39)
    at io.reactivex.internal.operators.observable.ObservableMap$MapObserver.onNext(ObservableMap.java:58)
    at io.reactivex.internal.operators.observable.ObservableScalarXMap$ScalarDisposable.run(ObservableScalarXMap.java:246)
    at io.reactivex.internal.operators.observable.ObservableJust.subscribeActual(ObservableJust.java:35)
    at io.reactivex.Observable.subscribe(Observable.java:10179)
    at io.reactivex.internal.operators.observable.ObservableMap.subscribeActual(ObservableMap.java:32)
    at io.reactivex.Observable.subscribe(Observable.java:10179)
    at io.reactivex.internal.operators.observable.ObservableSubscribeOn$1.run(ObservableSubscribeOn.java:39)
    at io.reactivex.Scheduler$1.run(Scheduler.java:134)
    at io.reactivex.internal.schedulers.ScheduledRunnable.run(ScheduledRunnable.java:59)
    at io.reactivex.internal.schedulers.ScheduledRunnable.call(ScheduledRunnable.java:51)
    at java.util.concurrent.FutureTask.run(FutureTask.java:237)
    at java.util.concurrent.ScheduledThreadPoolExecutor$ScheduledFutureTask.run(ScheduledThreadPoolExecutor.java:269)
    at java.util.concurrent.ThreadPoolExecutor.runWorker(ThreadPoolExecutor.java:1113)
    at java.util.concurrent.ThreadPoolExecutor$Worker.run(ThreadPoolExecutor.java:588)
    at java.lang.Thread.run(Thread.java:818)
```

Looks at all these sweet `io.reactivex` and `java` packages! There is no single link to your code here. No hint on where the chain was created or who is the subscriber.

Now, it's probably time to abandon whatever you were doing and spend the rest of your day in that exciting task of bisecting your code in hope to isolate the bug. And dear lord you are lucky if this one is easy to reproduce!

This kind of problems happens pretty regularly. For example the log above was produced by the following code:

```kotlin
just("a string").map { null }
  .subscribeOn(Schedulers.io())
  .subscribe()
}
```

All you need to have to have this kind of bugs is this:

1. Crash occurred in the guts of RxJava
2. Crash occurred on a background thread

Also you may be able to identify which observable crashed but unable to identify in which chain it was used. For example Retrofit observable reused. 

You can have similar problem if one of your observables is reused across several chains and runs on separate thread (a Retrofit observable for example). Then you'll see which observable've crashed, but you will be unable to see in which chain.

To fix it use breadcrumbs:

```kotlin
fun <T> Observable<T>.dropBreadcrumb(): Observable<T> {
  val breadcrumb = BreadcrumbException()
  return this.onErrorResumeNext { error: Throwable ->
    throw CompositeException(error, breadcrumb)
  }
}
```

Usage of breadcrumbs:

```kotlin
just("a string").map { null }
  .subscribeOn(Schedulers.io())
  .dropBreadcrumb()
  .subscribe()
```

TODO: Add Java example

Now, you at least know which of your observables just exploded!

<!-- Oh my, something inside RxJava guts just crashed! -->


<!-- Breadcrumbs. It's hard to understand what crashed if something crashed inside RxJava. -->
<!-- Error handling in RxJava, part 2. Breadcrumbs -->

<!-- Title -->
<!-- Title -->
<!-- Title -->

<!-- RxJava: better crashlogs with breadcrumbs -->



<!-- Intro -->
<!-- Intro -->
<!-- Intro -->

<!-- Sometimes RxJava produces useless crashlogs

It happens from time to time that RxJava produces crashlogs that are totally useless.

It happens from time to time that RxJava produces crash logs that are totally useless. This article is about how to spice up your crash logs with more useful information.

Sometimes something crashes in your RxJava observables

Sometimes your RxJava observables crash. And sometimes crashlogs produced by this crashes are totally meaningless.

Sometimes your RxJava observables crash. And crash logs produced by these crashes are totally useless sometimes.

Sometimes your RxJava observables crash. And crash logs produced can be totally useless.

Sometimes your RxJava observables crash and crash logs with crash logs which are useless.

Sometimes your RxJava observables crash and spit up crash logs which are useless. -->

<!-- Once in a while your RxJava observable will crash and spit out crash log which is totally useless. -->

<!-- Once in a while RxJava observables crash and spit out crash logs which are totally useless. -->

<!-- Once in a while your RxJava observable will crash and spit out crash log which is totally useless.

Once in a while something inside RxJava will crash and spit out crash log which is totally useless.

Once in a while something inside RxJava will crash and give you totally useless crashlog.

Once in a while something inside RxJava guts will crash and you will be presented with perfectly useless crash log. -->

<!-- Once in a while, something inside RxJava guts will crash and spit out perfectly useless crash log.

This is how to avoid it. -->

<!--
Once in a while, something inside RxJava guts explodes and spits out perfectly useless crash log.

Once in a while, one of your observables will crash and spit out perfectly useless crash log. Something like this.
-->

<!-- How to make these crash logs more informative? -->

<!-- Something -->
<!-- Something -->
<!-- Something -->

<!-- Now, having crash on your hands which you can't localize can -->

<!-- is a very very grim situation. make you -->

<!-- Here is how to improve the situation. -->

<!-- This post is about how to avoid this kind of situation and make all your RxJava crash logs meaningful.

This post is about how to spice up your crash logs with useful data

This post is about how to add information to the crash logs that will help localize the problem.

Learn how to make crash logs more informative.

And then you are in a very bad situation.

Now, having crash on your hands which you can't localized is a very very bad situation. -->
