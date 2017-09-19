---
layout: post
title:  "How to make RxJava crash logs more informative"
tags: rxjava rx
---

## Breadcrumbs

<img src="{{site.url}}{{site.baseurl}}/assets/hansel-and-gretel-breadcrumbs.jpg" alt="Drawing" style="width: 678px;"/>

TODO: better title, something about a problem and how to solve it

TODO: Intro about that sometimes something crash in RxJava and you have no idea what

Consider following code:

```kotlin
just("a string").map { null }
  .subscribeOn(Schedulers.io())
  .subscribe()
}
```

You'll see this crash log:

```
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

Oh my, something inside RxJava guts just crashed!

Ho hint on where the chain was created or where is the subscriber.

Now, if it's a real project, you can spend the rest of your day bisecting your code in hopes to eventually isolate the bug. And even this will work only if you know how to reproduce the thing. If it's something from crash report system, then you have a crash on your hands and you can do nothing to fix it.

If you think that this is synthetic example, don't. I've seen this kind of crashlogs pretty regularly. All you need to do to have it is:

1. Crash occurred in the guts of RxJava
2. Crash occurred on background thread

You can have similar problem if one of your observables is reused across several chains and runs on separate thread (a Retrofit observable for example). Then you'll see which observable've crashed, but you will be unable to see in which exactly chain.

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

<!-- Breadcrumbs. It's hard to understand what crashed if something crashed inside RxJava. -->
<!-- Error handling in RxJava, part 2. Breadcrumbs -->
