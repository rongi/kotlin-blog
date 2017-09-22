---
layout: post
title: "RxJava - Better crash logs or how to always know which of your observables just crashed"
excerpt: "Once in a while, something inside RxJava will crash and give you absolutely useless crash log.
How to never get into this kind of situation?"
tags: rxjava rx
categories: rxjava
published: false
---

<img src="{{site.url}}{{site.baseurl}}/assets/hansel-and-gretel-breadcrumbs.jpg" alt="Drawing" style="width: 740px;"/>

Once in a while, one of your observables will crash and leave behind a crash log like this:

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

Just look at all these sweet `io.reactivex` and `java` packages! There is no single link to your code here. No hint on where the chain was created or who is the subscriber.

Now, it's probably time to abandon whatever you were doing right now and spend the rest of your day in an exciting task of bisecting your code in hope to isolate the crash. And dear lord you are lucky if this one is easy to reproduce!

This kind of problems happens quite regularly. For example, the crash log above was produced by this code:

```kotlin
just("a string").map { null }
  .subscribeOn(Schedulers.io())
  .subscribe()
}
```

Just run some of your observables on a separate thread (like you do it with Retrofit observables) and you can run into this kind of issues.

Similar thing can happen if you reuse the same observable (or observable creation code, like Retrofit calls) across multiple chains. Or if you subscribe to these observables more than once. In this case you may be unable to identify which chain have just crashed even if you can identify the crashed observable itself.

You may think: "Sounds like it may be useful to add info to your logs about where the crashed observable was created." But how to do that? Is that possible?

## Breadcrumb exception

Well, it is possible. Just decorate all your chains with this extension function:

```kotlin
fun <T> Observable<T>.dropBreadcrumb(): Observable<T> {
  val breadcrumb = BreadcrumbException()
  return this.onErrorResumeNext { error: Throwable ->
    throw CompositeException(error, breadcrumb)
  }
}
```

Like this:

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

## How does it work

If you look at the code then it's pretty obvious that it just creates an exception with a callstack pointing to the point where the chain was created. Or, to be more precice, where `dropBreadcrumb()` was applied. Then it decorates each error emitted by the chain with this exception.

## Can I do it with Java?

For people like me, who are still forced to do Java, here is Java example:

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
```

Here is how you use it:

```java
Observable.just("a string").map(...)
    .subscribeOn(Schedulers.io())
    .compose(new DropBreadcrumb<String>())
    .subscribe();
```

## Read next

> #### [Error handling in RxJava]({{site.baseurl}}{% post_url 2017-08-01-error-handling-in-rxjava %})
>
>What is the best way to handle errors in RxJava?

<!-- Now, you at least know which of your observables just exploded!
 -->

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
