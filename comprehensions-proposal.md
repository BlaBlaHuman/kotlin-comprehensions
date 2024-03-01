# Comprehensions syntax in Kotlin
This page is dedicated to researching the possible ways of adding “syntax sugar” for handling call chains and data transformations in a more convenient way.

## Table of Contents

* [Motivation](#motivation)
* [Current Workarounds](#current-workarounds-)
* [Ideas from other Programming Languages](#ideas-from-other-programming-languages-)
  * [Set-builder notation](#set-builder-notation)
  * [List monads](#list-monads)
  * [Query syntax](#query-syntax)
  * [Pipe-forwarding](#pipe-forwarding)
* [Library Solutions](#library-solutions)
* [List of Discussions](#list-of-discussions)
* [Additional Resources](#additional-resoures)

## Motivation

After researching a lot of discussions/libraries/tickets on comprehension, chaining, and data transformation
it was noticed that almost all of them were regarding the readability problems. 
That's why we decided to focus on this particular issue.
Kotlin offers a range of expressive and powerful features that greatly enhance working with lambdas and collections. 
However, during the process of product development, there are cases when the usage of these constructions can result in code that is difficult to read and unnecessarily complex due to deep nesting (example taken from [KT-18861](https://youtrack.jetbrains.com/issue/KT-18861)):
```Kotlin
fun login(oldAuth: AuthToken): Collection<UserModel> {
  credentialService.oauthLogin(oldAuth).flatMap { token ->
    securityService.getRole(token).flatMap { role ->
      credentialService.getUserDetails(token, role).map { userDetails ->
        (token, role, userDetails)
      }
    }
  }
    .filter { (token, role, userDetails) -> areValidModelParams(token, role, userDetails) }
    .flatMap { (token, role, userDetails) ->
      token.flatMap { t: AuthToken ->
        role.flatMap { r: Role ->
          userDetails.map { u: UserDetails ->
            (t, r, u)
          }
        }
      }
        .map { (token, role, userDetails) ->
          promotionsService.getPromotions(token, userDetails).flatMap { promotions ->
            adService.getAds(token, userDetails).map { ads ->
              UserModel(token, userDetails, role, promotions, ads)
            }
          }
        }
    }
}
```

Many Kotlin frameworks incorporate the concept of context-driven programming, in which contexts serve as containers for application parameters. 
However, in some cases, multiple nested contexts are required just for a few lines of code, leading to a convoluted nesting structure that is difficult to comprehend.

```kotlin
eventLoop(...) { 
    withSettings(...) { 
        withEnviroment(...) { 
            withAnotherScope(...) {
              doSomething()
            }   
        }
    }
}
```

And here are some real-life examples taken from [Task Tracker JetBrains project](https://github.com/JetBrains-Research/tasktracker-3/tree/b6e78c867123332a498828c61720e034b532acbc):
```kotlin
private fun removeVirtualFileListener(virtualFiles: List<VirtualFile>) {
        virtualFiles.forEach { file ->
            ApplicationManager.getApplication().invokeAndWait {
                val document = FileDocumentManager.getInstance().getDocument(file)
                document?.let {
                    DocumentLogger.removeDocumentLogPrinter(document)
                    document.removeDocumentListener(listener)
                }
            }
        }
    }

private fun getContentEntry(url: VirtualFile?, rootModel: ModifiableRootModel): ContentEntry? {
        rootModel.contentEntries.forEach { e ->
            url?.let {
                if (VfsUtilCore.isEqualOrAncestor(e.url, url.url)) {
                    return e
                }
            }
        }
        return null
    }
```

For simplicity, in this document we will try to rewrite the following code using different approaches.
This code exploits monads, but this fact doesn't matter anyhow. We could operate on any collection or data type in a simillar way.

```Kotlin
val io = printIntroductionText()
        .flatMap { retrieveWorldSizeKm() }
        .map { worldSizeKm -> convertKmToMiles(worldSizeKm) }
        .flatMap { worldSizeMiles ->
          readInitialPosition(worldSizeMiles)
            .flatMap { pos ->
              readInitialDirection()
                .flatMap { dir ->
                    initState(worldSizeMiles, pos, dir)
                }
            }
        }
```
## Current Workarounds 
There are some workarounds presented on the internet:
* In many cases with nested computations there are only at most two collections involved. 
Moreover, usually the computations happen on the Cartesion product of these collections.
That's why the following structure sometimes can be handy (taken from [this StackOverflow answer](https://stackoverflow.com/a/48872563)):
  ```kotlin
  fun <F,S> Collection<F>.cartesian(other: Collection<S>): Sequence<Pair<F,S>> =
      this.asSequence().map { f -> other.asSequence().map { s-> f to s } }.flatten()
  ```
  And some of the usages:
  ```kotlin
  
  setA.cartesian(setB).filter { (a,b) ->
      a.toString().slice(2..3) == b.toString().slice(0..1)
  }.forEach{ (a,b) -> test.add(setOf(a,b)) }
  
  
  setA.cartesian(setB).filter { (a,b) ->
      a.toString().slice(2..3) == b.toString().slice(0..1)
  }.forEach{ test2.add(it) }
  
  test.addAll(setA.cartesian(setB).filter { (a,b) ->
                    a.toString().slice(2..3) == b.toString().slice(0..1)
              }.map { (a,b) -> setOf(a,b) } )
  
  test2.addAll(setA.cartesian(setB).filter { (a,b) ->
          a.toString().slice(2..3) == b.toString().slice(0..1)
      })
  ```

* Take a look at this [CodeWars kata](https://www.codewars.com/kata/5a6f71185084d76d2000001b/kotlin) that offers users to write a simple monad comprehension.
The best current solution for this Kata is this:
  ```kotlin
  package solution
  
  import java.util.Optional
  
  fun <T> `for`(lambda: suspend SequenceScope<Optional<T>>.() -> Unit): Optional<T> {
      return sequence<Optional<T>> { lambda() }.first()
  }
  
  suspend fun <T> SequenceScope<Optional<T>>.yield(value: T) = yield(Optional.of(value))
  
  suspend fun <T, U> SequenceScope<Optional<T>>.bind(value: Optional<U>): U {
      if (!value.isPresent) {
          yield(Optional.empty())
      }
      return value.get()
  }
  ```
  
## Ideas from other Programming Languages 

### Set-builder notation
The most common appearance of list comprehensions in programming languages, named "Set-builder notation" after the [syntax used for constructing sets in math](https://en.wikipedia.org/wiki/Set-builder_notation).

*Python* is the most known language that chose that way (see the famous [PEP-202](https://peps.python.org/pep-0202/) for more):
```Python
[(a, b) for a in Collection1 for b in Collection2 if Condition]
["FizzBuzz" if x % 3 == 0 and x % 5 == 0 else "Fizz" if x % 3 == 0 else "Buzz" if x % 5 == 0 else str(x) for x in range(1, 101)]
```
In languages with such list comprehensions syntax roughly looks like this:
```
#bracket #element #iterator #predicate #bracket

or

#bracket #iterator #predicate #element #bracket
```
Such syntax is great for:
* Working with Cartesian product
* Working with complex predicates

With such syntax it could be possible to generate collections in an other way (taken from [KT-14983](https://youtrack.jetbrains.com/issue/KT-14983/Collection-builder-convention)):
```kotlin
val collection = listOf(1, 2, 3)
val ex = HashSet().(for (x in collection) if (x % 2 == 0) x * x else continue)

// instead of

val ex = collection.filter {
        it % 2 == 0
    }.map {
        it * it
    }.toHashSet()

```

Let's imagine how our code could look like with such style:
```kotlin
io = for {
    yield initState(worldSizeMiles, pos, dir)
    for dir in readInitialDirection().flatMap()
    for pos in readInitialPosition(worldSizeMiles).flatMap()
    for worldSizeMiles in retrieveWorldSizeKm().map {worldSizeKm -> convertKmToMiles (worldSizeKm) }
    for _ in printIntroductionText().flatMap()
}
```


### List monads
*Monads* is a widely used concept in function languages. One of the monad types is *List monads*, which is presented is **Haskell**, **Scala** and **F#**. 

Take a look at the example from Haskell:
```Haskell
pyth :: Int -> [(Int, Int, Int)]
pyth n =
  [ (x, y, z)
  | x <- [1 .. n] 
  , y <- [x .. n] 
  , z <- [y .. n] 
  , x ^ 2 + y ^ 2 == z ^ 2 ]
```
The code above might look exactly like [Set-Builder notation](#set-builder-notation), but it is actually just a sugar for the monadic approach:
```Haskell
pythagoreanTriples :: Integer -> [(Integer, Integer, Integer)]
pythagoreanTriples n =
  [1 .. n] >>= (\x ->
  [x+1 .. n] >>= (\y ->
  [y+1 .. n] >>= (\z ->
  if x^2 + y^2 == z^2 then return (x,y,z) else [])))
```
We can also write it using *do-notation*:
```haskell
pythagoreanTriples :: Integer -> [(Integer, Integer, Integer)]
pythagoreanTriples n = do x <- [1 .. n]
                          y <- [x+1 .. n]
                          z <- [y+1 .. n]
                          if x^2 + y^2 == z^2 then return (x,y,z) else []
```
Such syntax is great for:
* Working with Cartesian product
* Working with complex predicates
* Unwrapping data from different monadic types

Let's rewrite our code:
```Kotlin
val io =  {
            worldSizeMiles <- printIntroductionText()
                    .flatMap { retrieveWorldSizeKm() }
                    .map { worldSizeKm -> convertKmToMiles(worldSizeKm) }
            pos <- readInitialPosition(worldSizeMiles)
            dir <- readInitialDirection()
          }
          yield initState(worldSizeMiles, pos, dir)
```


### Query syntax
*Query syntax* is a very rare mechanism, which syntax looks like a database query. 
From all the mainstream languages, *C#* seems like the only one that has this kind of construction (see [LINQ](https://learn.microsoft.com/en-us/dotnet/csharp/linq/)). 
It can be written either in query syntax or in method syntax:
```C#
int[] numbers = [ 5, 10, 8, 3, 6, 12 ];

//Query syntax:
IEnumerable<int> numQuery1 =
    from num in numbers
    where num % 2 == 0
    orderby num
    select num;

//Method syntax:
IEnumerable<int> numQuery2 = 
    numbers
    .Where(num => num % 2 == 0)
    .OrderBy(n => n);
```
[Here](https://learn.microsoft.com/en-us/dotnet/csharp/linq/standard-query-operators/#query-expression-syntax-table) all the supported operations are listed.

Such syntax is great for:
* Working with simple product transformations

Our code could look somehow like this:
```kotlin
val io = printIntroductionText()
    .SelectMany(_ => retrieveWorldSizeKm())
    .Select(worldSizeKm => convertKmToMiles(worldSizeKm))
    .SelectMany(worldSizeMiles => readInitialPosition(worldSizeMiles)
        .SelectMany(pos => readInitialDirection()
            .SelectMany(dir => initState(worldSizeMiles, pos, dir))));

val io = 
    (from _ in printIntroductionText()
    from worldSizeKm in retrieveWorldSizeKm()
    let worldSizeMiles = convertKmToMiles(worldSizeKm)
    from pos in readInitialPosition(worldSizeMiles)
    from dir in readInitialDirection()
    from state in initState(worldSizeMiles, pos, dir)
    select state);
```

### Pipe-forwarding
*Pipe forwarding* (see the [discussion](https://discuss.kotlinlang.org/t/pipe-forward-operator/2098)) is a mechanism used in such languages as **F#**, **Ruby**, **R** and **Elixir**. 
The idea introduces several new operators:
* `|>` (pipe forward) - the operator that passes an argument to a function: 
`(|>) x f = f x`
* `<|` (pipe backward) - the same as `|>`, but inverted: 
`(<|) f x = f x`
* `>>` (forward composition) - the operator that lets you compose the functions together: 
`(>>) f g x = g (f x)`
* `<<` (backward composition) - the same as `>>`, but inverted: 
`(<<) f g x = f (g x)`

Here is a small showcase in *F#*:
```F#
let finalSeq = 
    seq { 0..10 }
    |> Seq.filter (fun c -> (c % 2) = 0)
    |> Seq.map ((*) 2)
    |> Seq.map (sprintf "The value is %i.")
    
let addTimesSubtract = add1 >> times2 >> subtract20
```

Such syntax is great for:
* Composing functions from both left and right sides
* Passing data to functions from both left and right sides
* Languages with partial evaluation

While not being able to rewrite choosen code in this style, I suggest you to look at [this Gist](https://gist.github.com/vjache/5a7977fc6fb113944ae7969f567b7ead#file-pipeforward-kt) 
that shows a small test implementation for pipes-forward.

It seems like the mentioned approach is not so suitable for Kotlin for [these](https://discuss.kotlinlang.org/t/pipe-forward-operator/2098/14) reasons.

## Library Solutions

* [Komprehensions](https://github.com/pakoito/Komprehensions) is one of the most starred solution for this problem on *GitHub*.
It offers several constructions those allow users to write all the computation lambdas on the same level and have access to the results of previous computation blocks:
    ```Kotlin
    /**
     * Composes a sequence from multiple creation functions chained by let.
     *
     * @return chain
     */
    fun <A, B, R> doLet(
            zero: () -> A,
            one: (A) -> B,
            two: (A, B) -> R): R =
            zero.invoke()
                    .let { a ->
                        one.invoke(a)
                                .let { b ->
                                    two.invoke(a, b)
                                }
                    }
    
    fun calculateDoubles(calcParams: Params) =
        doLet(
            { calcParams },
            { params -> defrombulate(params.first, param.second) },
            { params, result -> gaussianRoots(result) },
            { params, result, grtts -> storeResult(params.first, params.second, result, grtts) }
        )
    ```
    Other constructions provide the simillar syntax but for collections:
    ```Kotlin
    /**
     * Composes an [Iterable] from multiple functions chained by [Iterable.map]
     *
     * @return iterable
     */
    fun <A, B, C, R> doMapIterable(
            zero: () -> Iterable<A>,
            one: (A) -> B,
            two: (B) -> C,
            three: (C) -> R): Iterable<R> =
            zero()
                    .map(one)
                    .map(two)
                    .map(three)
    
    /**
     * Composes an [Iterable] from multiple creation functions chained by flatMap.
     *
     * @return iterable
     */
    fun <A, B, R> doFlatMapIterable(
            zero: () -> Iterable<A>,
            one: (A) -> Iterable<B>,
            two: (A, B) -> Iterable<R>): Iterable<R> =
            zero.invoke()
                    .flatMap { a ->
                        one.invoke(a)
                                .flatMap { b ->
                                    two.invoke(a, b)
                                }
                    }
    ```
  This library also has a version for **RxJava** and **Reactor** objects. Here are code examples those were found on GitHub:
  ```kotlin
    fun updateProduct(updatedProduct: Product): Mono<Product> {
      return doFlatMapMono(
          { getProductById(updatedProduct.id) },
          { currentProduct -> updateColors(currentProduct.colors, updatedProduct.colors, currentProduct.id).collectList() },
          { currentProduct, _ -> updateSizes(currentProduct.sizes, updatedProduct.sizes, currentProduct.id).collectList() },
          { _, _, _ -> productRepository.update(updatedProduct) },
          { _, newColors, newSizes, _ -> updatedProduct.copy(colors = newColors, sizes = newSizes).toMono() }
      )
  }
  
  fun getProductById(productId: Int): Mono<Product> {
      return doFlatMapMono(
          { productRepository.findById(productId) },
          { _ -> colorRepository.findByProduct(productId).collectList() },
          { _, _ -> sizeRepository.findByProduct(productId).collectList() },
          { product, colors, sizes -> product.copy(colors = colors, sizes = sizes).toMono() }
      )
  }
  ```
  ```kotlin
  doSwitchMap(
              { sites },
              { Observable.interval(0, 60, TimeUnit.SECONDS) }
          ) { list, interval ->
              Observable.just(Pair(list, interval))
          }
              .doOnSubscribe { setState { copy(isLoading = true) } }
              .execute {
                  copy(
                      listOfItems = it()?.first ?: emptyList(),
                      interval = it()?.second ?: 0,
                      isLoading = false
                  )
              }
  ```
  Let's omit the fact that it works only on iterables and try to imagine our code with such approach:
  ```kotlin
    val result = 
            doFlatMap(
                {
                    printIntroductionText()
                    .flatMap { retrieveWorldSizeKm() }
                    .map { worldSizeKm -> convertKmToMiles(worldSizeKm) }
                },
                { worldSizeMiles -> readInitialPosition(worldSizeMiles)},
                { _, _ -> readInitialDirection() },
                {worldSizeMiles, pos, dir -> initState(worldSizeMiles, pos, dir) }
            )
  ```

* [KIO](https://github.com/colomboe/KIO) offers two ways of list comprehension:
  * The first one is simillar to the one used in `Komprehensions`. It utilizes `mapT` and `flatMapT`, those get the result from the argument function and put it in a tuple with the provided input parameter:
  ```Kotlin
  val io = printIntroductionText()
    .flatMap { retrieveWorldSizeKm() }
    .map { worldSizeKm -> convertKmToMiles(worldSizeKm) }
    .flatMapT { worldSizeMiles -> readInitialPosition(worldSizeMiles) }
    .flatMapT { (_, _) -> readInitialDirection() }
    .flatMap { (size, pos, dir) -> initState(size, pos, dir) }
  ```
  * The second approach is way more interesting. From the documentation:
    * `+` operator is used to sequence effects when the output of the first operand is discarded;
    * `to` infix function, that extracts the output of the same-row function call and introduces a new variable in the comprehension context with that value;
    * `set` infix function, that is equivalent to the to function but must be used if the same-row function call doesn’t return an effect instance (basically the difference is the same of map and flatMap chaining).
  
    See the [blog-post](https://www.msec.it/blog/comprehension-like-syntax-in-kotlin/) from the author.
  Unfortunately, no usages of this library were found.
  Our code would look somehow like that:
    ```Kotlin
    val io = 
      printIntroductionText()                 +
      retrieveWorldSizeKm()                   to  { worldSizeKm ->
      convertKmToMiles(worldSizeKm)           set { worldSizeMiles ->
      readInitialPosition(worldSizeMiles)     to  { pos ->
      readInitialDirection()                  to  { dir ->
      initState(worldSizeMiles, pos, dir)
    }}}}
    ```

* [functional-kotlin](https://github.com/alexandrepiveteau/functional-kotlin) and [funKTionale](https://github.com/MarioAriasC/funKTionale) provide almost identical methods for function composition.
The idea is very simillar to [forward and backward composition](#pipe-forwarding). Let's consider [functional-kotlin](https://github.com/alexandrepiveteau/functional-kotlin):
  * `f..g == f g`
  * `f.andThen(g) == g..f == g f`
  * `f compose g == f..g == f g`
  * `f forwardCompose g == g..f = g f`
  
  Unfortunately, no usages were found.

* [kotlin-monads](https://github.com/h0tk3y/kotlin-monads) offers *do-notation* and *list-monad* based on *Kotlin-coroutines*:
  ```kotlin
  val m = doReturning(MonadListReturn) {
      val x = bind(monadListOf(1, 2, 3))
      val y = bind(monadListOf(x, x + 1))
      monadListOf(y, x * y)
  }
  
  val m = monadListOf(1, 2, 3).bindDo { x ->
      val y = bind(monadListOf(x, x + 1))
      monadListOf(y, x * y)
  }
  
  val m = monadListOf(1, 2, 3).bind { x ->
      monadListOf(x, x + 1).bind { y -> 
          monadList(y, x * y)
      }
  }
  
  assertEquals(monadListOf(1, 1, 2, 2, 2, 4, 3, 6, 3, 9, 4, 12), m)
  ```

## List of Discussions
* [KT-18861 - Is there possibility that kotlin could support for-comprehensions?](https://youtrack.jetbrains.com/issue/KT-18861)
* [KT-14983 Collection builder convention](https://youtrack.jetbrains.com/issue/KT-14983/Collection-builder-convention)
* [Kotlin Discussions - For as an expression](https://discuss.kotlinlang.org/t/for-as-an-expression/1795/7)
* [Kotlin Discussions - Why not multi for?](https://discuss.kotlinlang.org/t/why-not-multi-for/2241)
* [Kotlin Discussions - Iterate over a collection and create a map](https://discuss.kotlinlang.org/t/iterate-over-a-collection-and-create-a-map/808)
* [Kotlin Discusssions - 1 Liners for List Comprehension-like operations?](https://discuss.kotlinlang.org/t/1-liners-for-list-comprehension-like-operations/1900)
* [Kotlin Discussions - Pipe-forward operator |>](https://discuss.kotlinlang.org/t/pipe-forward-operator/2098)
* [StackOverflow - What is the equivalent of Python list, set, and map comprehensions in Kotlin?](https://stackoverflow.com/questions/50323940/what-is-the-equivalent-of-python-list-set-and-map-comprehensions-in-kotlin)
* [StackOverflow - Nested comprehension in Kotlin](https://stackoverflow.com/questions/48848023/nested-comprehension-in-kotlin)
* [StackOverflow - implement a monad comprehension on a list in kotlin using a coroutine](https://stackoverflow.com/questions/67084114/implement-a-monad-comprehension-on-a-list-in-kotlin-using-a-coroutine)
* [StackOverflow - Does Kotlin support monadic comprehension?](https://stackoverflow.com/questions/34248483/does-kotlin-support-monadic-comprehension)

## Additional Resoures:
* [Rosetta Code -List comprehensions](https://rosettacode.org/wiki/List_comprehensions#)
* [Wikipedia - Comparison of programming languages (list comprehension)](https://en.wikipedia.org/wiki/Comparison_of_programming_languages_(list_comprehension))
* [List comprehensions across languages](https://langexplr.blogspot.com/2007/02/list-comprehensions-across-languages_18.html)
* [Rosetta Code - List Monad](https://rosettacode.org/wiki/Monads/List_monad)