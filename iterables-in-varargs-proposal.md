# Allowing `Iterable` collections passing to functions with variadic arguments using spread operator
This page is dedicated to researching the possibility of passing elements of `Iterable` collections as parameters for functions with variadic arguments using spread operator `*`.

See [KT-12663 Spread (*) operator for Iterable (Collection?) in varargs](https://youtrack.jetbrains.com/issue/KT-12663).

## Table of Contents
* [Motivation](#motivaton)
* [Current workarounds](#current-workarounds)

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