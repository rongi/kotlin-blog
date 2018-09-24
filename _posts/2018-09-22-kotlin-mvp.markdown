---
layout: post
title: "Kotlin MVP/MVVM design hints"
excerpt: "Kotlin MVP design hints"
tags: kotlin mvp
categories: kotln mvp mvvm
image: /assets/rxui/display-with-wires.jpg
permalink: kotlin-mvp.html
date: 2018-09-22
published: false
---

I've been doing Kotlin MVP for some time and we with my team came to some best practices.

## Add stuff to the constructor

Immutable data is your friend.

do/dont

Smaller mocks - faster tests

## View should provide as few data as possible, prefer providers in the constructor

## Presenter functions are preferred to be onSomethingHappened(), not doSomething()

## Use funtions instead of providers

## Create constructor funtions for tests

<!-- Maybe say something about presenter interface/presenter incoming callbacks -->

<!-- Maybe say something about view presenters -->

