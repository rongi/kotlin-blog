---
layout: post
title:  "Clear application cache from console"
categories: adb android
---

This here is the single most useful command in the adb arsenal:

```bash
adb shell pm clear my.app.package.name
```

It clears application cache with a single console command. The alternative is to browse list of installed apps on the phone, find your app and then click "clear cache" button. Pretty tedious if you need to do it often.

Why do you need to clear the cache? It's a simple way to reliably remove all the user data to test the app with another user/environment. Also it's common that during development we introduce changes that make current cache invalid. And it's a good way to simulate a fresh install of the app.

As a bonus here is a command to uninstall your application:

```bash
adb uninstall my.app.package.name
```
