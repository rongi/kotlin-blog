---
layout: post
title:  "Error handling in RxJava"
description: "What is the best way to handle errors in RxJava?"
excerpt: "What is the best way to handle errors in RxJava?"
categories: rxjava rx
tags: rxjava rx
---

<img src="{{site.url}}/{{site.baseurl}}/assets/error-handling-in-rxjava-title.jpg" alt="Drawing" style="width: 638px;"/>

Once you start writing RxJava code you realize that some things can be done in different ways and sometimes it's hard to identify best practices right away. Error handling is one of these things.

So, what is the best way to handle errors in RxJava and what are the options?

## Handling errors in onError consumer

Let's say you have an observable that can produce an exception. How to handle that? First instinct is to handle errors directly in `onError` consumer.

```kotlin
  userProvider.getUsers().subscribe(
    { users -> onGetUsersSuccess(users) },
    { e -> onGetUsersFail(e) } // Stop the progress, show error message, etc.            
  )
```

It's similar to what we used to do with `AssyncTasks` and looks pretty much like a try-catch block.

There is one big problem with this though. Say there is a programming error inside `userProvider.getUsers()` observable that leads to `NullPointerException` or something like this. It'll be super convenient here to crash right away so we can detect and fix the problem on the spot. But we'll see no crash, the error will be handled as an expected one: an error message will be shown, or in some other graceful way.

Even worse is that there wouldn't be any crash in the tests. The tests will just fail with mysterious unexpected behavior. You'll have to spend time on debugging instead of seeing the reason right away in a nice call stack.

## Expected and unexpected exceptions

Just to be clear let me explain what do I meant here by expected and unexpected exceptions.

Expected exceptions are those that are expected to happen in a bug-free program. Examples here are various kinds of IO exceptions, like no network exception, etc. Your software is supposed to react on these exceptions gracefully, showing error messages, etc. Expected exceptions are like second valid return value, they are part of method's signature.

Unexpected exceptions are mostly programming errors. They can and will happen during development, but they should never happen in the finished product. At least it's a goal. But if they do happen, usually it's a good idea just to crash the app right away. This helps to raise attention to the problem quickly and fix it as soon as possible.

In Java expected exceptions are mostly implemented using checked exceptions (subclassed directly from `Exception` class). The majority of unexpected ones are implemented with unchecked exceptions and derived from `RuntimeException`.

## Crashing on RuntimeExceptions

So, if we want to crash why don't just check if the exception is a `RuntimeException` and rethrow it inside `onError` consumer? And if it's not just handle it like we did it in the previous example?

```kotlin
  userProvider.getUsers().subscribe(
    { users -> onGetUsersSuccess(users) },
    { e ->
      if (e is RuntimeException) {
        throw e
      } else {
        onGetUsersFail(e)
      }
    }
  )
```

This one may look nice, but it has a couple of flaws:

1. In RxJava 2 this will crash in the live app but not in the tests. Which can be extremely confusing. In RxJava 1 though it will crash both in the tests and in the application.
2. There are more unchecked exceptions besides `RuntimeException` that we want to crash on. This includes `Error`, etc. It's hard to track all exceptions of this kind.

But the main flaw is this:

During application development your Rx chains will become more and more complex. Also your observables will be reused in different places, in the contexts you never expected them to be used in.

Imagine you've decided to use `userProvider.getUsers()` observable in this chain:

```kotlin
Observable.concat(userProvider.getUsers(), userProvider.getUsers())
  .onErrorResumeNext(just(emptyList()))
  .subscribe { println(it) }
```

What will happen if both `userProvider.getUsers()` observables emit an error?

Now, you may think that both errors will be mapped to an empty list and so two empty lists will be emitted. You may be surprised to see that actually only one list is emitted. This is because error occurred in the first `userProvider.getUsers()` will terminate the whole chain upstream and second parameter of `concat` will never be executed.

You see, errors in RxJava are pretty destructive. They are designed as fatal signals that stop the whole chain upstream. They aren't supposed to be part of interface of your observable. They perform as unexpected errors.

Observables designed to emit errors as a valid output have limited scope of possible use. It's not obvious how complex chains will work in case of error, so it's very easy to misuse this kind of observables. And this will result in bugs. Very nasty kind of bugs, those that are reproducible only occasionally (on exceptional conditions, like lack of network) and don't leave stack traces.

## Result class

So, how to design observables that return expected errors? Just make them return some kind of `Result` class, which will contain either result of the operation or an exception. Something like this:

```kotlin
data class Result<out T>(
  val data: T?,
  val error: Throwable?
)
```

Wrap all expected exceptions into this and let all unexpected ones fall through and crash the app. Avoid using `onError` consumers, let RxJava do the crashing for you.

Now, while this approach doesn't looks particularly elegant or intuitive and produces quite a bit of boilerplate, I've found that it causes the least amount of problems. Also, it looks like this is an "official" way to do error handling in RxJava. I saw it recommended by RxJava maintainers in multiple discussions across Internet.

## Some useful code snippets

To make your Retrofit observables return `Result` you can use this handy extension function:

```kotlin
fun <T> Observable<T>.retrofitResponseToResult(): Observable<Result<T>> {
  return this.map { it.asResult() }
    .onErrorReturn {
      if (it is HttpException || it is IOException) {
        return@onErrorReturn it.asErrorResult<T>()
      } else {
        throw it
      }
    }
}

fun <T> T.asResult(): Result<T> {
  return Result(data = this, error = null)
}

fun <T> Throwable.asErrorResult(): Result<T> {
  return Result(data = null, error = this)
}
```

Then your `userProvider.getUsers()` observable can look like this:

```kotlin
class UserProvider {
  fun getUsers(): Observable<Result<List<String>>> {
    return myRetrofitApi.getUsers()
      .retrofitResponseToResult()
  }
}
```
