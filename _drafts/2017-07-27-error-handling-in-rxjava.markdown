---
layout: post
title:  "Error handling in RxJava, best practices"
categories: rxjava rx
---

Once you start write RxJava code you realize that there are so many ways to handle errors. What is the best one? I've spent quite some time trying to answer this question and here are my findings along with some supporting arguments.

## Expected and unexpected exceptions

Just to start on the same page I want to remind you that errors/exceptions in software development can be divided into two groups.

Expected exceptions are the ones that are expected to happen in a bug-free program. Examples here are various kinds of IO exceptions, like no network exception, etc. Developers should write code that reacts on this kind of exceptions, shows error message, etc. Expected exceptions are like second valid return value, they are always part of a method's signature.

Unexpected exceptions are programming errors mostly. They can and will happen during development, but they should never happen in the finished product. At least it's the goal. But if they do happen, it's a good idea usually just to crash the app right away. This helps to raise attention to the problem and fix it as soon as possible.

In Java expected exceptions are mostly implemented using checked exceptions (subclassed directly from `Exception` class). The majority of unexpected ones are implemented with unchecked exceptions and derived from `RuntimeException`.

## Handling errors in onError consumer

So, let's say you have an observable that can produce an exception. How to handle that? First instinct is to handle errors directly in `onError` consumer.

```kotlin
  userProvider.getUsers().subscribe(
    { users -> onGetUsersSuccess(users) },
    { e -> onGetUsersFail(e) } // Stop the progress, show error message, etc.            
  )
```

It's similar to what we used to do with `AssyncTasks` and looks pretty much like a try-catch block.

There is one big problem with this though. Say there is a programming error inside `userProvider.getUsers()` observable that leads to `NullPointerException` or something like this. This error will be handled as an expected one: an error message will be shown and app will not crash. It'll be super convenient here though to see the crash right away so we can detect and fix problem on the spot.

Even worse is that there wouldn't be any crash in the unit tests. The tests will just fail with mysterious unexpected behavior. You'll have to spend time on investigating this instead of seeing the reason right away in the call stack. Crashing is super convenient here.

## Crashing on RuntimeExceptions

Ok, how about this then:

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

This one has many flaws though. Here are some of them:

1. In RxJava 2 this will crash in live app but not in unit tests which can be extremely confusing. In RxJava 1 though it will crash both in unit tests and in live app.
2. There are more unchecked exceptions besides `RuntimeException` that we want to crash on. This includes `Error`, etc. It's hard to track on all exceptions of this kind.

But the main flaw of this approach is this:

During application development Rx chains become more and more complex. Also your observables will be reused in different places in different chains. In the contexts you never expected them to be used in.

Imagine you've decided to use `userProvider.getUsers()` observable in this chain:

```kotlin
Observable.concat(userProvider.getUsers(), userProvider.getUsers())
  .onErrorResumeNext(just(emptyList()))
  .subscribe { println(it) }
```

What will happen if both `userProvider.getUsers()` observables emit an error?

Now, you may think that both of these error will be mapped to an empty list and two empty lists will be emitted. You may be surprised to see that actually only one list is emitted.

You see, errors in RxJava are pretty destructive. They are designed as fatal conditions that stops the whole chain upstream. They aren't supposed to be part of interface of your observable. They are handled by RxJava as unexpected errors should be handled.

If you design your observables to emit errors as a valid output these observables will have limited scope of possible use. It's not obvious how complex chains work in case of error, so it's very easy to misuse error-emitting observables and this will result in bugs. Very nasty kind of bugs, the ones that are reproducible only occasionally (on exceptional conditions, like lack of network) and don't leave stack traces.

## Result class

So, how to design observables that return expected errors? Just wrap them into some kind of `Result` class, some variation of this:

```kotlin
data class Result<out T>(
  val data: T?,
  val error: Throwable?
)
```

Now, while this approach doesn't looks particularly elegant or intuitive and produces quite a bit of boilerplate, I've found that it causes the least amount of problems. Also, it looks like this is an "official" way to do error handling in RxJava. I saw it recommended by RxJava maintainers in multiple discussions across Internet.

## Some useful code snippets

To convert your Retrofit observables to the ones returning `Result` you can use this handy extension function:

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

## WWWWWWWW

Why this article if the conclusion is an official recommendations? Because official recommendations in combination of being counterintuitive and lack supporting arguments are not convincing enough.

not vocal enough

> This is because RxJava is designed in the way that errors received in this callback are not expected errors but rather unexpected ones, like programming errors.

> RxJava errors are RuntimeErrors, programming errors, unexpected errors. If you are designing an observable that is expected to throw errors sometimes, like http requests are expected to have "no internet" errors, then you better use compound result with either return value or exception.

> Throwing inside an observable expecting to "catch" that exception inside onError() method of the subscriber is not a valid option.

> Yes, it is uncomfortable

> If your observables are designed to emit errors as a valid output sooner or later you'll end up in situation when you program doesn't work as expected by some mysterious reason.
