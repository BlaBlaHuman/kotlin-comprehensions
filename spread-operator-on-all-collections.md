# Allowing using *Spread operator (\*)* on all **Iterable** collections, regardless of the type of the *vararg* array

* **Type**: Design Proposal
* **Author**: Oleg Makeev
* **Prototype**: In Progress
* **Discussion**: [KT-2462](https://youtrack.jetbrains.com/issue/KT-2462)

## Abstract

Currently, in **Kotlin** the usage of *Spread operator (\*)* is limited to array types.
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
    * [Additional issues](#additional-issues)
* [Variadic functions in other programming languages](#variadic-functions-in-other-programming-languages)
    * [C](#c)
    * [Python](#python)
    * [JavaScript](#javascript)
    * [Java](#java)
* [Current workarounds](#current-workarounds)
* [Current implementation](#current-implementation)
    * [Variadic function signature transformation](#variadic-function-signature-transformation)
    * [Function call candidates gathering](#function-call-candidates-gathering)
    * [Vararg lowering phase](#vararglowering-stage)
* [Proposed implementation](#proposed-implementation)
    * [Argument type checking](#argument-type-checking)
    * [`VarargLowering` stage](#vararglowering-stage-1)
* [Prototype steps](#prototype-steps)
* [Results and performance evaluation](#results-and-performance-evaluation)
    * [Results](#results)
    * [Benchmarks](#benchmarks)
        * [Compilation](#compilation)
        * [Execution](#execution)
    * [Notes on interop with Java](#notes-on-interop-with-java)
* [Alternative approaches](#alternative-approaches)
    * [Artificialy inserting needed cast in FIR](#artificialy-inserting-needed-cast-in-fir)
    * [Using boxed arrays only](#using-boxed-arrays-only)
    * [Deprecating `vararg` keyword and moving to collection literals](#deprecating-vararg-keyword-and-moving-to-collection-literals)
* [Potential extensions](#potential-extensions)
    * [Alternative underlying collection](#alternative-underlying-collection)
    * [Expand spread operator usage to objects](#expand-spread-operator-usage-to-objects)
    * [`dataarg`](#dataarg)
    * [One-or-more varargs](#one-or-more-varargs)
    * [Keyword variadics](#keyword-variadics)
* [Related discussions](#related-discussions)

## Background and motivation

### Variadic functions
*Kotlin* provides a special syntax for creating functions with variadic number of arguments.
It is possible to create a variadic parameter by using the `vararg` keyword in the function signature right before this parameter's type.

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

All the variadic parameters are desugared to arrays, which contain all the passed arguments.
The type of the array depends on the type of the variadic parameter (see the [Kotlin Specification](https://kotlinlang.org/spec/declarations.html#variable-length-parameters)).

* Parameters of primitive types are desugared to primitive arrays (e.g. `Int` parameter is converted to `IntArray`)
* All other types are desugared to boxed arrays with `out` projection `Array<out T>`

The need for primitive types is motivated by the performance reasons.
When targeting `JVM` platform, primitive arrays avoid wrapping and unwrapping of primitive types (`IntArray` in **Kotlin** is turned into `int[]` in **Java**).
If we were to use boxed arrays with primitve types, these types would be wrapped into primitive wrappers (`Integer`, `Boolean`, etc., see the [**Java** documentation](https://docs.oracle.com/javase/tutorial/java/data/autoboxing.html)).
This means that, for example, `Array<out Int>` in **Kotlin** is turned into `Integer[]` in **Java**, which introduces unnecessary overhead.

There are 8 types of primitive arrays in **Kotlin**:
  * `IntArray`
  * `BooleanArray`
  * `ByteArray`
  * `CharArray`
  * `DoubleArray`
  * `FloatArray`
  * `LongArray`
  * `ShortArray`

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

The *spread operator (\*)* is a special operator that allows unwrapping existing collections on the call site and passing the elements from these collections to variadic functions.

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

This leads to a lot of boilerplate cast calls.
A number of iterable types in **Kotlin** doesn't support direct casting to arrays (e.g. progression types), so the user has to call an additional cast, which results in two or more unnecessary copies.
```kotlin
fun foo(vararg x: Int) {
}

fun main() {
    val prog: IntProgression = 1..10
    foo(*prog.toList().toIntArray()) // IntProgression cannot be directly cast to IntArray
}
```

[Sourcegraph](https://sourcegraph.com/search) is a tool that allows searching for regular expressions across all open repositories.
Using **Sourcegraph**, it was found out that:

* There were 69,574 usages of spread operator without any casts on the call site
* 7,830 more usages used casts to boxed array (`toTypedArray()`)
* 272 more usages used primitive array casts

### Variadic functions in standard library

Variadic functions a particularly interesting, as they are actively utilized by the **Kotlin** standard library.
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

### Additional issues

* [KT-20113](https://youtrack.jetbrains.com/issue/KT-20113) Spread operator not working on vararg indexed access operator
  
  The issue is connected to the resolution of operator calls.
  The square brackets notation for `get` operator doesn't support nested spread operator.
  ```kotlin
  class A {
      operator fun get(vararg index: Int): Int {
          return 0
      }
  }
  
  val a = A()
  val b = a[1, 2, 3] // OK
  val c = intArrayOf(1, 2, 3)
  val d = a.get(*c) // OK
  val e = a[c] // Error: Kotlin: Type mismatch: inferred type is IntArray but Int was expected
  val f = a[*c] /* Error: Kotlin: None of the following functions can be called with the arguments supplied: 
  public final operator fun times(other: Byte): Int defined in kotlin.Int
  public final operator fun times(other: Double): Double defined in kotlin.Int
  public final operator fun times(other: Float): Float defined in kotlin.Int
  public final operator fun times(other: Int): Int defined in kotlin.Int
  public final operator fun times(other: Long): Long defined in kotlin.Int
  public final operator fun times(other: Short): Int defined in kotlin.Int
  */
  ```
  
* [KT-62746](https://youtrack.jetbrains.com/issue/KT-62746/Kotlin-Native-allows-to-overload-by-varargs.-Kotlin-JVM-doesnt-allow-it) Kotlin/Native allows to overload by `vararg`s. Kotlin/JVM doesn't allow it
  
  Currently, there is an inconsistency between **Kotlin/JVM** and **Kotlin/Native** platforms.
  The issue is connected to the way the compiler handles variadic function signature and calls resolution.
  In **Kotlin/Native** it is possible to override a function with a regular array parameter using `vararg` parameter, while **Kotlin/JVM** prohibits it, as the signatures of two functions in **JVM** are identical.
  ```kotlin
  // Runs on Native, fails on JVM
  
  fun foo(bar: IntArray) {
    println("array")
  }
  
  fun foo(vararg bar: Int) {
      println("vararg")
  }
  
  fun main() {
      foo(1, 2) // vararg
      foo(intArrayOf(1, 2)) // array
  }
  ```
## Variadic functions in other programming languages

Early languages didn't formally introduce the concept of variadic functions and sometimes didn't require any function declarations in advance. 
The example of such a language is **B**, the predecessor of **C** (see the [B documentation](https://www.thinkage.ca/gcos/expl/b/manu/manu.html#Section4_3)). 
Since the function call has no information about the number of arguments, it allows passing a variable number of parameters.
Inside the function, one can use a special function `nargs()` that returns the number of passed arguments to the function. 
It's needed to fill in default values for the parameters that were not initialized.
Otherwise, if the number of passed arguments is greater than the number of parameters, all the surplus arguments are discarded.

In modern languages that don't have a dedicated mechanism for variadic functions, variadic behaviour could be mimicked by overloading a function for each required number of arguments.
It requires building a call chain from functions with fewer parameters towards the main function of maximum arity, containing all the required logic.

```kotlin
fun sum(x1: Int): Int = sum(x1, 0)

fun sum(x1: Int, x2: Int): Int = sum(x1, x2, 0)

fun sum(x1: Int, x2: Int, x3: Int): Int = x1 + x2 + x3
```

However, this approach is not really scalable and efficient, as it contains a lot of boilerplate code and its usage is limited to the maximum number of parameters for which the user has implemented the call chain.
So the concept of variadic functions started to become a widely used native feature in many programming languages. 

### C
**C** is a classical compiled low-level programming language.
It has one of the first and most known appearances of variadic functions, which is [`printf`](https://cplusplus.com/reference/cstdio/printf/) function that takes a formatting string and arbitrary number of arguments and then prints the arguments according to the formatting string. 

Variadic functions in **C** are marked with three dots (`...`) in the parameter list right after the last named parameter.
One can access variadic arguments inside the function using several special types and macros (see [doc](https://cplusplus.com/reference/cstdarg/)) based on the pointer arithmetic.
However, as there is no way to calculate the number of passed arguments, a dedicated parameter for this purpose is needed.
In the case of `printf` function, the number of arguments expected is derived from the format string itself.

```C
#include <stdarg.h>
#include <stdio.h>

double sum(int count, ...) {
    va_list ap;
    int j;
    double sum = 0;

    va_start(ap, count); 
    for (j = 0; j < count; j++) {
        sum += va_arg(ap, int); 
    }
    va_end(ap);

    return sum;
}

int main() {
    printf("%f\n", sum(3, 1, 2, 3));
    return 0;
}
```

`va_list` is a dedicated type definition for the type that holds all the required information regarding the variadic parameter of the current function. 
The usage of variadic parameters is based on several function macros: 

* `va_start(list, par)` --- takes a `va_list` object and a reference to the function's last parameter.
In the latest language standard, the second parameter is no longer required.
It initialises the `va_list` object for use by `va_arg` or `va_copy`
* `va_arg(list, type)` --- takes two parameters, a `va_list` object and a type descriptor.
It expands to the next variable argument, and has the specified type.
Successive invocations of `va_arg` allow processing each of the variable arguments in turn.
Unspecified behavior occurs if the type is incorrect or there is no next variable argument.
* `va_end(list)` --- takes one parameter, a `va_list` object, and resets it.
For example, to scan the variable arguments multiple times, the programmer would need to re-initialize the `va_list` object by calling `va_end` and then `va_start` again on it.
* `va_copy(list1, list2)` --- takes two parameters, both of them `va_list` objects.
It clones the second (which must have been initialised) into the first.

However, the `printf` function was considered dangerous by many experts for its vulnerability called [format string attack](https://cs155.stanford.edu/papers/formatstring-1.2.pdf).
After the invocation, the function puts all the elements on the stack and parses the format string and searches for `%_` specifiers in the string to embed arguments.
Then it pops the element from the stack and prints it.
However, many developers made a mistake by writing `printf(buffer)` instead of `printf("\%s", buffer)` in order to print the `buffer` string.
In this case, `printf` considers the passed string as a format string.
If it contains an element embedding `%_` it tries to pop an element from the stack, which is not presented, so it pops further.
This hack allows attackers to access the program stack and change the stored data. 


### Python
**Python** is one of the most popular languages nowadays.
It's a high-level general-purpose interpreted language. 
It has two options for variadic parameters:

* Non-keyword variadic parameters.
These are marked with an *asterisk (\*)* before the parameter name in the function declaration.
Inside the function, users can access the passed arguments via the variadic parameter name.
The type of the parameter is tuple.
  ```python
  def foo(*args): # type(args) == <class 'tuple'>
    for element in args:
      print(element, end='')
  
  foo(1, 2, 3) # prints 123
  ```
* Keyword variadic parameters.
Such parameters are marked with double *asterisk (\*\*)*. It allows passing named arguments to the function.
The type of such a parameter inside the function is a dictionary
  ```python
  def foo(**kwargs): # type(kwargs) == <class 'dict'>
    print(kwargs["a"]) // hello
    print(kwargs["b"]) // 1
    
  foo(a = "hello", b = 1)
  ```
  
Two mentioned approaches can be used together.
But named and non-named arguments have to be in the same order as the parameters defined in the function signature.

**Python** also provides a way to pass existing collections to these parameters.
To unpack a collection to a non-named parameter, a single *asterisk (\*)* is used. 
For named arguments, the *double asterisks (\*\*)* operator is used.
```python
def foo(*args, **kwargs):
    print(args) # (0, 1, 2, 3)
    print(kwargs) # {'a': 1, 'b': 2, 'c': 3}

list = [1, 2]
dict = {"a": 1, "b": 2}
foo(0, *list, 3, **dict, c = 3)
```

The presence of two types of variadic parameters is quite convenient for writing function wrappers:
```python
def decorator(old):
    def new(*args, **kwargs):
        # ...
        return old(*args, **kwargs)
    return new
```

### JavaScript

In **JavaScript** variadic parameters are called [*rest* parameters](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Functions/rest_parameters).
Such a parameter can be only one in the function signature and must be on the last position.
Rest parameters are marked with three dots `...` right before the name of the parameter.
These parameters are desugared to arrays containing all the passed arguments.

```javascript
function sortRestArgs(param1, param2, ...theArgs) {
  const sortedArgs = theArgs.sort();
  return sortedArgs;
}
```

**JavaScript** additionally provides a [spread operator](https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/Spread_syntax) that is also marked with three dots `...` before the collection.
This operator is somewhat different from similar operators in other languages.
This operator is placed in front of an iterable collection and allows unpacking its elements in three cases:
* On functions call site `myFunction(a, ...iterableObj, b)`
* In array literals `[1, ...iterableObj, '4', 'five', 6]`
* In object literals `{ ...obj, key: 'value' }`

The spread operator provides more handy and flexible ways to work with collections in general.

For example, the array concatenation can be done without calling `Array.prototype.concat()` function:
```javascript
let arr1 = [0, 1, 2];
const arr2 = [3, 4, 5];

arr1 = [...arr1, ...arr2];
// Instead of
arr1 = arr1.concat(arr2);
```

It is also possible to conditionally add values to arrays:
```javascript
const isSummer = false;
const fruits = ["apple", "banana", ...(isSummer ? ["watermelon"] : [])];
// ['apple', 'banana']
```

Spread elements can be mixed with other spread and non-spread elements in all three cases.

```javascript
function foo(...theArgs) {
  ..
}

const arr1 = [1, 2, 3];
const arr2 = [4, 5, 6];
foo(...arr1, ...arr2, 7, 8, 9);
```


### Java

In **Java**, each function can have at most one variadic parameter, and it has to be in the last position in the function signature.
Such a parameter is marked with *triple dots (...)* after the parameter type.

```Java
public class Main {
  public static void printAll(int... things){
     for(Object i:things){
        System.out.print(i);
     }
  }
  
  public static void main(String[] args) {
    printAll(1, 2, 3); // prints 123
  }
}
```

It is also possible to pass elements of an existing array to the variadic parameters.
However, this can only be done for one array per call without the possibility for any other arguments to be passed.

```Java
public class Main {
  public static void printAll(int... things){
     for(Object i:things){
        System.out.print(i);
     }
  }

  public static void main(String[] args) {
    int[] arr = {1,2,3};
    printAll(arr); // prints 123
  }
}
```

In such cases, arrays are passed by reference, just like any other normal argument.
Inside the function, all the elements are accessed using the vararg parameter, which has an array type with elements being of the type of the variadic parameter.

Unfortunately, as **Java** only allows functions to have variadic parameter only in the last position, it is sometimes impossible to 

The following code works just fine:
```Java
// SomeClass.java

public class SomeClass {
    public static void main(String[] args) {
        MainKt.bar(1, "hello", "world"); // Compiles and runs
    }
}
```

```kotlin
// Main.kt

fun bar(x: Int, vararg args: String) {
    args.forEach {
        println(it)
    }
}
```

However, if the variadic parameter in the **Kotlin** function signature is not on the last position, the **Java** code will not compile with single variadic arguments:

```Java
// SomeClass.java

public class SomeClass {
    public static void main(String[] args) {
        MainKt.bar("hello", "world", 1); // Compilation error 
      
        String[] arr = new String[] {"hello", "world"};
        MainKt.bar(arr, 1); // Compiles and runs
    }
}
```

```kotlin
// Main.kt

fun bar(vararg args: String, x: Int) {
    args.forEach {
        println(it)
    }
}
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

* As for the overriding, the most obvious and simple workaround is to create an overriding method that accepts boxed array data and then calls the real variadic method with this data cast to primitive array.
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


## Current implementation

The whole current implementation can be split into three major steps:

### Variadic function signature transformation
On this step, the original type of `vararg` parameter is replaced with the corresponding array type.
It is done using `FirTypeResolveTransformer` on the `TYPES` resolution phase.

### Function call candidates gathering
This step is done on `BODY_RESOLVE` phase for each function call.
For each potential candidate, several checks are performed.
One of these checks is argument type checking, which is done using `CheckArguments` checker stage.
It compares the type of variadic arguments against the type of the `vararg` parameter (for single elements) or the type of `vararg` array (for spread arguments).

### `VarargLowering` stage
On the backend part, several lowering phases are executed, which desugar higher order concepts.
On `VarargLowering` phase, all the passed variadic arguments are copied into one unified array on the call site.
For this purpose, a special `IrArrayBuilder` is used to add array initialization to the generated IR.

In cases with non-spread arguments only, the compiler just initializes the array of the required size and copies all the passed arguments using `set`.

When there is only one argument, which is spread, the compiler copies the passed array using `Array.copyOf` or `System.arraycopy`.

In cases of multiple arguments with at least one being spread, `IrArrayBuilder` utilizes two additional builders:
* `PrimitiveSpreadBuilder` --- is an interface, used for primitive arrays.
  It has a concrete implementation for every primitive type.
* `SpreadBuilder` --- is a class used for non-primitive arrays.

These helper builders expose several methods: `add`, `addSpread` and `toArray`.
First two methods are used for adding single elements and spread arguments to the array.
The last method is responsible for finalizing the array and returning it.
`IrArrayBuilder` chooses the right helper builder and then generates IR calls to the constructor, `add`/`addSpread` for each argument and `toArray`.

```kotlin
fun bar(vararg x: Int){
}

fun main() {
    val x = intArrayOf(1, 2, 3)
    bar(*x, *x)
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

After that, the `vararg` parameter is finally desugared to a regular array parameter.

## Proposed implementation
We have to modify the last two steps ([Function call candidates gathering](#function-call-candidates-gathering) and [Vararg lowering phase](#vararglowering-stage)) in order to achieve the desired functionality.

### Argument type checking
Currently, when gathering candidates for a function call, all the spread arguments' types are checked against the type of the `vararg` array.
However, this is the main source of all the issues. 
Moreover, it provides a strange and inconvenient type incompatibility message:
```kotlin
fun foo(vararg x: Int) {}

foo(*listOf("Hello"))
// Type mismatch:
// inferred type is List<String> but IntArray was expected
```

The prototype resolves this problem by type checking spread arguments only by the type of their elements.
This was done by introducing several special utilitiy functions.
These functions are able to unwrap any array, `CharSequence` and `Iterable` type.

Type incompatibility errors are now more informative and clear:
```kotlin
fun foo(vararg x: Int) {}

foo(*listOf("Hello"))
// Type mismatch:
// inferred type is String but Int was expected
```

### `VarargLowering` stage

These builders were extended to support all types of spread arguments.
In fact, `SpreadBuilder` already supported spread arguments of all types, except for primitive arrays.

However, there is one more case that needs to be handled.
When there is only one argument, which is spread, the compiler will try to call `copy` method on it.
But if the spread argument is not an array, this will lead to a type cast error.
To avoid this, for all cases with single non-array spread arguments, the compiler will just use the same strategy as for multiple mixed arguments (using additional builders).
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

### Notes on interop with Java
It's important to note that the prototype doesn't break the compatibility with **Java**, as most of the changes only touched the compiler's frontend.

However, the prototype now also allows to pass `Iterable` collections to variadic functions from **Java**:

```Java
// VarargUtil.java

public class VarargUtil {
    public static void printVararg(String... args) {
        for (String arg : args) {
            System.out.println(arg);
        }
    }
}
```

```Kotlin
// Main.kt

import VarargUtil.printVararg

fun main() {
    val mutableSet = mutableSetOf("a", "b", "c")
    printVararg(*mutableSet) // Compiles and runs successfully
}
```

## Prototype steps
✅ Rewrite typechecking pipeline

✅ Rewrite `VarargLowering` stage

✅ Run builtin tests

✅ Benchmark the performance

✅ Add function resolution tests

✅ Rewrite typechecking pipeline used for diagnostics

✅ Add type checking tests

## Alternative approaches

### Artificialy inserting needed cast in FIR

One of the implemented approaches was the idea of artificially adding calls to required cast functions in the FIR.
It consists of the following steps:
1. The typechecking is also done just by the element types
2. After the suitable candidate function is found for each call, all spread arguments on the call site are wrapped into an artificial cast to the required array

Such an approach takes the responsibility of calling casts off the user. 
However, it just hides the underlying problem and doesn't solve it. 

### Using boxed arrays only

There is a `TypeResolveTransformer` that is responsible for transforming types of FIR nodes.
It is also used for transforming the type of `vararg` parameters to the corresponding array type.
For this purpose, it uses a special utility function `createArrayType` that accepts a type and returns an array type with elements of the original type.
This function has an interesting boolean parameter `createPrimitiveArrayTypeIfPossible`, which is set to `true` by default.
If this flag is set to `false`, all the `vararg` parameters will use the boxed array type, regardless of whether the element type is primitive or not.

This approach helps to get rid of any type incompatibility issues. 
However, there are several drawbacks:
1. Boxed arrays introduce significantly more overhead compared to primitive arrays
2. It totally breaks the backward compatibility with the existing code base

### Deprecating `vararg` keyword and moving to collection literals

The main idea is to use a more flexible and convenient approach with collection literals. 
Currently, `vararg` functionality feels legacy, outdated and bulky.
Arrays are not native to **Kotlin,** and it would be great to provide a way to use any other buildable type in the function signature.

The ongoing proposal regarding collection literals (see [KT-43871](https://youtrack.jetbrains.com/issue/KT-43871/Collection-literals)) offers a way to build any collections in-place.
The main idea of collection literals is to extend collection types and add some `build` method to their companion objects:
```kotlin
List [1, 2, 3, someOtherCollection]

// Is converted to

List<Int>.build {
    add(1)
    add(2)
    add(3)
    addAll(someOtherCollection)  
}
```

Using such a feature we could get rid of the `vararg` keyword at all and use collection literals on the call site instead.
This allows using parameters of any collection type (and not just arrays) and "spreading" existing collections to every collection type parameter.

```kotlin
fun foo(x: List<Int>, y: Set<Char>) {}


fun main() {
  val someIntArray = intArrayOf(1, 2, 3)
  val someIntSet = setOf(4, 5, 6)
  val someNumber = 10
  
  val someCharBoxedArray = arrayOf('a', 'b', 'c')
  val someCharSet = setOf('d', 'e', 'f')
  val someChar = 'x'
  
  foo(
    [1, someNumber, 2, *someIntArray, *someIntSet],
    ['a', 'b', 'c', *someCharSet, *someCharBoxedArray, someChar]
  )
}
```

## Potential extensions

### Alternative underlying collection
The variadic functions still feel out of place due to the usage of non-native **Kotlin** arrays. 
It would be more natural to use `Iterable` as the underlying type for variadic functions.
However, it's not that easy, as we have to preserve the compatibility with the existing code base and **Java**.

### Expand spread operator usage to objects

See [Kotlin Discussions](https://discuss.kotlinlang.org/t/spread-object-operator-missing-in-kotlin/7978) Spread-object operator missing in kotlin?

A lot of requests were made to extend the spread operator usage to be able to unwrap not only arrays, but other kinds of objects.
This approach comes directly from [**JavaScript**](#javascript), where spread operator has a lot of use cases when building new objects.

### `dataarg`

See [KT-8214](https://youtrack.jetbrains.com/issue/KT-8214/Allow-kind-of-vararg-but-for-data-class-parameter) "Allow kind of "vararg" but for data class parameter" and related issues 

The concept of `dataarg` is the generalization of the `vararg` keyword to data classes construction.
It's the extension of [the previous idea](#expand-spread-operator-usage-to-objects) to classes constructors.
The main purpose is to be able to smoothly pass duplicating arguments through the sequence of calls.

It should be possible to create a single data class parameter for all the duplicated parameters and pass it between constructors.
However, on the call site it should be possible to pass the options in the named form and using any order.

Such a feature removes the boilerplate code in cases when some class extends another one and has the same constructor parameters.
```kotlin
data class ButtonProps(val prop1: Int, val prop2: String, ...)

class Button(props: ButtonProps) : Component { ... }
class CustomButton(props: ButtonProps, customProp1: Boolean, ...) : Button(props) { ... }

// Instead of
// class Button(prop1: Int, prop2: String, ...)
// class CustomButtor(prop1: Int, prop2: String, customProp1: Boolean, ...) : Button(prop1, prop2, ...)

fun main() { 
  val button = Button((prop1 = 42, prop2 = "helloWorld"))
  val customButten = CustomButton((prop1 = 42, prop2 = "helloWorld"), customProp1 = false)
}
```

### One-or-more varargs
See [KT-57266](https://youtrack.jetbrains.com/issue/KT-57266/One-or-more-varags) "One-or-more varags" and [KT-55890](https://youtrack.jetbrains.com/issue/KT-55890/Mandatory-varargs) "Mandatory varargs"

Several proposals were made to introduce a new kind of variadic parameter that would require at least one argument to be passed.
```kotlin
// One or more elements checked at compile time when called with literals.
// Calling with spread operator would still entail runtime checking, 
// unless maybe a `NonEmptyArray` type was introduced and intrinsified.
fun <T> foo(vararg+ elements: T) { 
    // ...
}

fun main() {
  foo() // Error: at least one element is required
  foo(1) // OK
  foo(1, 2, 3) // OK
}

```

Currently, such behaviour could be mimicked by providing an additional regular parameter right before the `vararg` one:
```kotlin
fun <T> foo(head: T, vararg tail: T) {
  val elements = arrayOf(head) + tail
  // ...
}
```
However, it results in additional array copy.

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

## Related discussions
* [KT-2462](https://youtrack.jetbrains.com/issue/KT-2462) Varargs
* [KT-12663](https://youtrack.jetbrains.com/issue/KT-12663) Spread (*) operator for Iterable (Collection?) in varargs
* [KT-6846](https://youtrack.jetbrains.com/issue/KT-6846) Spread operator doesn't work for boxed arrays (e.g. Array)
* [KT-25350](https://youtrack.jetbrains.com/issue/KT-25350) Scalability Issue when combining collections and vararg function / spread operator
* [KT-9471](https://youtrack.jetbrains.com/issue/KT-9471) Spread operator should be able to mix IntArray and Array
* [KT-9495](https://youtrack.jetbrains.com/issue/KT-9495) vararg and substitution of primitives for type parameters
* [KT-27013](https://youtrack.jetbrains.com/issue/KT-27013) Spread operator doesn't work on primitive type vararg parameters
* [KT-47711](https://youtrack.jetbrains.com/issue/KT-47711) Type mismatch when unpacking IntArray and re-packing to Array
* [KT-19840](https://youtrack.jetbrains.com/issue/KT-19840) Spread operator doesn't work for non-primitive arrays
* [KT-26146](https://youtrack.jetbrains.com/issue/KT-26146) Unable to override generic function with "primitive" vararg type parameter
* [KT-30837](https://youtrack.jetbrains.com/issue/KT-30837) Confusing error message for passing a list/collection to `spread` operator
* [KT-64324](https://youtrack.jetbrains.com/issue/KT-64324) Enum entries should be spreadable
* [KT-27013](https://youtrack.jetbrains.com/issue/KT-27013/Spread-operator-doesnt-work-on-primitive-type-vararg-parameters) Spread operator doesn't work on primitive type vararg parameters
* [KT-17043](https://youtrack.jetbrains.com/issue/KT-17043) Do not create new arrays for pass-through vararg parameters
* [KT-27538](https://youtrack.jetbrains.com/issue/KT-27538) Avoid Arrays.copyOf when inlining a function call with vararg
* [KT-32474](https://youtrack.jetbrains.com/issue/KT-32474) Prevent empty array allocation when vararg is called without arguments
* [KT-9429](https://youtrack.jetbrains.com/issue/KT-9429) Change 'vararg' semantics to always copy
* [KT-22405](https://youtrack.jetbrains.com/issue/KT-22405) Extra array copies created on calling "string".format()
* [Kotlin-Discussions](https://discuss.kotlinlang.org/t/scalability-issue-spread-operator-with-collections/8466) Scalability Issue: Spread operator with collections
* [Kotlin-Discussions](https://discuss.kotlinlang.org/t/allocation-of-vararge-with-spread-operator/2015) Allocation of vararge with spread operator
* [Kotlin-Discussions](https://discuss.kotlinlang.org/t/hidden-allocations-when-using-vararg-and-spread-operator/1640) Hidden allocations when using vararg and spread operator
* [StackOverflow](https://stackoverflow.com/questions/49892840/kotlin-android-spread-an-array-list-for-non-variadic-functions) Kotlin (Android) - Spread an Array/List for Non-Variadic Functions
* [StackOverflow](https://stackoverflow.com/questions/60850350/kotlin-spread-operator-behaviour-on-chars-array) Kotlin spread operator behaviour on chars array
* [r/kotlin](https://www.reddit.com/r/Kotlin/comments/9gkfdv/question_regarding_spread_operator_and_varargs/) Question regarding spread operator and varargs
* [r/kotlin](https://www.reddit.com/r/Kotlin/comments/kslxzk/how_often_do_you_use_the_spread_operator/) How often do you use the spread operator "*" ?