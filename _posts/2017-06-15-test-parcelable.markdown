---
layout: post
title:  "Test parcelable serialisation/deserialisation"
date:   2017-06-15 16:07:20 +0200
categories: kotlin parcelable test
---

It happens pretty often that you make some update in a parcelable that breaks it's serilisation/deserialisation process. It results in crashes that are detected late in the test process or even not detected at all and end up in production.

This function tests that parcelable serializes and deserialises without crashes and into the same object. It should be run from Android instrumentation test.

```kotlin
fun <T : Parcelable> assertRecreates(initial: T) {
  val parcel = Parcel.obtain()
  initial.writeToParcel(parcel, 0)  

  parcel.setDataPosition(0)
  val creator = initial.javaClass.fields.first { it.name == "CREATOR" }.get(null) as Parcelable.Creator<*>
  val fromParcel = creator.createFromParcel(parcel)

  assertEquals(initial, fromParcel)
}
```

Usage:

```kotlin
@RunWith(AndroidJUnit4::class)
class ParcelablesTest {

  @Test
  fun testParcelable() {
    val user = User("Jon", "Snow")
    assertRecreates(user)
  }
}
```
