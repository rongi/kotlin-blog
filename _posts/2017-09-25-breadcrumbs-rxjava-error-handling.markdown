---
layout: post
title: "RxJava - Better crash logs or how to always know which of your observables crashed"
excerpt: "Once in a while, something inside RxJava will crash and give you absolutely useless crash log.
How to never get into this kind of situation?"
tags: rxjava rx breadcrumbs
categories: rxjava
image: /assets/hansel-and-gretel-breadcrumbs.jpg
---

<img src="{{site.url}}{{site.baseurl}}/assets/hansel-and-gretel-breadcrumbs.jpg" alt="Drawing" style="width: 740px;"/>

Once in a while, one of your observables will crash and leave behind a useless crash log. Something like this:

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

Just look at all these sweet `io.reactivex` and `java` packages. There is no single link to your code! No hints on where the chain was created or who is the subscriber.

Now, it's probably time to abandon whatever you were doing and spend the rest of your day in an exciting task of bisecting your code in hopes of isolating the crash. You are lucky if this one is easy to reproduce!

This kind of problem happens quite regularly. For example, the crash log above was produced by this code:

```kotlin
just("a string").map { null }
  .subscribeOn(Schedulers.io())
  .subscribe()
}
```

This code models a pretty common situation: you've forgotten that something is nullable and return it from your `map` operator. This can happen with your Retrofit observables when you are interested in only a part of the response, and that part happened to be null.

Generally, just run some of your observables on a separate thread (like you do it with Retrofit observables), and you are in a danger zone. RxJava returns call stack only for the thread that crashed, and if the said call stack doesn't contain your code, then, you are out of luck.

A similar thing can happen if you use the same observable (or observable creation code, like Retrofit calls) in several places. In this case, you may be unable to identify which chain have just crashed even if you were lucky enough to identify the crashed observable itself.

You may think: "Sounds like it may be useful to see where the crashed observable was created." Can you do that? Maybe you can add this kind of information to the crash log?

## Breadcrumb exception

Well, actually you can. Just decorate all your chains with this extension function:

```kotlin
fun <T> Observable<T>.dropBreadcrumb(): Observable<T> {
  val breadcrumb = BreadcrumbException()
  return this.onErrorResumeNext { error: Throwable ->
    throw CompositeException(error, breadcrumb)
  }
}

class BreadcrumbException : Exception()
```

Here is an example:

```kotlin
just("a string").map { null }
  .subscribeOn(Schedulers.io())
  .dropBreadcrumb()
  .subscribe()
```

Then you'll start to see crash logs like this:

```java
io.reactivex.exceptions.CompositeException: 2 exceptions occurred.
     at io.reactivex.internal.operators.observable.ObservableOnErrorNext$OnErrorNextObserver.onError(ObservableOnErrorNext.java:94)
     at io.reactivex.internal.operators.observable.ObservableSubscribeOn$SubscribeOnObserver.onError(ObservableSubscribeOn.java:68)
     at io.reactivex.internal.observers.BasicFuseableObserver.onError(BasicFuseableObserver.java:100)
     at io.reactivex.internal.observers.BasicFuseableObserver.fail(BasicFuseableObserver.java:110)
     at io.reactivex.internal.operators.observable.ObservableMap$MapObserver.onNext(ObservableMap.java:60)
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
   ComposedException 1 : java.lang.NullPointerException: The mapper function returned a null value.
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
  Caused by: test.com.sandbox_kotlin.BreadcrumbException
     at test.com.sandbox_kotlin.MainActivityKt.dropBreadcrumb(MainActivity.kt:75)
     at test.com.sandbox_kotlin.MainActivity.onCreate(MainActivity.kt:23)
     at android.app.Activity.performCreate(Activity.java:6237)
     at android.app.Instrumentation.callActivityOnCreate(Instrumentation.java:1107)
     at android.app.ActivityThread.performLaunchActivity(ActivityThread.java:2369)
     at android.app.ActivityThread.handleLaunchActivity(ActivityThread.java:2476)
     at android.app.ActivityThread.-wrap11(ActivityThread.java)
     at android.app.ActivityThread$H.handleMessage(ActivityThread.java:1344)
     at android.os.Handler.dispatchMessage(Handler.java:102)
```

Not excited? Look at this line:
```java
at test.com.sandbox_kotlin.MainActivity.onCreate(MainActivity.kt:23)
```
This is our code! You can just click there now and get to the place where the crashed chain was created.

{:refdef: style="text-align: center;"}
<img src="https://i.giphy.com/media/12NUbkX6p4xOO4/giphy.webp" alt="Drawing" style="width: 300px; margin-top: 30px; margin-bottom: 20px;"/>
{: refdef}

## How does it work?

If you look at the code, then it's pretty obvious that it just creates an exception with a call stack pointing to the point where the chain was created --- or, to be more precise, where `dropBreadcrumb()` was applied. Then it decorates each error emitted by the chain with this exception.

## Can I do it with Java?

For people like me, who are still forced to do Java, here is a Java example:

```java
public final class DropBreadcrumb<T> implements ObservableTransformer<T, T> {
    @Override
    public ObservableSource<T> apply(Observable<T> upstream) {
        final BreadcrumbException breadcrumb = new BreadcrumbException();
        return upstream.onErrorResumeNext(new Function<Throwable, ObservableSource<? extends T>>() {
            @Override
            public ObservableSource<? extends T> apply(Throwable throwable) throws Exception {
                throw new CompositeException(throwable, breadcrumb);
            }
        });
    }
}

class BreadcrumbException extends Exception {
}
```

Here is how you use it:

```java
Observable.just("a string").map(...)
    .subscribeOn(Schedulers.io())
    .compose(new DropBreadcrumb<String>())
    .subscribe();
```

## Can I do it automatically?

<!--
Yep, and that's exactly what this plugin does: https://github.com/T-Spoon/Traceur.
It uses Rx2's plugin system to hook into every operator call and performs the same logic as described in the article to dump granular stacktraces.

Ours cover the entire JVM in case you have difficulty locating the source of the error. Yours can be applied when the search is narrower.
 -->

## More about error handling in RxJava

> ### [Error handling in RxJava]({{site.baseurl}}{% post_url 2017-08-01-error-handling-in-rxjava %})
> Once you start writing RxJava code you realise that some things can be done in different ways and sometimes it’s hard to identify best practices right away. Error handling is one of these things.<br><br>So, what is the best way to handle errors in RxJava and what are the options?
