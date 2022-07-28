# 12 - Computation Expressions

In this chapter, we are going to look at computation expressions including a first look at asynchronous code in F#. Computation expressions are syntactic sugar for simplifying code when working with effects like Option, Result, and Async. Async is the asynchronous support in F# and is similar to async/await in C#, except that it is lazily evaluated. We now also have *Task* support in F# core, which we will use in the next chapter when we start working with web development.

We will learn how to create our own simple computation expression, how to use it, and use a more complex example where we combine two effects, Async and Result, together. 

We are only going to scratch the surface of what is possible with computation expressions. Once you have finished this chapter, it might be worth going back to Chapter 8 and having another look at the functional validation example which used a custom computation expression.

## Setting Up

Create a new folder in VS Code, open a new Terminal to create a new console app using:

```fsharp
dotnet new console -lang F#
```

## Introduction

Add a new file above *Program.fs* called *OptionDemo.fs*. In the new file, create the following code:

```fsharp
namespace ComputationExpression

module OptionDemo =

    let multiply x y = // int -> int -> int
        x * y

    let divide x y = // int -> int -> int option
        if y = 0 then None
        else Some (x / y)

    // The formula is: f x y = ((x / y) * x) / y
    let calculate x y =
        divide x y
        |> fun v -> multiply v x // compiler error
        |> fun t -> divide t y
```

We have two simple functions, `multiply` and `divide`, one of which returns an *Option*, and we are using them to compose a function called `calculate`. Don't worry about what the functions are doing, we are only interested in the effect, in this case, *Option*.

Because the order of the input parameters is not ideal for forward-piping, we have used anonymous functions to get access to the data passed down. 

This function does not compile due to the divide function returning an *Option*. Let's expand out the function using the match expression to make the function compile:

```fsharp
let calculate x y =
    divide x y
    |> fun result ->
        match result with
        | Some v -> multiply v x |> Some
        | None -> None
    |> fun result ->
        match result with
        | Some t -> divide t y
        | None -> None
```

This function is much larger with all of the pattern matching but it does make it explicitly clear what the function does and shows that there is no early return. 

Think about what happens in the cases where `y = 0` and `y = 1`. In the first case, the `multiply` and the second `divide` function will not be called because the code is on the None track.

The expanded match expressions follow patterns that are well known to us, which means that we can use `Option.map` and `Option.bind` to simplify the function:

```fsharp
let calculate x y =
    divide x y
    |> Option.map (fun v -> multiply v x)
    |> Option.bind (fun t -> divide t y)
```

We use `map` when the function we are applying it to does not generate the effect, in this case, *Option*, when executed and `bind` when the function does apply the effect as `divide` does.

This is much nicer but this is a chapter about computation expressions and how they can simplify coding with effects like *Option*, so let's move on.

Whilst there are some computation expressions built into the core F# language, there isn't one for *Option*, so we are going to create a basic computation expression ourselves. 

Create the following code between the namespace and the module declaration in *OptionDemo.fs*:

```fsharp
[<AutoOpen>]
module Option =

    type OptionBuilder() =
        // Supports let!
        member _.Bind(x, f) = Option.bind f x
        // Supports return
        member _.Return(x) = Some x
        // Supports return!
        member _.ReturnFrom(x) = x

    // Computation Expression for Option 
    // Usage will be option {...}
    let option = OptionBuilder()
```

The ```[<AutoOpen>]``` attribute means that when the namespace is referenced, we automatically get access to the types and functions in the module.

This is a very simple computation expression but is enough for our needs. At this stage, it's not important to fully understand how this works but you can see that we define a class type with some required member functions and then create an instance of that type for use in our code.

Comment out the `calculate` function and create the following that uses our new option computation expression:

```fsharp
// let calculate x y =
//     divide x y
//     |> Option.map (fun v -> multiply v x)
//     |> Option.bind (fun t -> divide t y)

let calculate x y =
    option {
        let! v = divide x y
        let t = multiply v x
        let! r = divide t y
        return r
    }
```

The code does exactly the same as the commented-out version but it hides the *None* track from the user, allowing you to concentrate on the happy path.

The bang (`!`) unwraps the effect using the `Bind` function, in this case, the *Option*, for the happy path. If you look at the *OptionBuilder* code, `return` does the opposite and applies the effect to the data.

If you hover your mouse over the first value, it tells you that it is an `int` value because the bang (`!`) has unwrapped the *Option* for us. Delete the bang and hover over the first value again. You'll see that it is now an `option<int>`, so it hasn't been unwrapped. You will also get an error from the compiler. Put the bang back.

The nice thing about this function is that we don't need to know about `Option.bind` or `Option.map`; It's nearly all hidden away from us, so we can concentrate on providing the functionality.

The bindings with the `let!` get processed by the ```Bind``` function in the computation expression. Let bindings that would have used ```Option.map``` are automatically handled by the underlying computation expression code. The ```return``` matches the ```Return``` function in the computation expression. If we want to use the ```ReturnFrom``` function, we do the following:

```fsharp
let calculate x y =
    option {
        let! v = divide x y
        let t = multiply v x
        return! divide t y
    }
```

Notice the bang (`!`) on the return. If you look at the *OptionBuilder* code, you will notice that this does nothing to the output which is correct because the `divide` function already applies the effect.

To test the code, replace the code in *Program.fs* with the following:

```fsharp
open ComputationExpression.OptionDemo

[<EntryPoint>]
let main argv =
    calculate 8 0 |> printfn "calculate 8 0 = %A" // None
    calculate 8 2 |> printfn "calculate 8 2 = %A" // Some 16
    0
```

Leave the 0 at the end to signify the code has been completed successfully.

Now type `dotnet run` in the Terminal. You should get None and Some 16 as the output in the terminal window.

## The Result Computation Expression

One of the great things about computation expressions is that they are a generic solution to the effect problem.

Create a new file *ResultDemo.fs* above *Program.fs*.

Rather than create our own computation expression for *Result*, we will use an existing one from the *FsToolkit.ErrorHandling* NuGet package.

Use the Terminal to add a NuGet package that contains the result computation expression that we want to use:

```fsharp
dotnet add package FsToolkit.ErrorHandling
```

Add the following code to *ResultDemo.fs*:

```fsharp
namespace ComputationExpression

module ResultDemo =

    open FsToolkit.ErrorHandling

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
                if customer.Id % 2 = 0 then (customer, 120M) else (customer, 80M)
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
            let increase = if customer.IsVip then 100M else 50M
            Ok { customer with Credit = customer.Credit + increase }
        with
        | ex -> Error ex

    // Customer -> Result<Customer,exn>
    let upgradeCustomer customer =
        customer 
        |> getPurchases 
        |> Result.map tryPromoteToVip
        |> Result.bind increaseCreditIfVip
```

Now let's see what turning the `upgradeCustomer` function into a computation expression looks like by replacing it with the following:

```fsharp
let upgradeCustomer customer =
    result {
        let! purchases = getPurchases customer
        let promoted = tryPromoteToVip purchases
        return! increaseCreditIfVip promoted
    }
```

Notice that the bang `!` is only applied to those functions that apply the effect, in this case, Result.

I find this style easier to read but some people do prefer the previous version.

## Introduction to Async

Using the Explorer view, create a folder called *resources* in the code folder for this chapter and then create a new file called *customers.csv*. Copy the following data into the new file:

```text
CustomerId|Email|Eligible|Registered|DateRegistered|Discount
John|john@test.com|1|1|2015-01-23|0.1
Mary|mary@test.com|1|1|2018-12-12|0.1
Richard|richard@nottest.com|0|1|2016-03-23|0.0
Sarah||0|0||
```

Create a new file *AsyncDemo.fs* above *Program.fs* and add the following code:

```fsharp
namespace ComputationExpression

module AsyncDemo =

    open System.IO

    type FileResult = {
        Name: string
        Length: int
    }
    
    let getFileInformation path =
        async {
            let! bytes = File.ReadAllBytesAsync(path) |> Async.AwaitTask
            let fileName = Path.GetFileName(path)
            return { Name = fileName; Length = bytes.Length }
        }
```

The `Async.AwaitTask` function call is required because `File.ReadAllBytesAsync` returns a *Task* because it is a .NET function rather than an F# one. It is very similar in style to Async/Await in C# but Async in F# is lazily evaluated and Task is not.

Let's test our new code. Add the following import declaration to Program.fs:

```fsharp
open System.IO
open ComputationExpression.AsyncDemo
```

Replace the code in the main function with the following code:

```fsharp
Path.Combine(__SOURCE_DIRECTORY__, "resources", "customers.csv")
|> getFileInformation
|> Async.RunSynchronously
|> printfn "%A"
0
```

We have to use `Async.RunSynchronously` to force the *Async* code to run. You should only do this at the entry point of the application.

Run the code by typing the following in the terminal window:

```plaintext
dotnet run
```

## Compound Computation Expressions

So far we have seen how to use a single effect but what happens if we have to use two (or more) like *Async* and *Result*? Thankfully, such a thing is possible and quite commonplace in this style of programming. We are going to use the `asyncResult` computation expression from the *FsToolkit.ErrorHandling* NuGet package we installed earlier.

The original code for this example comes from [FsToolkit.ErrorHandling](<https://demystifyfp.gitbook.io/fstoolkit-errorhandling/asyncresult/ce>).

Create a new file called *AsyncResultDemo.fs* above *Program.fs* and add the following code:

```fsharp
namespace ComputationExpression

module AsyncResultDemo =

    open System
    open FsToolkit.ErrorHandling

    type AuthError =
        | UserBannedOrSuspended

    type TokenError =
        | BadThingHappened of string

    type LoginError = 
        | InvalidUser
        | InvalidPwd
        | Unauthorized of AuthError
        | TokenErr of TokenError

    type AuthToken = AuthToken of Guid

    type UserStatus =
        | Active
        | Suspended
        | Banned

    type User = {
        Name : string
        Password : string
        Status : UserStatus
    } 
```

Add some constants below the type definitions, marked with the [<Literal>] attribute:

```fsharp
[<Literal>]
let ValidPassword = "password"
[<Literal>]
let ValidUser = "isvalid"
[<Literal>]
let SuspendedUser = "issuspended"
[<Literal>]
let BannedUser = "isbanned"
[<Literal>]
let BadLuckUser = "hasbadluck"
[<Literal>]
let AuthErrorMessage = "Earth's core stopped spinning"
```

They could also be written like this:

```fsharp
let [<Literal>] ValidPassword = "password"
let [<Literal>] ValidUser = "isvalid"
let [<Literal>] SuspendedUser = "issuspended"
let [<Literal>] BannedUser = "isbanned"
let [<Literal>] BadLuckUser = "hasbadluck"
let [<Literal>] AuthErrorMessage = "Earth's core stopped spinning"
```

Now we add the core functions, some of which are asynchronous, some return a Result, and one returns both. Don't worry about the how the functions are implemented internally, concentrate only on the inputs and outputs to the functions:

```fsharp
// string -> Async<User option>
let tryGetUser username =
    async {
        let user = { Name = username; Password = ValidPassword; Status = Active }
        return
            match username with
            | ValidUser -> Some user
            | SuspendedUser -> Some { user with Status = Suspended }
            | BannedUser -> Some { user with Status = Banned }
            | BadLuckUser -> Some user
            | _ -> None
    }

// string -> User -> bool
let isPwdValid password user =
    password = user.Password

// User -> Async<Result<unit, AuthError>>
let authorize user =
    async {
        return 
            match user.Status with
            | Active -> Ok ()
            | _ -> UserBannedOrSuspended |> Error 
    }

// User -> Result<AuthToken, TokenError>
let createAuthToken user =
    try
		if user.Name = BadLuckUser then failwith AuthErrorMessage
		else Guid.NewGuid() |> AuthToken |> Ok
    with
    | ex -> ex.Message |> BadThingHappened |> Error
```

The final part is to add the main `login` function that uses the previous functions and the asyncResult computation expression. The function does four things - tries to get the user from the datastore, checks the password is valid, checks the authorization and creates a token to return:

```fsharp
let login username password : Async<Result<AuthToken, LoginError>> =
    asyncResult {
        let! user = username |> tryGetUser |> AsyncResult.requireSome InvalidUser
        do! user |> isPwdValid password |> Result.requireTrue InvalidPwd
        do! user |> authorize |> AsyncResult.mapError Unauthorized
        return! user |> createAuthToken |> Result.mapError TokenErr
    }
```

The return type from the function is *Async<Result<AuthToken, LoginError>>*, so `asyncResult` is *Async* wrapping *Result*. This is very common in Line of Business (LOB) applications written in F#.

Notice the use of functions from the referenced library which make the code nice and succinct. The `do!` supports functions that return *unit* and is telling the computation expression to ignore the returned value unless it is an error. The `Result.mapError` functions are used to convert the specific errors from the `authorize` and `createAuthToken` functions to the *LoginError* type used by the `login` function.

Only those functions that are asynchronous need to use helper functions from the *AsyncResult* module for handling error cases, the others can use those from the *Result* module.

To test the code, copy the following under the existing code in *AsyncResultDemo.fs* or create a new file called *AsyncResultDemoTests.fs* between *AsyncResultDemo.fs* and *Program.fs*.

```fsharp
module AsyncResultDemoTests =

    open AsyncResultDemo

    let [<Literal>] BadPassword = "notpassword"
    let [<Literal>] NotValidUser = "notvalid"

    let isOk (input:Result<_,_>) : bool =
        match input with
        | Ok _ -> true
        | _ -> false
    
    let matchError (error:LoginError) (input:Result<_,LoginError>) =
        match input with
        | Error ex -> ex = error
        | _ -> false  

    let runWithValidPassword (username:string) = 
        login username ValidPassword |> Async.RunSynchronously
    
    let success =
        let result = runWithValidPassword ValidUser
        result |> isOk 

    let badPassword =
        let result = login ValidUser BadPassword |> Async.RunSynchronously
        result |> matchError InvalidPwd

    let invalidUser =
        runWithValidPassword NotValidUser
        |> matchError InvalidUser

    let isSuspended =
        runWithValidPassword SuspendedUser
        |> matchError (UserBannedOrSuspended |> Unauthorized)

    let isBanned =
        let result = runWithValidPassword BannedUser
        result |> matchError (UserBannedOrSuspended |> Unauthorized)

    let hasBadLuck =
        let result = runWithValidPassword BadLuckUser
        result |> matchError (AuthErrorMessage |> BadThingHappened |> TokenErr)
```

Replace the code in the *Program.fs* `main` function with the following:

```fsharp
printfn "Success: %b" success
printfn "BadPassword: %b" badPassword
printfn "InvalidUser: %b" invalidUser
printfn "IsSuspended: %b" isSuspended
printfn "IsBanned: %b" isBanned
printfn "HasBadLuck: %b" hasBadLuck
0
```

Run the program from the Terminal using `dotnet run`. You should see successful asserts.

## Debugging Code

The observant amongst you may have noticed that the ionide extension supports debugging. It's something that F# developers don't do very often as pure functions and F# Interactive (FSI) are more convenient. You can add a breakpoint as you would in other IDEs by clicking in the border to the left of the code.

If you put a breakpoint in the `createAuthToken` function on the if expression and debug the code by pressing the green button, you will see that you only hit the breakpoint twice, on the `success` and `hasbadluck` cases. Once the running code is on the *Error* track, it won't run any of the remaining code on the *Ok* track.

## Further Reading

Computation Expressions are very useful tools to have at your disposal and are well worth investigating further. We have only looked at one usage which is dealing with effects but they can also be used for creating Domain-Specific Languages (DSL). Examples of this that you can investigate are **[Saturn](https://saturnframework.org/)** and **[Farmer](https://compositionalit.github.io/farmer/)**. 

It is important that you learn more about how F# handles asynchronous code with *Async* and how it interacts with the *Task* type from .Net Core. You can find out more at the following link:

<https://docs.microsoft.com/en-us/dotnet/fsharp/tutorials/asynchronous-and-concurrent-programming/async>

## Summary

In this chapter, we have used computation expressions in F#. They can be a little confusing to understand but provide a generic approach to dealing with effects, simplify our code, and tend to deliver extremely readable code.

In the next chapter, we are going to start to put some of the skills we have developed throughout this book into practical use by looking at how to create APIs and websites with the Giraffe library.