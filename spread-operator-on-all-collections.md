# Allowing using *Spread operator (\*)* on all **Iterable** collections, regardless of the type of the *vararg* array

* **Type**: Design Proposal
* **Author**: Oleg Makeev
* **Prototype**: In Progress
* **Discussion**: [KT-2462](https://youtrack.jetbrains.com/issue/KT-2462)

## Abstract

Currently, **Kotlin** puts a constraint on using the *Spread operator (\*)* only on arrays.
Moreover, the passed array's type must match the type of the `vararg` array (`Array<out T>`, `IntArray`, etc.).
As arrays are not widely used in **Kotlin**, this constraint leads to a lot of overhead due to the need of casts and copying.

This proposal suggests allowing using the *Spread operator* on all collections, regardless of the type of the *vararg* array.

## Table of contents

* [Abstract](#abstract)
* [Table of contents](#table-of-contents)
* [Background and motivation](#background-and-motivation)
    * [Variadic functions](#variadic-functions)
    * [Desugaring to arrays](#desugaring-to-arrays)
    * [Spread operator](#spread-operator)
    * [Variadic functions in standard library](#variadic-functions-in-standard-library)
* [Current workarounds](#current-workarounds)
* [Technical details](#technical-details)
    * [Argument type checking](#argument-type-checking)
    * [VarargLowering stage](#vararglowering-stage)
* [Prototype steps](#prototype-steps)
* [Results and performance evaluation](#results-and-performance-evaluation)
    * [Results](#results)
    * [Benchmarks](#benchmarks)
        * [Compilation](#compilation)
        * [Execution](#execution)
* [Potential extensions](#potential-extensions)
    * [Alternative underlying collection](#alternative-underlying-collection)
    * [Keyword variadics](#keyword-variadics)

## Background and motivation

### Variadic functions
*Kotlin* provides a special syntax for creating functions with variadic number of arguments.
It is possible to create a variadic parameter by using the `vararg` keyword in the function signature right before this parameter.

```Kotlin
fun printElements(vararg elements: Int) {
    elements.forEach { print(it) }
}

fun main() {
    printElements(1, 2, 3) // Prints 123
}
```

The only constraint on the variadic function signature is that there can be only one `vararg` parameter.

### Desugaring to arrays

All the variadic parameters are desugared to arrays in the compiled code, which contain all the passed arguments.
The type of the array depends on the type of the variadic parameter (see the [Kotlin Specification](https://kotlinlang.org/spec/declarations.html#variable-length-parameters)).

* Parameters of primitive types are desugared to primitive arrays (e.g. `Int` parameter is converted to `IntArray`)
* All other types are desugared to boxed arrays with `out` projection `Array<out T>`

The need for primitive types is motivated by the performance reasons.
When targeting `JVM` platform, primitive arrays avoid wrapping and unwrapping of primitive types (`IntArray` in **Kotlin** is turned into `int[]` in **Java**).
If we were to use boxed arrays with primitve types, these types would be wrapped into primitive wrappers (`Integer`, `Boolean`, etc., see the [**Java** documentation](https://docs.oracle.com/javase/tutorial/java/data/autoboxing.html)).
This means that, for example, `Array<out Int>` in **Kotlin** is turned into `Integer[]` in **Java**, which introduces unnecessary overhead.

This approach already implies some limitations. 
For example, it is impossible to override a generic variadic method using primitive type ([KT-9495](https://youtrack.jetbrains.com/issue/KT-9495)). 

On the code snippet below there is a generic interface `A<T>` with a generic function `foo` that has a single generic parameter `x`.
Derived class `B` tries to override `foo` with a primitive type `Int`, but the compiler doesn't consider it as an override, as the function signature differs.
```Kotlin
interface A<T> { 
    fun foo(vararg x : T) // foo(x: Array<out T>)
}

class B : A<Int> {
    override fun foo(vararg x: Int) {} // foo(x: IntArray) 
    // Error, method overrides nothing
}
```

### Spread operator

The *spread operator (\*)* is a special operator that allows unwrapping existing collections on the call site and passing the collection elements to variadic functions.

```Kotlin
fun printElements(vararg elements: Int) {
    elements.forEach { print(it) }
}

fun main() {
    val x = intArrayOf(1, 2, 3)
    val y = intArrayOf(4, 5, 6)
    printElements(*x) // Prints 123
    printElements(*x, *y) // Prints 123456
    printElements(0, *x, *y, 7) // Prints 01234567
}
```

Such spread arguments can be mixed with other non-spread and spread arguments.

Currently, spread operator is only allowed on arrays whose type matches the type of the `vararg` array.
```Kotlin
fun printElements(vararg elements: Int) {
    elements.forEach { print(it) }
}

fun main() {
    val x = listOf(1, 2, 3)
    printElements(*x) 
    // Type mismatch:
    // inferred type is List<Int> but IntArray was expected
}
```

This limitation leads to the need of calling casts on non-array types.
These casts perform copying of the collection elements to the new array, which worsens the performance.

```Kotlin
fun printElements(vararg elements: Int) {
    elements.forEach { print(it) }
}

fun main() {
    val x = listOf(1, 2, 3)
    printElements(*x.toIntArray()) // OK, prints 123
}
```

[Sourcegraph](https://sourcegraph.com/search) is a tool that allows searching for regular expressions across all open repositories.
Using **Sourcegraph**, it was found out that:

* There were 69,574 usages of spread operator without any casts on the call site
* 7,830 more usages used casts to boxed array (`toTypedArray()`)
* 272 more usages used primitive array casts

### Variadic functions in standard library

Variadic functions a particularly interesting, as they are actively utilized by the **Kotlin** standart library.
They are mainly used for instantiating collections:
```kotlin
public fun <T> listOf(vararg elements: T): List<T> =
    if (elements.size > 0) elements.asList() 
    else emptyList()
```

Functions for initializing primitive arrays directly rely on the compiler to do all the job:
```kotlin
public inline fun intArrayOf(vararg elements: Int): IntArray = elements
```

## Current workarounds
* It is possible to use an `Iterable` collection with varargs functions, but it requires calling an explicit cast to the corresponding `Array` type and creating an additional copy:
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
      abstract fun foo(vararg x: T)
  }
  
  class B : A<Int>() {
      override fun foo(x: Array<out Int>) {
          foo(*x.toIntArray())
      }
  
      fun foo(vararg x: Int) {
          x.joinToString { it.toString() }
      }
  }
  ```

* One of the [proposed workarounds](https://discuss.kotlinlang.org/t/scalability-issue-spread-operator-with-collections/8466) to avoid copying and to is to call a utility method in **Java** that calls the original vararg function:
  ```Java
  // Java
  public class VarargUtil {
    public static void passArrayAsVarargs(@NotNull int[] ids) {
        MainKt.processIds(ids); // passes by reference
    }
  }
  ```
  ```kotlin
  // Kotlin
  fun processIds(vararg ids: Int) {
  }
  
  fun main() {
    val ids: IntArray = intArrayOf(1, 2, 3)
    passArrayAsVarargs(ids)
  }
  ```


## Technical details

The whole current implementation can be split into three major steps:
1. **Variadic function signature transformation** ---
On this step, the original type of `vararg` parameter is replaced with the corresponding array type.
It is done using `FirTypeResolveTransformer` on the `TYPES` resolution phase.

2. **Function call candidates gathering** ---
This step done on `BODY_RESOLVE` phase for each function call.
For each potential candidate, several checks are performed.
One of these checks is argument type checking, which is done using `CheckArguments` checker stage.
It compares the type of variadic arguments against the type of the `vararg` parameter (for single elements) or the type of `vararg` array (for spread arguments).

3. **Vararg lowering phase** ---
On the backend part, several lowering phases are executed, which desugar higher order concepts.
On `VarargLowering` phase, all the passed variadic arguments are copied into one unified array on the call site.
After that, the `vararg` parameter is finally desugared to a regular array parameter. 



### Argument type checking
Currently, all the spread arguments' types are checked against the type of the `vararg` array.
However, this is the main source of all the issues. 
Moreover, it provides a strange and inconvenient message:
```
Type mismatch:
inferred type is List<Int> but IntArray was expected
```

The prototype resolves this problem by type checking spread arguments only by the type of their elements.
This was done by introducing several special utilitiy functions.
These functions are able to unwrap any array type and all `Iterable` collections.

Type incompatibility errors are now more informative and clear:
```
Type mismatch:
inferred type is String but Int was expected
```

### `VarargLowering` stage
On the `VarargLowering` phase, a special `IrArrayBuilder` is used to add array initialization to the generated IR.
It utilizes two additional builders:
* `PrimitiveSpreadBuilder` --- is an interface, used for primitive arrays.
It has a concrete implementation for every primitive type.
* `SpreadBuilder` --- is a class used for non-primitive arrays.

These helper builders expose several methods: `add`, `addSpread` and `toArray`.
First two methods are used for adding single elements and spread arguments to the array.
The last method is responsible for finalizing the array and returning it.
`IrArrayBuilder` chooses the right helper builder and then generates IR calls to the constructor, `add`/`addSpread` for each argument and `toArray`.

These builders were extended to support all types of spread arguments.
In fact, `SpreadBuilder` already supported spread arguments of all types, except for primitive arrays.

## Results and performance evaluation

### Results
As a result, the prototype allows using spread operator on both arrays and `Iterable` collections.
Moreover, each collection is copied only once (on the `VarargLowering` stage), as compared to the original compiler, which required additional casts.

Now it is possible to compile and execute the following code:

```Kotlin
fun fooInt(vararg x: Int) {}

fun fooString(varargx x: String) {}

fun main() {
  val intPrimitiveArray = intArrayOf(1, 2, 3)
  val intList = listOf(4, 5, 6)
  val intArray = arrayOf(7, 8, 9)
  fooInt(*intPrimitiveArray, *intList, *intArray, 10, 11, 12)
  
  val stringList = listOf("a", "b", "c")
  val stringArray = arrayOf("d", "e", "f")
  fooString(*stringList, *stringArray, "g", "h", "i")
}
```

### Benchmarks
The benchmarks’ suite consisted of a set of various large `.kt` files.
The files that didn't use variadic functions were used for benchmarking both the original compiler and our prototype.
As for variadic functions, a special script was written that generates a blank variadic function, and then in the `main()` function it generates a number of random collections with arbitrary elements. 
Then it inserts calls to this one variadic function passing previously initialized collections via spread operator along with random single elements. 

It's important to note that for the prototype, spread operator was used directly on the created collections without any casts. 
However, for the original compiler these files were copied and changed such that there is a cast inserted for each collection on the call site. 

Example of benchmark file for the prototype:
```kotlin
fun callMe0(vararg a: Int) {
    println(a)
}

fun main() {
    val var0 = setOf(901, 272, 869, 69, 597, 300, 207, 414, 922, 390)
    val var1 = listOf(616, 937, 123, 762, 483)
    val var2 = mutableListOf(947, 449, 994, 913)
    val var3 = arrayOf(635, 611, 128, 275, 939, 878)
    // More collections here
    
    callMe0(*var0, *var1, *var2, *var3, 305, 35, 140, 504, 474, 213, 917)
    // More calls here
}
```

The same file version for the original compiler:
```kotlin
fun callMe0(vararg a: Int) {
    println(a)
}

fun main() {
    val var0 = setOf(901, 272, 869, 69, 597, 300, 207, 414, 922, 390)
    val var1 = listOf(616, 937, 123, 762, 483)
    val var2 = mutableListOf(947, 449, 994, 913)
    val var3 = arrayOf(635, 611, 128, 275, 939, 878)
    // More collections here
    
    callMe0(
        *var0.toIntArray(),
        *var1.toIntArray(),
        *var2.toIntArray(),
        *var3.toIntArray(),
        305, 35, 140, 504, 474, 213, 917)
    // More calls here
}
```

On average, there were ~30 collection variables and ~100 calls to variadic functions. 
Each call contained random([1, 30]) spread arguments and random([1, 20]) single arguments.

The benchmark performed 5 warmup runs for each file and then 10 normal runs. 
For each file it calculated the average measured time. 

It turned out that for files without variadic functions the compilation and performance benchmarks show equal results.
That's why we are going to only discuss the benchmarks performed on generated files with variadic function calls.

#### Compilation
The compilation was done by running `kotlinc test.kt` command.

* The average compilation time for the original compiler was **3.2** seconds
* The prototype showed the average compilation time of **2.6** seconds (18.75% faster)

#### Execution
The execution benchmark was done by running `kotlinc file.kt -include-runtime -d file.jar` command and then measuring the time of running `java -jar file.jar`.

* The average execution time for the original compiler was **5.88 * 10^(-2)** seconds
* The prototype showed the average execution time of **5.68 * 10^(-2)** seconds (3.4% faster)
`

## Prototype steps
✅ Rewrite typechecking pipeline

✅ Rewrite `VarargLowering` stage

✅ Run builtin tests

✅ Benchmark the performance

✅ Add function resolution tests

❌ Rewrite typechecking pipeline used for diagnostics

❌ Add type checking tests

## Potential extensions

### Alternative underlying collection
The variadic functions still feel out of place due to the usage of non-native **Kotlin** arrays. 
It would be more natural to use `Iterable` as the underlying type for variadic functions.
However, it's not that easy, as we have to preserve the compatibility with the existing code base and **Java**.

### Keyword variadics

The idea of keyword variadic arguments is a rare feature, that is present is some languages like **Python**.
In **Python**, the keyword variadic parameter is marked with double asterisks `**` before the parameter name in the function signature.
It allows passing a variadic number of arguments via named form and accessing them inside the function by the keyword string.
Functions in **Python** can have both variadic positional and keyword arguments.
However, on the call site, the positional and keyword arguments should be in the same order as they are in the function signature.

Double asterisks `**` are also used as a spread operator for unwraping dictionaries on the call site.

It is also important to note that inside the function, the keyword variadic parameter has a type of `dict` (the same as `Map` in **Kotlin**).
```python
def foo(**kwargs): # type(kwargs) == <class 'dict'>
    for key, value in kwargs.items():
        print(f"{key} = {value}")


foo(a=1, b=2, c=3)

dict = {"a": 1, "b": 2}
foo(**dict)
```

The same functionality could be added to Kotlin.

```kotlin
fun foo(kvararg x: Int) { // x has type Map<String, Int>
    x.forEach { (key, value) -> println("$key = $value") }
}


foo(a = 1, b = 2, c = 3)

val x = "someKeyword"
foo(*x = 1)
```

Such an approach seems easy to implement, as main ideas are similar to the ones used for positional variadic arguments:
* The function signature is transformed to accept a single `Map` parameter
* The argument mapping and type checking is performed
* The `VarargLowering` stage is used to copy all the passed named arguments into one `Map` object

However, there we don't have to preserve any special compatibility with **Java**, as it doesn't support named arguments.

Such an approach could be beneficial in many cases, e.g. when working with CLI arguments or with key-value databases.