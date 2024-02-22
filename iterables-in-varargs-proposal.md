# Allowing `Iterable` collections passing to functions with variadic arguments using spread operator
This page is dedicated to researching the possibility of passing elements of `Iterable` collections as parameters for functions with variadic arguments using spread operator `*`.

See [KT-12663 Spread (*) operator for Iterable (Collection?) in varargs](https://youtrack.jetbrains.com/issue/KT-12663).

## Table of Contents
* [Motivation](#motivaton)
* [Current workarounds](#current-workarounds)
* [Discussions](#list-of-discussions)

## Motivaton
Kotlin provides a special construct for writing a functions with variadic number of arguments.
However, when the function is designed to receive elements from a single collections, developers prefer using a single parameter instead of varargs. 
This is due to the fact that varargs require unnecessary overhead, as the spread operator only works on Java arrays, thus the conversion from `Iterable` to `Array` is needed. 
Moreover, there is a mess regarding boxed `Array<T>` and named arrays (such as `IntArray`):
  ```Kotlin
  fun printAllInt(vararg ts: Int) {
      ts.forEach { println(it) }
  }
  
  fun main() {
      printAllInt(*arrayOf(1, 2, 3)) // Type mismatch, IntArray expected, Array<int> found
      printAllInt(*listOf(1, 2, 3)) // Type mismatch, IntArray expected, List<int> found
  }    
  ```

Moreover, inheritance with method overriding is broken in some cases.
It happens when substituting any primitive type, as the compiled method uses the corresponding array type:
* `IntArray`
* `BooleanArray`
* `ByteArray`
* `CharArray`
* `DoubleArray`
* `FloatArray`
* `LongArray`
* `ShortArray`
  ```Kotlin
  interface A<T> {
      fun foo(vararg x : T) // Array<T> expected
  }
  
  class B : A<Int> {
      override fun foo(vararg x: Int) { } // Error, method overrides nothing, IntArray expected
  }
  ```
  
## Current workarounds
* It is possible to use an `Iterable` collection with varargs functions, but it requires calling an explicit cast to the corresponding `Array` type and creates an additional copy, which is an additional problem (see [KT-17043 Do not create new arrays for pass-through vararg parameters](https://youtrack.jetbrains.com/issue/KT-17043)):
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
