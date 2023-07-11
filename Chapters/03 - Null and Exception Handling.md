# 3 - Null and Exception Handling

In this chapter, we will investigate how we handle nulls and exceptions in F#. In addition, we'll be extending our understanding of function composition that we looked at in the previous chapter and will start looking at higher-order functions.

> **Higher-Order Functions** take one or more functions as arguments and/or return a function as its output.

## Null Handling

Most of the time, you will not have to deal with null in your F# code as it has a built-in type called Option that you will use instead. It looks very similar to this:

```fsharp
type Option<'T> =
    | Some of 'T
    | None
```

It is a discriminated union with two cases to handle whether there is data or not. The single tick (') before the T is the F# way of showing that T is a Generic type. As a consequence, any type can be treated as optional. 

> You must either delete this Option type definition or comment it out as it already exists in the F# language and it will interfere with the correct compilation of the code if you don't.

Create a new file in your folder called *option.fsx*.

Don't forget to highlight and run the code examples in this chapter in F# Interactive (FSI).

We'll start by creating a function to try to parse a string as a DateTime:

```fsharp
open System

// string -> Option<DateTime> 
let tryParseDateTime (input:string) =
    let (success, value) = DateTime.TryParse input
    if success then Some value else None
```

We need to make an import declaration to the top of the page using the ```open``` keyword to provide access to the .NET DateTime functionality.

If you are wondering how ```DateTime.TryParse``` returns a tuple, the answer is quite simple; the F# language does not support *out* parameters, so the clever folks behind F# made it so that during interop, *out* parameters are added to the function output, which generally results in a tuple. 

You can also pattern match the result of the function with a match expression instead of using an if expression:

```fsharp
// string -> Option<DateTime> 
let tryParseDateTime (input:string) =
    match DateTime.TryParse input with
    | true, result -> Some result
    | false, _ -> None
```

The wildcard symbol ```_``` implies the value can be anything.

```fsharp
// string -> Option<DateTime>
let tryParseDateTime (input:string) =
    match DateTime.TryParse input with
    | true, result -> Some result
    | _, _ -> None
```

The wildcard match ```_, _``` can be simplified to a single wildcard:

```fsharp
// string -> Option<DateTime>
let tryParseDateTime (input:string) =
    match DateTime.TryParse input with
    | true, result -> Some result
    | _ -> None
```

Any of these versions are acceptable but in this case, I would choose either the if expression or the first pattern match as they are the easiest to discern their intent.

Run the following examples in FSI with your chosen version of the function:

```fsharp
let isDate = tryParseDateTime "2019-08-01" // Some 01/08/2019 00:00:00 

let isNotDate = tryParseDateTime "Hello" // None
```

You will see that the string that can be parsed into a valid date returns Some of the valid date and the non-date string returns None.

Another way that the Option type can be used is for optional data like a person's middle name as not everyone has one:

```fsharp
type PersonName = {
    FirstName : string
    MiddleName : Option<string>
    LastName : string
}
```

If the person doesn't have a middle name, you set it to None and if they do you set it to Some "name", as shown here:

```fsharp
let person = { FirstName = "Ian"; MiddleName = None; LastName = "Russell"}

let person2 = { person with MiddleName = Some "????" }
```

Notice that we have used the copy-and-update record expression we met in the last chapter.

So far, we have used the ```Option<'T>``` style but there is an ```option``` keyword that we can use instead:

```fsharp
type PersonName = {
    FirstName : string
    MiddleName : string option
    LastName : string
}
```

You will tend to see this style used but neither is wrong.

Sadly, there is one area where nulls can sneak into your codebase and that is through interop with code/libraries written in other .Net languages such as C# including most of the .NET platform.

> **What is.NET?**
>
> .NET is a free, cross-platform, open-source developer platform for building many different types of applications.

## Interop With .NET

If you are interacting with code written in C#, there is a chance that you will have some null issues. In addition to the Option type, F# also offers the *Option* module that contains some very useful helper functions to make life easier.

Let's create a null for both a Reference type and a Nullable primitive:

```fsharp
open System

// Reference type
let nullObj:string = null

// Nullable type
let nullPri = Nullable<int>()
```

Run the code in FSI to prove that they are both null.

To convert from .Net to an F# Option type, we can use the `Option.ofObj` and `Option.ofNullable` functions:

```fsharp
let fromNullObj = Option.ofObj nullObj

let fromNullPri = Option.ofNullable nullPri
```

To convert from an Option type to .Net types, we can use the `Option.toObj` and `Option.toNullable` functions.

```fsharp
let toNullObj = Option.toObj fromNullObj

let toNullPri = Option.toNullable fromNullPri
```

Run the code in FSI to show that this works correctly:

What happens if you want to convert from an *Option* type to something that doesn't support null but instead expects a placeholder value? You could use pattern matching as Option is a discriminated union or you can use the `Option.defaultValue` function:

```fsharp
let resultPM input =
    match input with 
    | Some value -> value
    | None -> "------"

let resultDV = Option.defaultValue "------" fromNullObj
```

If the *Option* value is Some, then the value wrapped by Some is returned, otherwise, the default value, in these cases, "------", is returned.

You could also use the forward pipe operator (|>):

```fsharp
let resultFP = fromNullObj |> Option.defaultValue "------" 

let resultFPA = 
    fromNullObj 
    |> Option.defaultValue "------" 
```

If you use this a lot, you may find that using Partial Application might make the task more pleasurable. We create a function that takes the default but not the Option value:

```fsharp
// (string option -> string)
let setUnknownAsDefault = Option.defaultValue "????" 

let result = setUnknownAsDefault fromNullObj
```

Or using the forward pipe operator:

```fsharp
let result = fromNullObj |> setUnknownAsDefault
```

As you can see, handling of null and optional values is handled very nicely in F#. If you are diligent, you should never see a NullReferenceException in a running F# application.

## Handling Exceptions

Create a new file called *result.fsx* in your folder.

We will create a function that does simple division but returns an exception if the divisor is 0:

```fsharp
open System

// decimal -> decimal -> decimal
let tryDivide (x:decimal) (y:decimal) = 
    try
        x/y 
    with
    | :? DivideByZeroException as ex -> raise ex
```

Whilst this code is perfectly valid, the function signature is lying to you; It doesn't always return a decimal. The only way I would know this is by looking at the implementation or getting the error when the code is executed. This goes against the general ethos of F# coding; you should be able to trust function signatures.

Most functional languages implement a type that offers a choice between success and failure and F# is no exception. This is an example of a potential implementation:

```fsharp
type Result<'TSuccess,'TFailure> = 
    | Success of 'TSuccess
    | Failure of 'TFailure
```

Unsurprisingly, there is one built into the language (from F# 4.1) but rather than Success/Failure, it uses Ok/Error. We can delete or comment out our Result type as we will use the one in the F# language.

Let's use the Result type in our `tryDivide` function:

```fsharp
// decimal -> decimal -> Result<decimal,exn>
let tryDivide (x:decimal) (y:decimal) = 
    try
        Ok (x/y) 
    with
    | :? DivideByZeroException as ex -> Error ex
```

The ```try...with``` expression is used to handle exceptions and is analogous to Try-Catch in C#. In our case, we are only expecting one specific error type: ```System.DivideByZeroException```. The cast operator ```:?``` is a pattern matching feature to match a specified type or subtype, in this case, if the error can be cast as ```System.DivideByZeroException```. The ```as ex``` gives us access to the actual exception instance which we then pass as the case data to construct the *Error* case. Any unexpected exception gets passed up the call chain until something else handles it or it crashes the application, just like the rest of .NET.

Run the code in FSI to see what you get from an error and success cases:

```fsharp
// Error "System.DivideByZeroException: Attempted to divide by zero..."
let badDivide = tryDivide 1M 0M 

let goodDivide = tryDivide 1M 1M // Some 1M
```

Next, we are going to look at how we can incorporate the Result type into the composition code we used in the last chapter.

## Function Composition With Result

I have modified the `getPurchases` and `increaseCreditIfVip` functions to return *Result* types but have left the `tryPromoteToVip` function alone so that we see the impact the changes have on function composition:

```fsharp
open System

type Customer = {
    Id : int
    IsVip : bool
    Credit : decimal
}

// Customer -> Result<(Customer * decimal),exn>
let getPurchases customer = 
    try
        // Imagine this function is fetching data from a Database
        let purchases = 
            if customer.Id % 2 = 0 then (customer, 120M) 
            else (customer, 80M)
        Ok purchases
    with
    | ex -> Error ex

// Customer * decimal -> Customer
let tryPromoteToVip purchases = 
    let customer, amount = purchases
    if amount > 100M then { customer with IsVip = true }
    else customer

// Customer -> Result<Customer,exn>
let increaseCreditIfVip customer = 
    try
        // Imagine this function could cause an exception
        let increase =  
            if customer.IsVip then 100.0M else 50.0M
        Ok { customer with Credit = customer.Credit + increase }
    with
    | ex -> Error ex

let upgradeCustomer customer =
    customer 
    |> getPurchases 
    |> tryPromoteToVip // Compiler problem
    |> increaseCreditIfVip

let customerVIP = { Id = 1; IsVip = true; Credit = 0.0M }
let customerSTD = { Id = 2; IsVip = false; Credit = 100.0M }

let assertVIP = 
	upgradeCustomer customerVIP = Ok {Id = 1; IsVip = true; Credit = 100.0M }
let assertSTDtoVIP = 
	upgradeCustomer customerSTD = Ok {Id = 2; IsVip = true; Credit = 200.0M }
let assertSTD = 
	upgradeCustomer { customerSTD with Id = 3; Credit = 50.0M } = Ok {Id = 3; IsVip = false; Credit = 100.0M }
```

There is a problem in the `upgradeCustomer` function on the call to the `tryPromoteToVip` function because the function signatures don't match up any longer. To solve this, we are going to use Higher-Order functions.

> **Higher Order Functions** take one or more functions as arguments and/or return a function as its output.

[Scott Wlaschin](<https://fsharpforfunandprofit.com/rop/>) visualises composition with the Result type as two parallel railway tracks which he calls Railway Oriented Programming (ROP), with one track for Ok and one for Error. You travel on the Ok track until you have an error and then you switch to the Error track. He defines the `tryPromoteToVip` function as a one-track function because it doesn't output a Result type and will only execute on the Ok track. This causes an issue because the output from the previous function and input to the next function doesn't match. 

The first thing that we need to do is to use a match expression in an anonymous function to unwrap the tuple from the Ok part of the Result returned from the `getPurchases` function:

```fsharp
let upgradeCustomer customer =
    customer 
    |> getPurchases 
    |> fun result -> 
        match result with
        | Ok x -> Ok (tryPromoteToVip x)
        | Error ex -> Error ex
    |> increaseCreditIfVip // Compiler problem
```

We have to return a Result from this code, so we wrap the call to `tryPromoteToVip` in an Ok.

We now do the same for the `increaseCustomerCreditIfVip` function. In this case, we don't need to add the Ok as it already returns a result:

```fsharp
let upgradeCustomer customer =
    customer 
    |> getPurchases 
    |> fun result -> 
        match result with
        | Ok x -> Ok (tryPromoteToVip x)
        | Error ex -> Error ex
    |> fun result ->
        match result with
        | Ok x -> increaseCreditIfVip x
        | Error ex -> Error ex
```

It is possible to simplify this code slightly like this:

```fsharp
let upgradeCustomer customer =
    customer 
    |> getPurchases 
    |> function
        | Ok x -> Ok (tryPromoteToVip x)
        | Error ex -> Error ex
    |> function
        | Ok x -> increaseCreditIfVip x
        | Error ex -> Error ex

```

Either style is fine to use.

What we need to do now is to convert our anonymous functions into named functions. We start with the `tryToPromoteToVip` block. Our new function has the following signature:

```plaintext
(Customer * decimal -> Customer) -> Result<Customer * decimal, exn> -> Result<Customer, exn>
```

We are going to call this function `map` for reasons that will become clear later in the section:

```fsharp
// (Customer * decimal -> Customer) -> Result<Customer * decimal, exn> 
// -> Result<Customer, exn>
let map (tryPromoteToVip:Customer * decimal -> Customer) (result:Result<Customer * decimal, exn>) 
	: Result<Customer, exn> =
    match result with
    | Ok x -> Ok (tryPromoteToVip x)
    | Error ex -> Error ex
```

The `map` function is a higher-order function because it takes the `tryPromoteToVip` function as an input parameter.

We don't use the types specific to `tryPromoteToVip` or the result, so we can make this function take generic parameters:

```fsharp
// ('a -> 'b) -> Result<'a,'c> -> Result<'b,'c>
let map (f:'a -> 'b) (result:Result<'a, 'c>) : Result<'b, 'c> =
    match result with
    | Ok x -> Ok (f x)
    | Error ex -> Error ex
```

Although we have left the Error case data identifier as `ex`, it is important to know that now that it is generic, it doesn't have to be an exception and could just as easily be a string or a discriminated union. In fact, it can be any F# type that we choose.

We can remove the types from the function and the compiler will confirm that it is generic:

```fsharp
// ('a -> 'b) -> Result<'a,'c> -> Result<'b,'c>
let map f result = 
    match result with
    | Ok x -> Ok (f x)
    | Error ex -> Error ex
```

We can easily show this is true by trying the new map function with some code that matches the same pattern, such as the `tryParseDateTime` function we met earlier:

```fsharp
// ('a -> 'b) -> Result<'a,'c> -> Result<'b,'c>
let map f result = 
    match result with
    | Ok x -> Ok (f x)
    | Error ex -> Error ex

// string -> DateTime option
let tryParseDateTime (input:string) =
    let success, value = DateTime.TryParse input
    if success then Some value else None

// Result<string, exn>
let getResult = 
    try
        Ok "Hello"
    with
    | ex -> Error ex

// Result<DateTime option, exn>
let parsedDT = getResult |> map tryParseDateTime
```

If we use the `map` function in our `upgradeCustomer` function we get this:

```fsharp
let upgradeCustomer customer =
    customer
    |> getPurchases
    |> map tryPromoteToVip
    |> fun result ->
        match result with
        | Ok x -> increaseCreditIfVip x
        | Error ex -> Error ex
```

Now we do a similar thing for the `increaseCreditIfVip` function. This is slightly different to the map function as we don't need to wrap the output in a result as the `increaseCreditIfVip` already does that. The function will be called `bind` for reasons we will see very soon:

```fsharp
// (Customer -> Result<Customer, exn>) -> Result<Customer, exn> -> Result<Customer, exn>
let bind (increaseCreditIfVip:Customer -> Result<Customer, exn>) (result:Result<Customer, exn>) : Result<Customer, exn> =
	match result with
    | Ok x -> increaseCreditIfVip x
    | Error ex -> Error ex
```

As with the `map` function, `bind` is also a higher order function because it takes the `increaseCreditIfVip` function as an input parameter.

We can make `bind` generic as we don't use any of the parameters in the function:

```fsharp
// ('a -> Result<'b, 'c>) -> Result<'a, 'c> -> Result<'b, 'c>
let bind (f:'a -> Result<'b, 'c>) (result:Result<'a, 'c>) : Result<'b, 'c> =
	match result with
    | Ok x -> f x
    | Error ex -> Error ex
```

And we can remove the parameter types:

```fsharp
// ('a -> Result<'b, 'c>) -> Result<'a, 'c> -> Result<'b, 'c>
let bind f result = 
	match result with
    | Ok x -> f x
    | Error ex -> Error ex
```

Let's plug the `bind` function into `upgradeCustomer`:

```fsharp
let upgradeCustomer customer =
    customer 
    |> getPurchases 
    |> map tryPromoteToVip 
    |> bind increaseCreditIfVip
```

The code should now have no compiler warnings. Run the code in FSI and then the asserts to verify the code works as expected.

Written as procedural code, the `upgradeCustomer` function looks like this:

```fsharp
let upgradeCustomer customer =
    let purchasedResult = getPurchases customer
    let promotedResult = map tryPromoteToVip purchasedResult
    let increaseResult = bind increaseCreditIfVip promotedResult
    increaseResult
```

The reason for using `map` and `bind` as function names is because the *Result* module in F# has them built in as it is a common requirement. Let's update the `upgradeCustomer` function to use the *Result* module functions rather than our own.

```fsharp
let upgradeCustomer customer =
    customer 
    |> getPurchases 
    |> Result.map tryPromoteToVip 
    |> Result.bind increaseCreditIfVip
```

We can delete our `map` and `bind` functions as we don't need them any longer.

If you want to learn more about this style of programming, I highly recommend Scott's book [Domain Modelling Made Functional](<https://pragprog.com/book/swdddf/domain-modeling-made-functional>). Not only does it cover Railway Oriented Programming (ROP) but also lots of very useful Domain-Driven Design information. It's my favourite technical book!

## Summary

We've completed another chapter and have covered a lot of additional features and concepts that build upon our existing knowledge.

- Null handling
- Option type and module
- Exception handling
- Result type and module
- Higher Order Functions

In the next chapter, we'll start looking at organising your code into projects and unit testing.