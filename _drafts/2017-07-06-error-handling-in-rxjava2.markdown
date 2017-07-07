---
layout: post
title:  "Error handling in RxJava 2"
categories: rxjava rx
published: false
---

## Intro

When using RxJava many thing you have to figure out on your own. One of these things is how to handle errors.

## Don't throw

First instinct is to handle errors on onError() callback of Subscriber. This have some problems. RxJava is designed in the way that errors received in this callback are not expected errors but rather unexpected ones, like programming errors.

Why not do like this:

```kotlin
/**
 * Will crash on RuntimeExceptions
 */
fun <T> Observable<T>.subscribeX(onNext: (T) -> Unit, onError: (Throwable) -> Unit) {
  this.subscribe({
    onNext(it)
  }, { e ->
    if (e is RuntimeException) {
      throw e
    } else {
      onError(e)
    }
  })
}
```

Examples:

1.
```kotlin
Observable.just("1", "2", "3")
  .map(string -> { throw new RuntimeException(); })
  .onErrorReturn(error -> "Error")
  .subscribe(System.out::println);
```

2.
It doesn't throw in unit tests.


First I tried to use onError() to handle expected exceptions and I'd invested quite an effort into it. But I encountered more and more problems on the way.

Handling errors in onError() causes problems in tests in RxJava 2.

RxJava errors are RuntimeErrors, programming errors, unexpected errors. If you call is expected to throw some errors sometimes, like http requests are expected to have "no internet" errors, then you better use compound result with either return value or exception.

Throwing inside an observable expecting to "catch" that exception inside onError() method of the subscriber is not a valid option.

Yes, it is uncomfortable

## Breadcrumbs

Breadcrumbs. It's hard to understand what crashed if something crashed inside RxJava.
