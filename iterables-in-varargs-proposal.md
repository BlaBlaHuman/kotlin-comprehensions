# Allowing `Iterable` collections passing to functions with variadic arguments using spread operator
This page is dedicated to researching the possibility of passing elements of `Iterable` collections as parameters for functions with variadic arguments using spread operator `*`.

See [KT-12663 Spread (*) operator for Iterable (Collection?) in varargs](https://youtrack.jetbrains.com/issue/KT-12663).

## Table of Contents
* [Motivation](#motivaton)
* [Current Workarounds](#current-workarounds)
* [Discussions](#list-of-discussions)

## Motivaton
Kotlin provides a special construct for writing a functions with variable length parameters.
However, when the function is designed to receive elements from a single collections, developers prefer using a single collection parameter instead of varargs. 
This is due to the fact that varargs require unnecessary overhead for a number of reasons:
*  Spread operator, which unpacks a collection and passes its elements of a collection as varargs, only works on Java arrays, thus the conversion from `Iterable` to `Array` is needed (see the [Kotlin specification](https://kotlinlang.org/spec/expressions.html#spread-operator-expressions)). 
  ```Kotlin
  fun printAllInt(vararg ts: Int) {
      ts.forEach { println(it) }
  }
  
  fun main() {
      printAllInt(*intArrayOf(1, 2, 3))
      printAllInt(*listOf(1, 2, 3)) // Type mismatch, IntArray expected, List<int> found
  }    
  ```

  `SpreadBuilder` class that performs the elements copying of the array passed to spread operator already supports `Iterable` collections, but this is banned on the compiler's frond-end
(see [SpreadBuilder](https://github.com/JetBrains/kotlin/blob/5e81850bb12dd095dd8d94b5c9ded043e81caf7a/libraries/stdlib/jvm/runtime/kotlin/jvm/internal/SpreadBuilder.java#L13) from the **Kotlin** repo).


* Additionally, inheritance with method overriding is broken in some cases. 
This is because vararg functions use *Java* `Array` to store all the passed variadic arguments.
When the type of variadic arguments is explicitly stated as any primitive type, the compiled method uses the corresponding array type:
  * `IntArray`
  * `BooleanArray`
  * `ByteArray`
  * `CharArray`
  * `DoubleArray`
  * `FloatArray`
  * `LongArray`
  * `ShortArray`

  However, functions with templated varargs use boxed arrays under the hood.
  This introduces inconsistency and a lot of type issues. 
  ```kotlin
  fun printAllInt(vararg ts: Int) { // Uses IntArray
    ts.forEach { println(it) }
  }
  
  fun <T>printAllGen(vararg ts: T) { // Uses Array<T>
      ts.forEach { println(it) }
  }
  
  fun main() {
      printAllInt(*intArrayOf(1, 2 ,3)) // OK
      printAllInt(*arrayOf(1, 2 ,3)) // Type mismatch, IntArray expected, Array<Int> found
      printAllInt(*listOf(1, 2, 3).toIntArray()) // OK
  
      printAllGen(*intArrayOf(1, 2 ,3)) // Type mismatch, Array<Int> expected, IntArray found
      printAllGen(*arrayOf(1, 2 ,3)) // OK
      printAllGen(*listOf(1, 2, 3).toTypedArray()) // OK
  }
  ```
  
  Templated vararg methods also cannot be overridden by any primitive type substitution due to the same reason.
  ```Kotlin
  interface A<T> {
      fun foo(vararg x : T) // x has type Array<T> 
  }
  
  class B : A<Int> {
      override fun foo(vararg x: Int) { } // Error, method overrides nothing, x has type IntArray
  }
  ```
  
## Current Workarounds
* It is possible to use an `Iterable` collection with varargs functions, but it requires calling an explicit cast to the corresponding `Array` type and creates a copy, which is an additional problem (see [KT-17043 Do not create new arrays for pass-through vararg parameters](https://youtrack.jetbrains.com/issue/KT-17043)):
  ```Kotlin
  fun printAllInt(vararg ts: Int) {
      ts.forEach { println(it) }
  }
  
  fun main() {
      printAllInt(*listOf(1, 2, 3).toIntArray()) 
  } 
  ```

* As for the overriding, the most obvious and simple workaround is to create a dummy method that accepts data and then calls the "real" overriden method with this data casted to array.
  ```Kotlin
  abstract class A<T> {
      abstract fun broken(vararg values: T): String
  }
  
  class B : A<Int>() {
      override fun broken(values: Array<out Int>): String {
          return broken(*values.toIntArray())
      }
  
      fun broken(vararg values: Int): String {
          return values.joinToString { it.toString() }
      }
  }
  ```

* One of the [proposed workarounds](https://discuss.kotlinlang.org/t/scalability-issue-spread-operator-with-collections/8466) to avoid copying is to call a utility method in *Java* that calls the original vararg function:
  ```Java
  // Java
  public class VarargUtil {
    public static void passArrayAsVarargs(@NotNull String[] ids) {
        MainKt.processIds(ids); //processIds accepts variable arguments
    }
  }
  ```
  ```kotlin
  // Kotlin
  fun processIds(vararg ids: String) {
    ...
  }

  fun main() { 
    val ids: List<String> = listOf("...")
    passArrayAsVarargs(ids)
  }
  ```

The mentioned overhead and issues prevent users from having a smooth experience and are the reason why people tend not to use varargs in their code and why this feature is not widely used.

## List of Discussions
- [KT-2462](https://youtrack.jetbrains.com/issue/KT-2462) Varargs
- [KT-12663](https://youtrack.jetbrains.com/issue/KT-12663) Spread (*) operator for Iterable (Collection?) in varargs
- [KT-6846](https://youtrack.jetbrains.com/issue/KT-6846) Spread operator doesn't work for boxed arrays (e.g. Array)
- [KT-25350](https://youtrack.jetbrains.com/issue/KT-25350) Scalability Issue when combining collections and vararg function / spread operator
- [KT-9471](https://youtrack.jetbrains.com/issue/KT-9471) Spread operator should be able to mix IntArray and Array
- [KT-9495](https://youtrack.jetbrains.com/issue/KT-9495) vararg and substitution of primitives for type parameters&
- [KT-27013](https://youtrack.jetbrains.com/issue/KT-27013) Spread operator doesn't work on primitive type vararg parameters
- [KT-47711](https://youtrack.jetbrains.com/issue/KT-47711) Type mismatch when unpacking IntArray and re-packing to Array
- [KT-19840](https://youtrack.jetbrains.com/issue/KT-19840) Spread operator doesn't work for non-primitive arrays
- [KT-26146](https://youtrack.jetbrains.com/issue/KT-26146) Unable to override generic function with "primitive" vararg type parameter
- [KT-30837](https://youtrack.jetbrains.com/issue/KT-30837) Confusing error message for passing a list/collection to `spread` operator
- [KT-64324](https://youtrack.jetbrains.com/issue/KT-64324) Enum entries should be spreadable
- [Kotlin-Discussions | Scalability Issue: Spread operator with collections](https://discuss.kotlinlang.org/t/scalability-issue-spread-operator-with-collections/8466)
