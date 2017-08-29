---
layout: post
title:  "Generate Butter Knife boilerplate"
tags: butterknife android-studio as live-template bind
---

Butter Knife library is pretty cool and it's a big step forward compared to writing `findViewById()` code yourself. Still, it's plenty of boilerplate. There should be a way to generate it, right?

Sure there is. There is a [plugin](https://www.google.com) for Android Studio.

There is also an option to do it with Android Studio's Live Templates.

Here is how Live Templates solution looks in action:

![bind view example]({{site.url}}/{{site.baseurl}}/assets/bind-view-example.gif)

Here you type only meaningful parts of the boilerplate: view id and view type, the rest is auto generated. View name is generated from view id and has proper camel case. All done with standard Android Studio feature. Pretty exciting I would say!

## How to add this to your Android Studio

Open "Live Templates" section in Android Studio preferences. Create a new group there.

<img src="{{site.url}}/{{site.baseurl}}/assets/bind-view-create-group.png" alt="Drawing" style="width: 800px;"/>

Take this XML

```xml
<template name="bind" value="@BindView(R.id.$resId$)&#10;$class$ $variable$;" description="Bind a field to the view for the specified ID." toReformat="false" toShortenFQNames="true">
  <variable name="resId" expression="complete()" defaultValue="" alwaysStopAt="true" />
  <variable name="class" expression="classNameComplete()" defaultValue="" alwaysStopAt="true" />
  <variable name="variable" expression="camelCase(resId)" defaultValue="" alwaysStopAt="true" />
  <context>
    <option name="JAVA_DECLARATION" value="true" />
    <option name="KOTLIN_CLASS" value="true" />
  </context>
</template>
```

and paste it into the group you've just created.

<img src="{{site.url}}/{{site.baseurl}}/assets/bind-view-paste-template.png" alt="Drawing" style="width: 638px;"/>

Voila! Now just type "bind" somewhere inside a class and hit your code completion hotkey.
