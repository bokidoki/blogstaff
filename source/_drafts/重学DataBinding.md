---
title: 重学DataBinding
date: 2019-11-21 10:09:08
tags:
categories:
img:
---

- databinding xml中字符串比较，首先字符串用``引用，其次就无所谓了，可以用equals, ==, 三元表达式等

```xml
 android:text="@{UserDataStore.instance.userInfo.sex.equals(`0`) ? R.string.xcl_male : R.string.xcl_female}"
```
