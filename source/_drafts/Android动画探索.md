---
title: Android动画探索
tags:
categories:
date:
img:
---

## 与动画相关的包

com.andriod.animation

- Animator 所有动画的基类 √
- AnimatorInflater animation xml 解析器 ×
- AnimatorListenerAdapter animation 动画监听适配器AnimatorListener、AnimatorPauseListener的空实现，可以自行选择去监听animator的状态 ×
- AnimatorSet 以特定的顺序执行Animator，可以选择一起执行、顺序执行、延迟执行 √

- ArgbEvaluator argb颜色的估值器 ×
- FloatArrayEvaluator 单精度浮点数组的估值器 ×
- IntArrayEvaluator 整型数组的估值器 ×
- IntEvaluator 整型估值器 ×
- TypeEvaluator 类型估值器基类 专门为ValueAnimator属性动画setEvaluator使用 ×

- BidirectionalTypeConverter<T, V> 可以将T转换成V，也可以将V转换回来 ×
- TypeConverter<T, V> 类型转换器基类 ×

- Keyframe 关键帧，保存着时间/值的

com.android.view.animation
com.google.android.material.animation

## 阅读笔记

- animator 与 animation 的区别
- 差值器和估值器的区别

## 使用笔记

- 如何在xml中配置activity的进出场动画
- activityOpenEnterAnimation、activityOpenExitAnimation、activityCloseEnterAnimation、activityCloseExitAnimation这些值又代表什么呢？
- 共享元素过渡动画是什么?怎么去实现?
