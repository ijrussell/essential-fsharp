# 2 - Functions

In this chapter, we are going to concentrate on functions. Functions are the workhorses of the Functional programming worl
We will introduce the practice of composing pipelines of functions that can perform more complicated tasks. We will also see why understanding function signatures is very important in F# programming.

## Getting Started

Create a new folder in the solution to store the code from this chapter. Create a new F# script file in the new folder.

## Functions 101

Functions in F# are fundamentally very simple:

> **Functions have one rule: They take one input and return one output.**

In the previous chapter, we wrote a function that had two input parameters. Later in this Chapter, we will discover why that fact and the one input parameter rule for functions are not actually conflicting.

In this chapter, we are going to concentrate on a special subset of functions: `Pure Functions`.

> A **Pure Function** has the following rules:
>
> - They are deterministic. Given the same input, they will always return the same output.
> - They generate no side effects.
>
> **Side Effects** are activities such as talking to a database, sending email, handling user input, and random number generation or using the current date and time. It is highly unlikely that you will ever write a program that is free of side effects.

Functions that satisfy these rules have many benefits: They are easy to test, cacheable, and parallelizable.

You can do a lot in a single function but you can have better code reusability by combining smaller functions together: We call this `Function Composition`.

## Function Composition

The composition of two functions relies on the output of the first one matching the input of the next.

If we have two functions (f1 and f2) that look like this pseudocode:

```fsharp
// The ' implies a generic (any) type
f1 : 'a -> 'b
f2 : 'b -> 'c
```

As the output of f1 matches the input of f2, we can combine them together to create a new function (f3):

```fsharp
f3 : f1 >> f2 // 'a -> 'c
```

Treat the composition operator `>>` as a general-purpose operator for composing any two functions.

What happens if the output of f1 does not match the input of f2?:

```fsharp
f1 : 'a -> 'b
f2 : 'c -> 'd
where 'b is not the same type as 'c
```

To resolve this, we would create an adaptor function, or use an existing one, that we can plug in between f1 and f2:

```fsharp
f3 : 'b -> 'c
```

After plugging f3 in, we get a new function f4:

```fsharp
f4 : f1 >> f3 >> f2 // 'a -> 'd
```

Any number of functions can be composed together in this way.  

Let's look at a concrete example.

## In Practice

I've taken and slightly simplified some of the code from Jorge Fioranelli's excellent [F# Workshop](http://www.fsharpworkshop.com/). Once you've finished this Chapter, I suggest that you download the workshop (it's free!) and complete it.

This example has a simple `Record` type and three functions that we can compose together because the function signatures match up.

```fsharp
type Customer = {
    Id : int
    IsVip : bool
    Credit : decimal
}

// Customer -> (Customer * decimal)
let getPurchases customer = 
    let purchases = if customer.Id % 2 = 0 then 120M else 80M
    (customer, purchases) // Parentheses are optional

// (Customer * decimal) -> Customer 
let tryPromoteToVip purchases = 
    let (customer, amount) = purchases
    if amount > 100M then { customer with IsVip = true }
    else customer

// Customer -> Customer
let increaseCreditIfVip customer = 
    let increase = if customer.IsVip then 100M else 50M
    { customer with Credit = customer.Credit + increase }
```

There are a few things in this sample code that we haven't seen before. The `getPurchases` function returns a `Tuple`. `Tuples` can be used for transferring small chunks of data around, generally as the output from a function. Notice the difference between the definition of the tuple `(Customer * decimal)` and the usage `(customer, amount)` when we decompose it into its constituent parts.

The other new feature is the copy-and-update record expression. This allows you to create a new record instance based on an existing record, usually with some modified data. Records and their properties are immutable by default.

F# supports many ways to compose functions together, including non-FP styles:

```fsharp
// Nested
let upgradeCustomerNested customer = 
    increaseCreditIfVip(tryPromoteToVip(getPurchases customer))

// Procedural
let upgradeCustomerProcedural customer = 
    let customerWithPurchases = getPurchases customer
    let promotedCustomer = tryPromoteToVip customerWithPurchases
    let increasedCreditCustomer = increaseCreditIfVip promotedCustomer
    increasedCreditCustomer
```

Composition Operator

```fsharp
let upgradeCustomerComposed = 
    getPurchases >> tryPromoteToVip >> increaseCreditIfVip
```

Forward Pipe Operator

```fsharp
let upgradeCustomerPiped customer = 
    customer 
    |> getPurchases 
    |> tryPromoteToVip 
    |> increaseCreditIfVip
```

They all have the same `Function Signature` and will produce the same result with the same input.

The *upgradeCustomerPiped* function uses the forward pipe operator (|>). It is equivalent to the *upgradeCustomerProcedural* function but without having to specify the intermediate values. The value from the line above gets passed down as the last input argument of the next function. The difference between the function composition operator `>>` and the forward pipe operator `|>` is that the function composition operator sits between two functions, whereas the forward pipe sits between a value and a function.

> Use the forward pipe operator (`|>`) as your default style.

It is quite easy to verify the output of the upgrade functions using FSI.

```fsharp
let customerVIP = { Id = 1; IsVip = true; Credit = 0.0M }
let customerSTD = { Id = 2; IsVip = false; Credit = 100.0M }

let assertVIP = 
    upgradeCustomerComposed customerVIP = {Id = 1; IsVip = true; Credit = 100.0M }
let assertSTDtoVIP = 
    upgradeCustomerComposed customerSTD = {Id = 2; IsVip = true; Credit = 200.0M }
let assertSTD = 
    upgradeCustomerComposed { customerSTD with Id = 3; Credit = 50.0M } = {Id = 3; IsVip = false; Credit = 100.0M }
```

`Records` use `Structural Equality`; If two `Records` contain the same data, they are equal using the `Equality Operator` (`=`).

Try replacing the *upgradeCustomerComposed* function with any of the other three functions in the asserts to confirm that they produce the same results in FSI.

## Unit

All functions must have one input and one output. To solve the problem of a function that doesn't require an input value or produce any output, F# has a special type called `Unit`.

> Any function taking `Unit` as the only input or returning it as the output is causing side effects or does nothing.

```fsharp
open System

// unit -> System.DateTime
let now () = DateTime.UtcNow 

// 'a -> unit
let log msg = 
    // Log message or similar task that doesn't return a value
    ()
```

Unit appears in the function signature as `unit` but in code, you will use `()`.

If you call the *now* function without the unit () parameter you will get a result similar to this:

```fsharp
val it : (unit -> DateTime) = <fun:it@30-4>
```

This looks really strange but what you actually get is a function as the output that takes unit and returns a `DateTime` value as shown by the signature. To execute the function, you need to supply the input parameter. This is an important F# feature called `Partial Application`, which we will explain later in this chapter.

If you forget to add the unit parameter `()` when you define the binding, you get a fixed value from the first time the line is executed:

```fsharp
let fixedNow = DateTime.UtcNow

let theTimeIs = fixedNow
```

Wait a few seconds and run the `theTimeIs` binding again and you'll notice that the date and time haven't changed from when the binding was created.

> A function binding always has at least one parameter, even if it is just unit, otherwise, it is a value binding. You can tell if it's a function by checking whether the signature has an arrow (`->`) in it.

## Anonymous Functions

So far, we have only dealt with named functions but we can also create functions without names, commonly called `Anonymous Functions`. We'll start with a simple named function:

```fsharp
// int -> int -> int
let add x y = x + y

let sum = add 1 4
```

We can rewrite the `add` function to use a lambda (`->`):

```fsharp
// int -> int -> int
let add = fun x y -> x + y

let sum = add 1 4
```

Notice that the signatures remain the same (`int -> int -> int`). If you're wondering why the compiler has inferred that the inputs and output are integers, it is because the default for `+` usage is with integers.

Functions are first-class citizens in F# which means that we can use them like other values as input parameters for functions. In the following function, `f` is a function:

```fsharp
// (int -> int -> int) -> int -> int -> int
let apply (f:int -> int -> int) (x:int) (y:int) : int = f x y
```

The compiler has decided that this is a generic function that can take any function with two input parameters. Let's call the `apply` function with the `add` function and two input values:

```fsharp
let sum = apply add 1 4
```

We can also pass an anonymous function with the same signature as `add` into the function:

```fsharp
let sum = apply (fun x y -> x + y) 1 4
```

Anonymous functions are used throughout F# because they save us having to write lots of small functions to do simple, one-off tasks.

We will look much more closely at this in Chapter 3 when we will be introduced to Higher-Order functions.

## Type Inference

```fsharp
// ('a -> 'b -> 'c) -> 'a -> 'b -> 'c
let apply f x y = f x y
```

## Scope

#########################

```fsharp
open System

// unit -> int
let rnd () =
    let rand = Random()
    rand.Next(100)

List.init 50 (fun _ -> rnd())
```

The underscore (`_`) tells the compiler that we can ignore that value because we are not going to use it.

This will create a new instance of the Random class every time that it is called when the list is created.

The version below uses scoping rules to re-use the same instance of the Random class each time you call the *rnd* function:

```fsharp
open System

// unit -> int
let rnd =
    let rand = Random()
    fun () -> rand.Next(100)

List.init 50 (fun _ -> rnd())
```

## Multiple Parameters

At the start of the chapter, I stated that it is a rule that all functions must have one input and one output but in the last chapter, we created a function with multiple input parameters:

```fsharp
// Customer -> decimal -> decimal
let calculateTotal customer spend = ... 
```

The *calculateTotal* function takes a Customer as input and returns a function with the signature (decimal -> decimal) as output; That function takes a decimal as input and returns a decimal as output. The ability to automatically chain single input/single output functions together like this is called `Currying` after Haskell Curry, a US Mathematician. It allows you to write functions that appear to have multiple input arguments but are actually a chain of one input/one output functions. This seemingly odd behaviour that F# performs under the hood for us, opens the way to another very powerful functional concept called `Partial Application`.

#########

```fsharp
// int -> int -> int
let add x y = x + y

// int -> int -> int
let add = fun x y -> x + y
```


```fsharp
// int -> int -> int
let add x = fun y -> x + y
```

```fsharp
// int -> int
let sum = add 5
```


## Partial Application (Part 1)

We will use the *calculateTotal* function to illustrate how partial application works. We can ignore the actual implementation of the function as we only care about the input parameters:

```fsharp
// Customer -> decimal -> decimal
let calculateTotal customer spend = ...
```

If I only provide the first parameter, I get a new function as output with signature (Decimal -> Decimal):

```fsharp
// Decimal -> Decimal
let partial = calculateTotal john
```

If we then supply the final parameter, the original function completes and returns the expected output:

```fsharp
// Decimal
let complete = partial 100.0M
```

Run the code in FSI and look at the outputs from both lines of code.

You must add the input parameters in strict left to right order, in ones or multiples. Trying to add a parameter out of order will not compile if the types don't match:

```fsharp
let doesNotWork = calculateTotal 100M // Does not compile
```

You may wonder why you would want to do this but the partial application of function parameters has some very important uses such as allowing the forward pipe operator `|>` we used earlier in this chapter.

## The Forward Pipe Operator

As the forward pipe operator is something that we will be using a lot, it's worth spending some time investigating what it is and how it works under the covers. Hint: It relies on partial application.

We saw that we could take the `calculateTotal` function:

```fsharp
// Customer -> decimal -> decimal
let calculateTotal customer spend = ...
```

By only applying the first input parameter, we can create a partially applied function which we can complete by applying the missing input parameter:

```fsharp
// Decimal -> Decimal
let partial = calculateTotal john

// Decimal
let complete = partial 100.0M
```

We can do the same thing without the need for the named function by using the forward pipe operator:

```fsharp
let complete = 100.0M |> calculateTotal john
```

The value to the left of the forward pipe operator is applied as the missing parameter to the partially applied function on the right of the operator.

The simple asserts that we wrote in the last chapter, can be made more readable than this example:

```fsharp
let assertJohn = calculateTotal john 100.0M = 90.0M
```

We added a simple helper function and used that:

```fsharp
// 'a -> 'a -> bool (' means generic in F#)
let areEqual expected actual =
    expected = actual

let assertJohn = areEqual 90.0M (calculateTotal john 100.0M)
```

This works nicely but it would be nice if the assertion read as `calculateTotal john 100.0M` is equal to `90.0M`. We can't quite do this but using the forward pipe operator we can get quite close:

```fsharp
// 'a -> 'a -> bool
let isEqualTo expected actual =
    expected = actual
    
let assertJohn = calculateTotal john 100.0M |> isEqualTo 90.0M
```

You don't have to change the name of the helper function but we did in this case as it makes it read more like an English sentence.

In this example, `isEqualTo` is partially applied as part of the forward pipe because only one of the two input parameters that it needs is provided. The other parameter is passed through by the forward pipe operator. In this case, we are passing the result of a function to the right-hand side.

So what is the forward pipe operator (`|>`)? It is defined as part of FSharp.Core but it looks something like this:

```fsharp
// 'a -> ('a -> 'b) -> 'b
let (|>) v f = f v
```

Despite this being very short, there is a lot to unpack here.

This looks like a function and it is but it's a special category called a custom operator. The name of the function is `|>` but the parentheses surrounding it are required because it can be used in two different ways which we will see shortly.

The operator takes a value of type *'a* as the first parameter and a function that takes an *'a* and returns a *'b* as the second parameter. In the scope of the forward pipe, the function is considered to be partially applied because it has not had all of its required parameters applied until the one is passed from the left of the forward pipe.

This operator is a higher-order function because it takes a function as a parameter.

> **Higher-Order Functions** take one or more functions as parameters and/or return a function as its output.

```fsharp
let sum = 5 + 7 // Infix
```

```fsharp
let sum = (+) 5 7 // Prefix
```

```fsharp
let double x = x * 2

let calculate = double 5 

let calculate = (|>) 5 double // Prefix

let calculate = 5 |> double // Infix
```


When the value on the left of the forward pipe operator is applied to the partially applied function on the right, it has all of the parameters it needs and can execute to return a value of type *'b*.

Let's use the operator in our assert in the defined prefix form:

```fsharp
// decimal -> (decimal -> bool) -> bool
let assertJohn = (|>) (calculateTotal john 100.0M) (isEqualTo 90.0M)
```

Normally we would skip this step as it looks worse that our original version but we can convert our operator from the current prefix form into the infix form like this:

```fsharp
// decimal -> (decimal -> bool) -> bool
let assertJohn = (calculateTotal john 100.0M) |> (isEqualTo 90.0M)
```

Notice that as an infix operator, you don't use the parentheses.

The last part is to remove the unnecessary parentheses:

```fsharp
// decimal -> (decimal -> bool) -> bool
let assertJohn = calculateTotal john 100.0M |> isEqualTo 90.0M
```

The `isEqualTo` function has a signature of `decimal -> decimal -> bool`, so if we apply the first input parameter, it now has the signature of `decimal -> bool`. When you apply the decimal result from the `calculateTotal` function, the `isEqualTo` function has all of its input parameters and executes producing the `bool` result.

Understanding how the forward pipe operator works is important as we will be using it throughout the book. In reality, to use it, you only need to remember that the value on the left of the operator is applied as the last parameter to the function on the right of the operator.

## Partial Application (Part 2)

Now that we have a better understanding of partial application, let's have a look at another example. Create a function that takes the log level as a discriminated union and a string for the message:

```fsharp
type LogLevel = 
    | Error
    | Warning
    | Info

// LogLevel -> string -> unit
let log (level:LogLevel) message = 
    printfn $"[{level}]: {message}"
    () // return unit
```

Every function in F# must return an output. In this case, we added an extra line to output *unit*. However, *printfn* returns *unit*, so we can remove the additional row:

```fsharp
// LogLevel -> string -> unit
let log (level:LogLevel) message = 
    printfn $"[{level}]: {message}"
```

To partially apply the *log* function, I'm going to define a new function that only takes the *log* function and its level argument but not the message:

```fsharp
let logError = log Error // string -> unit
```

The name *logError* is bound to a function that takes a *string* and returns *unit*. So now, we can use the *logError* function instead:

```fsharp
let m1 = log Error "Curried function" 

let m2 = logError "Partially Applied function"
```

As the return type is *unit*, you don't have to let bind the result of the function to a value:

```fsharp
log Error "Curried function" 

logError "Partially Applied function"
```

Partial application is a very powerful concept that is only made possible because of curried input parameters. It is not possible with a single, tupled parameter as you need to supply the whole parameter at once:

```fsharp
type LogLevel = 
    | Error
    | Warning
    | Info

// (LogLevel * string) -> unit
let log (level:LogLevel, message:string) = 
    printfn $"[{level}]: {message}"
```

There's so much more that we could cover here but we've already covered a lot in this chapter and we've got the rest of the book to fit them in!

## Summary

In this chapter, we have covered:

- Pure functions
- Anonymous functions
- Function composition
- Tuples
- Copy-and-update record expression
- Curried and tupled parameters
- Currying and partial application

We have now covered the fundamental building blocks of programming in F#: Composition of types and functions, expressions, and immutability.

In the next chapter, we will investigate the handling of null and exceptions in F#.