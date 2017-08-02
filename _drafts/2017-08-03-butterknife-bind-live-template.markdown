---
layout: post
title:  "Live template for Butter Knife bindings"
excerpt: "Live template for Butter Knife bindings"
tags: butterknife android-studio as live-template bind
---

with Powerful Android Studio Live Templates feature

code completion

![bind view example]({{site.url}}/{{site.baseurl}}/assets/bind-view-example.gif)

Create new group in "Live Templates" part of your settings

<img src="{{site.url}}/{{site.baseurl}}/assets/bind-view-create-group.png" alt="Drawing" style="width: 800px;"/>

Copy this xml

```xml
<template name="bind" value="@BindView(R.id.$resId$)&#10;$class$ $variable$;" description="Bind a field to the view for the specified ID. The view will automatically be cast to the field type." toReformat="false" toShortenFQNames="true">
  <variable name="resId" expression="complete()" defaultValue="" alwaysStopAt="true" />
  <variable name="class" expression="classNameComplete()" defaultValue="" alwaysStopAt="true" />
  <variable name="variable" expression="camelCase(resId)" defaultValue="" alwaysStopAt="true" />
  <context>
    <option name="JAVA_DECLARATION" value="true" />
  </context>
</template>
```

 Paste it into the group you've just created

<img src="{{site.url}}/{{site.baseurl}}/assets/bind-view-paste-template.png" alt="Drawing" style="width: 638px;"/>
