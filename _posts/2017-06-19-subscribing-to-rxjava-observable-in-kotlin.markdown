---
layout: post
title:  "Java vs Kotlin: subscribing to an observer in RxJava"
categories: kotlin rxjava rx
---

Using RxJava with Kotlin is much more convenient. The difference is so big, that RxJava simply becomes more powerful instrument with Kotlin.

Here is an example of how subscribing typically looks in Java and in Kotlin.

Java:

```java
void getUserPosts(InstagramApi instagramApi) {
    instagramApi.getRecentPosts().subscribe(new Consumer<List<InstagramPost>>() {
        @Override
        public void accept(List<InstagramPost> instagramPosts) throws Exception {
            onGetRecentPostsSuccess(instagramPosts);
        }
    }, new Consumer<Throwable>() {
        @Override
        public void accept(Throwable throwable) throws Exception {
            onGetRecentPostsFail(throwable);
        }
    });
}

private void onGetRecentPostsSuccess(List<InstagramPost> posts) {
    // handle posts received
}

private void onGetRecentPostsFail(Throwable posts) {
    // handle get posts fail
}
```

Kotlin:

```kotlin
fun getUserPosts(instagramApi: InstagramApi) {
  instagramApi.getRecentPosts().subscribe(
    { onGetRecentPostsSuccess(it) },
    { onGetRecentPostsFail(it) }
  )
}

private fun onGetRecentPostsSuccess(posts: List<InstagramPost>) {
  // handle posts received
}

private fun onGetRecentPostsFail(posts: Throwable) {
  // handle get posts fail
}
```

I should note here though, that handling errors in the subscriber is usually a bad idea. If you are expecting errors, like you usually do with HTTP requests, it's better to catch them with `onErrrorResumeNext()` or `onErrorReturn()`. And you observable should return object that contains either received data or error.

This is because of how RxJava is designed. Errors returned from `onError` are analogs of runtime errors, unexpected errors, like programming errors.