# Reducing amount of copying when passing collection elements to variadic parameters functions

This page is dedicated to researching the possibility of reducing the amount of copying performed when passing collection elements as parameters for functions with variadic arguments using spread operator `*`.

## Table of Contents
* [Motivation](#motivation)
  * [Double Copy when Using Spread on non-Array Collections](#double-copy-when-using-spread-on-non-array-collections)
  * [Additional Copying when Variadic Argument is Passed to Another Variadic Function](#additional-copying-when-variadic-argument-is-passed-to-another-variadic-function)
  * [Inconsistency in Calling Behaviour](#inconsistency-in-calling-behaviour)
  * [(*) Empty Array Allocation when Calling without Arguments](#-empty-array-allocation-when-calling-without-arguments)
* [Ideas](#ideas)
* [Discussions](#discussions)

## Motivation
Kotlin provides a special spread operator `*` for unpacking a collection and passing its elements as parameters for vararg functions,
which internally use *Java* arrays to store all the passed variadic arguments. Let's consider several cases where copying is performed when using spread operator.

Copying is not performed when using immediate arrays:
```kotlin
fun bar(vararg x: Int){
}

fun main() {
    val x = intArrayOf(1, 2, 3) 
    bar(*x) // Copy
    bar(*intArrayOf(1, 2, 3)) // No copy
}
```

Single arrays are copied using `Arrays.copyOf` method:
```kotlin
fun bar(vararg x: Int){
}

fun main() {
    val x = intArrayOf(1, 2, 3)
    bar(*x)
}
```
```java
...
ALOAD 0
ALOAD 0
ARRAYLENGTH
INVOKESTATIC java/util/Arrays.copyOf ([II)[I
INVOKESTATIC MainKt.bar ([I)V
...
```

[SpreadBuilder](https://github.com/JetBrains/kotlin/blob/5e81850bb12dd095dd8d94b5c9ded043e81caf7a/libraries/stdlib/jvm/runtime/kotlin/jvm/internal/SpreadBuilder.java#L13) is used for
passing several different arguments as the variadic parameter with at least one being passed using spread operator. Internally, it just copies the elements to a new array:
```kotlin
fun bar(vararg x: Int){
}

fun main() {
    val x = intArrayOf(1, 2, 3)
    bar(*x, *x)
    // Could be bar(1, *x) for example
}
```
```java
...
NEW kotlin/jvm/internal/IntSpreadBuilder
DUP
ICONST_2
INVOKESPECIAL kotlin/jvm/internal/IntSpreadBuilder.<init> (I)V
ASTORE 1
ALOAD 1
ALOAD 0
INVOKEVIRTUAL kotlin/jvm/internal/IntSpreadBuilder.addSpread (Ljava/lang/Object;)V
ALOAD 1
ALOAD 0
INVOKEVIRTUAL kotlin/jvm/internal/IntSpreadBuilder.addSpread (Ljava/lang/Object;)V
ALOAD 1
INVOKEVIRTUAL kotlin/jvm/internal/IntSpreadBuilder.toArray ()[I
INVOKESTATIC MainKt.bar ([I)V
...
```

We will consider several cases when usages 1 and 2 introduce unnecessary copying and overhead.
Let's take a look at some memory and copying problems that arise when using varargs:

### Double Copy when Using Spread on non-Array Collections
Consider the common case, when we are trying to pass a non-array collection to a vararg function with spread operator:
```kotlin
fun bar(vararg y: Any) {}

fun main() {
    val param = listOf(1, 2, 3)
    bar(*param.toTypedArray())
}
```
In this case we actually perform two copy operations. Take a look at the following *Java* bytecode:
```java
...
L4
LINENUMBER 14 L4
ALOAD 4
ICONST_0
ANEWARRAY java/lang/Integer
INVOKEINTERFACE java/util/Collection.toArray ([Ljava/lang/Object;)[Ljava/lang/Object; (itf) // copy in .toTypedArray()
L5
LINENUMBER 10 L5
CHECKCAST [Ljava/lang/Integer;
ASTORE 1
ALOAD 1
ALOAD 1
ARRAYLENGTH
INVOKESTATIC java/util/Arrays.copyOf ([Ljava/lang/Object;I)[Ljava/lang/Object; // copy in spread operator
INVOKESTATIC MainKt.bar ([Ljava/lang/Object;)V
...
```

Even though the `SpreadBuilder` supports non-array collections, it is banned on the compiler's front-end. So we still have to cast these collections to arrays with additional copying:
```kotlin
fun bar(vararg x: Int){
}

fun main() {
    val y = listOf(1, 2, 3)
    bar(1, *y.toIntArray())
}
```
```java
  NEW kotlin/jvm/internal/IntSpreadBuilder
  DUP
  ICONST_2
  INVOKESPECIAL kotlin/jvm/internal/IntSpreadBuilder.<init> (I)V
  ASTORE 1
  ALOAD 1
  ICONST_1
  INVOKEVIRTUAL kotlin/jvm/internal/IntSpreadBuilder.add (I)V
  ALOAD 1
  ALOAD 0
  CHECKCAST java/util/Collection
  INVOKESTATIC kotlin/collections/CollectionsKt.toIntArray (Ljava/util/Collection;)[I
  INVOKEVIRTUAL kotlin/jvm/internal/IntSpreadBuilder.addSpread (Ljava/lang/Object;)V
  ALOAD 1
  INVOKEVIRTUAL kotlin/jvm/internal/IntSpreadBuilder.toArray ()[I
  INVOKESTATIC MainKt.bar ([I)V
```

### Additional Copying when Variadic Argument is Passed to Another Variadic Function
See [KT-17043](https://youtrack.jetbrains.com/issue/KT-17043).

Sometimes, vararg functions directly delegate their argument to another variadic function. In this case, the argument is copied once again:
```kotlin
fun foo1(vararg x: Any) {
  bar(*x) // Copy
}
fun bar(vararg y: Any) {}
```
Such constructions ofter arise when using wrappers for *Java* APIs.

### Inconsistency in Calling Behaviour
The current state introduces inconsistency in calling functions:
```kotlin
fun foo(x: IntArray) {}
fun bar(vararg x: Int){}

fun main() {
    val x = intArrayOf(1, 2, 3)
    foo(x = x) // No copy here
    bar(x = x) // Copy
}
```
Both functions are called with the same syntax. However, `x` is copied when being passed to `bar` as opposed to `foo` call.

### (*) Empty Array Allocation when Calling without Arguments
See [KT-32474](https://youtrack.jetbrains.com/issue/KT-32474).

Vararg functions can be called without any arguments. In this case an empty array is created in *Java* bytecode.
The mentioned ticket proposes to avoid this allocation by using an empty array constant.

## Ideas
* Allow passing other collections using spread operator to avoid one copy operation (see [another proposal](comprehensions-proposal.md)).
* Do not perform copying of elements when passing vararg argument to spread operator (proposed [here](https://youtrack.jetbrains.com/issue/KT-17043/Do-not-create-new-arrays-for-pass-through-vararg-parameters#focus=Comments-27-2055092.0-0)):
  ```kotlin
  fun foo1(vararg x: Any) {
    bar(*x) // no copy
    val xx = x
    bar(*xx) // copy
  }
  fun bar(vararg y: Any) {}
  ```
  But this approach will break if `x` is changed via `y` reference, so we need to perform static analysis for this optimization.
* Do not perform copying when passing arrays explicitly as vararg parameters (proposed [here](https://youtrack.jetbrains.com/issue/KT-17043/Do-not-create-new-arrays-for-pass-through-vararg-parameters#focus=Comments-27-6070716.0-0)):
  ```kotlin
  fun bar(vararg x: Int){}
  
  fun main() {
      val x = intArrayOf(1, 2, 3)
      bar(x = x) // <- No copy here
  }
  ```
* Annotate methods that are guaranteed to return a new array that doesn't leak anywhere else and do not perform copying (as proposed in [this](https://youtrack.jetbrains.com/issue/KT-25350/Scalability-Issue-when-combining-collections-and-vararg-function-spread-operator#focus=Change-27-2963542.0-0) comment).
## Discussions
* [KT-2462](https://youtrack.jetbrains.com/issue/KT-2462) Varargs
* [KT-17043](https://youtrack.jetbrains.com/issue/KT-17043) Do not create new arrays for pass-through vararg parameters
* [KT-25350](https://youtrack.jetbrains.com/issue/KT-25350) Scalability Issue when combining collections and vararg function / spread operator
* [Kotlin-Discussions](https://discuss.kotlinlang.org/t/scalability-issue-spread-operator-with-collections/8466) Scalability Issue: Spread operator with collections
* [KT-27538](https://youtrack.jetbrains.com/issue/KT-27538) Avoid Arrays.copyOf when inlining a function call with vararg
* [KT-32474](https://youtrack.jetbrains.com/issue/KT-32474) Prevent empty array allocation when vararg is called without arguments
* [KT-9429](https://youtrack.jetbrains.com/issue/KT-9429) Change 'vararg' semantics to always copy
* [KT-22405](https://youtrack.jetbrains.com/issue/KT-22405) Extra array copies created on calling "string".format()