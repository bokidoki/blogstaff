---
title: Kotlin语法糖 Part1
tags: 基础
categories: Kotlin
date: 2019-03-23
thumbnail: https://dreamweaver.img.we1code.cn/kotlin-sugar.jpg
top: 0
---

Kotlin给我们提供了大量的工具和语法糖让我们能够更为便利的去编程，让代码有更好的可读性和可扩展性。写更少的代码做更多的事，用这句话概括Kotlin和Java之间的差异一点都不为过。面对Kotlin这种能减轻我们工作量的工具，我们有什么理由不去学习它呢？我相信有效地使用Kotlin会对你的身心带来巨大的愉悦，在使用Kotlin的过程中，它的简洁和优雅的语法不断地给我带来惊喜，可能这也是Google使用它作为Android官方编程语言的原因吧。Kotlin的语法糖有很多，我至今也还在学习中，接下来我将用三篇文章的篇幅将目前我使用较多的介绍给大家。这篇文章是这个系列的第一章，在这张中我们主要来了解下密封类(sealed class)的用法。

<!--more-->

## 密封类(sealed class)

用sealed关键字修饰的类我们称之为密封类，在[官方文档](https://www.kotlincn.net/docs/reference/sealed-classes.html)中是这么介绍的  
> 密封类用来表示受限的类继承结构，当一个值为有限集中的类型、而不能有任何其他类型。

这是不是很枚举类很相似呢，我认为sealed class比枚举的用法更为灵活，每个枚举类只能存在一个实例，但是密封类的子类可以包含多个不同内部状态的实例。上代码

```kotlin
sealed class Response

data class Success(val body: String): Response()

data class Error(val code: Int, val message: String): Response()

object Timeout: Response()

```

sealed class 自身抽象，它不能直接实例化，但是可以有abstract成员。我们使用IntelliJ IDEA Kotlin Bytecode工具将上面的代码还原程Java代码。

1.Reveal Koltin Bytecode

![pic_1](https://dreamweaver.img.we1code.cn/kotlin_syntax_pic_2.png "1.Reveal Kotlin Bytecode")

2.Decompile Kotlin Bytecode to Java code

![pic_2](https://dreamweaver.img.we1code.cn/kotlin_syntax_pic_1.png "2.Decompile Kotlin Bytecode to Java code")

经过这几步，就可以开始阅读转换后Java代码了

```java
public abstract class Response {
   private Response() {
   }

   // $FF: synthetic method
   public Response(DefaultConstructorMarker $constructor_marker) {
      this();
   }
}
```

你可能已经在想，seald classes专门为继承而生，因此他们是抽象的。但是，他们怎么会类似enums?下面是Kotlin编译器通过允许您使用Response类的子类作为(as case of)when（）函数的条件时给您带来巨大帮助的时候了。另外，kotlin提供了极大的灵活性(flexibility)，继承sealed class的对象可以
是一个data class(数据类)或object

```kotlin
fun sugar(response: Response) = when (response) {
    is Success -> ...
    is Error -> ...
    Timeout -> ...
}
```

这样的代码看上去不仅仅是结构清晰，在使用它时，你甚至不用去做额外的强制转换，在条件语句的最后也不再需要else子句了，我们已经覆盖了所有的情况。

```kotlin
fun sugar(response: Response) = when (response) {
    is Success -> println(response.body)
    is Error -> println("${response.code} ${response.message}")
    Timeout -> println(response.javaClass.simpleName)
}
```

不使用sealed关键字的代码是怎样的？可以用IntelliJ IDEA Kotlin Bytecode转换成Java看下

```java
public final void sugar(@NotNull Response response) {
   Intrinsics.checkParameterIsNotNull(response, "response");
  
   String var3;
   if (response instanceof Success) {
      var3 = ((Success)response).getBody();
      System.out.println(var3);
   } else if (response instanceof Error) {
      var3 = "" + ((Error)response).getCode() + ' ' + ((Error)response).getMessage();
      System.out.println(var3);
   } else {
      if (!Intrinsics.areEqual(response, Timeout.INSTANCE)) {
         throw new NoWhenBranchMatchedException();
      }

      var3 = response.getClass().getSimpleName();
      System.out.println(var3);
   }
}
```

可见使用Kotlin我们减少了多少代码量。

## 使用when()方法自由的排列组合

接下来我们看看枚举类和when()的配合。

```kotlin
enum class Employee {
    DEV_LEAD,
    SENIOR_ENGINEER,
    REGULAR_ENGINEER,
    JUNIOR_ENGINEER
}

enum class Contract {
    PROBATION,
    PERMANENT,
    CONTRACTOR,
}
```

Employee枚举定义了Company中的所有角色，Contract枚举包含了所有类型的员工合同。通过这两个枚举的排列组合我们要返回正确的SafariBookAccess。

```kotlin
fun access(employee: Employee,
           contract: Contract): SafariBookAccess
```

然后用定义SafariBooksAccess

```kotlin
sealed class SafariBookAccess

data class Granted(val expirationDate: DateTime) : SafariBookAccess()

data class NotGranted(val error: AssertionError) : SafariBookAccess()

data class Blocked(val message: String) : SafariBookAccess()
```

在access()方法中做排列组合

```kotlin
fun access(employee: Employee,
           contract: Contract): SafariBookAccess {
    return when (employee) {
        SENIOR_ENGINEER -> when (contract) {
            PROBATION -> NotGranted(AssertionError("Access not allowed on probation contract."))
            PERMANENT -> Granted(DateTime())
            CONTRACTOR -> Granted(DateTime())
        }
        REGULAR_ENGINEER -> when (contract) {
            PROBATION -> NotGranted(AssertionError("Access not allowed on probation contract."))
            PERMANENT -> Granted(DateTime())
            CONTRACTOR -> Blocked("Access blocked for $contract.")
        }
        JUNIOR_ENGINEER -> when (contract) {
            PROBATION -> NotGranted(AssertionError("Access not allowed on probation contract."))
            PERMANENT -> Blocked("Access blocked for $contract.")
            CONTRACTOR -> Blocked("Access blocked for $contract.")
        }
        else -> throw AssertionError()
    }
}
```

emm，这样写不够简洁，上述代码存在以下问题：

- 过多的when()方法。可以用Pair避免嵌套(nesting)
- 改变枚举类参数的顺序，使用Pair<Contract, Employee>()提高可读性
- 将返回相同的case合并
- 改为单表达式函数([Single-Expression functions](https://kotlinlang.org/docs/reference/functions.html#single-expression-functions))

然后，我们来改造一下这段代码

```kotlin
fun access(contract: Contract,
           employee: Employee) = when (Pair(contract, employee)) {
    Pair(PROBATION, SENIOR_ENGINEER),
    Pair(PROBATION, REGULAR_ENGINEER),
    Pair(PROBATION, JUNIOR_ENGINEER) -> NotGranted(AssertionError("Access not allowed on probation contract."))
    Pair(PERMANENT, SENIOR_ENGINEER),
    Pair(PERMANENT, REGULAR_ENGINEER),
    Pair(PERMANENT, JUNIOR_ENGINEER),
    Pair(CONTRACTOR, SENIOR_ENGINEER) -> Granted(DateTime(1))
    Pair(CONTRACTOR, REGULAR_ENGINEER),
    Pair(CONTRACTOR, JUNIOR_ENGINEER) -> Blocked("Access for junior contractors is blocked.")
    else -> throw AssertionError("Unsupported case of $employee and $contract")
}
```

现在看上去是不是很简洁了，但是还能更加简洁：

```kotlin
fun access(contract: Contract,
           employee: Employee) = when (contract to employee) {
    PROBATION to SENIOR_ENGINEER,
    PROBATION to REGULAR_ENGINEER -> NotGranted(AssertionError("Access not allowed on probation contract."))
    PERMANENT to SENIOR_ENGINEER,
    PERMANENT to REGULAR_ENGINEER,
    PERMANENT to JUNIOR_ENGINEER,
    CONTRACTOR to SENIOR_ENGINEER -> Granted(DateTime(1))
    CONTRACTOR to REGULAR_ENGINEER,
    PROBATION to JUNIOR_ENGINEER,
    CONTRACTOR to JUNIOR_ENGINEER -> Blocked("Access for junior contractors is blocked.")
    else -> throw AssertionError("Unsupported case of $employee and $contract")
}
```

希望这些语法糖能对你有所帮助，剩下的我们将在Part2中讲解。

转自 [Piotr Ślesarew @ Medium 6 magic sugars that can make your Kotlin codebase happier — Part 1](https://medium.com/grand-parade/6-magic-sugars-that-can-make-your-kotlin-codebase-happier-part-1-ceee3c2bc9d3)
编写 [snoopy@we1code.cn](http://we1code.cn/me)
