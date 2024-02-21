# Comprehensions syntax in Kotlin
This page is dedicated to researching the possible ways of adding “syntax sugar” for handling call chains and data transformations in a more convenient way.

## Table of Contents

* [Motivation](#motivation)
* [Current Workarounds](#current-workarounds-)
* [Ideas from other Programming Languages](#ideas-from-other-programming-languages-)
* [Library Solutions](#library-solutions)

## Motivation

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

For simplicity, in this document we will try to rewrite the following code using different approaches:

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

## Ideas from other Programming Languages 

### Set-builder notation
The most common appearance of list comprehensions in programming languages, named "Set-builder notation" after the [syntax used for constructing sets in math](https://en.wikipedia.org/wiki/Set-builder_notation).

*Python* is the most known language that chose that way (see the famous [PEP-202](https://peps.python.org/pep-0202/) for more):
```Python
[(a, b) for a in Collection1 for b in Collection2 if Condition]
```

### List monads
*Monads* is a widely used concept in function languages. One of the monad types is *List monads*, which is presented is **Haskell**, **Scala** and **F#**.
```Haskell
pyth :: Int -> [(Int, Int, Int)]
pyth n =
  [ (x, y, z)
  | x <- [1 .. n] 
  , y <- [x .. n] 
  , z <- [y .. n] 
  , x ^ 2 + y ^ 2 == z ^ 2 ]
```


### Query syntax
*Query syntax* is a very rare mechanism, which syntax looks like a database query. 
From all the mainstream languages, *C#* seems like the only one that has this kind of construction (see [LINQ](https://learn.microsoft.com/en-us/dotnet/csharp/linq/)):
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