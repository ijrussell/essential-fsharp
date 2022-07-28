# 6 - Reading Data From a File

In this chapter, we will introduce the basics of reading and parsing external data using sequences and the Seq module and learn how we can isolate code that talks to external services to make our codebase more testable.

## Setting Up

In a new folder for this chapter, create a console application by typing the following into a VS Code Terminal window:

```plaintext
dotnet new console -lang F#
```

Using the Explorer view, create a folder called *resource*s in the code folder for this chapter and then create a new file called *customers.csv*. Copy the following data into the new file:

```text
CustomerId|Email|Eligible|Registered|DateRegistered|Discount
John|john@test.com|1|1|2015-01-23|0.1
Mary|mary@test.com|1|1|2018-12-12|0.1
Richard|richard@nottest.com|0|1|2016-03-23|0.0
Sarah||0|0||
```

Remove all the code from *Program.fs* except the following:

```fsharp
open System

[<EntryPoint>]
let main argv =
    0
```

As of F# 6, the *EntryPoint* attribute is no longer necessary.

## Loading Data

We are going to use the .NET System.IO classes to open the *customers.csv* file:

```fsharp
open System.IO
```

To load the data, we need to create a function that takes a path as a string, reads the file contents, and returns a collection of lines as strings from the file. Let's start with reading from a file:

```fsharp
let readFile path = // string -> seq<string>
    seq { 
        use reader = new StreamReader(File.OpenRead(path))
        while not reader.EndOfStream do
            reader.ReadLine() 
    }
```

There are a few new things in this simple function!

The `seq {...}` type is called a Sequence Expression. The code inside the curly brackets will create a sequence of strings. We could have done the same with *list* and *array* but most of the System.IO methods output `IEnumerable<'T>` and `seq {...}` is the F# equivalent to `IEnumerable<'T>`.

The *StreamReader* class implements the `IDisposable<'T>` interface. Just like in C#, we need to make sure the code calls the `Dispose()` method when it has finished using it. To do this, we use the `use` keyword which is the equivalent of `using` in C#. The indenting defines the scope of the disposable; in this case, the `Dispose()` method will be called when the sequence has completed being generated. It is required that we use the `new` keyword when we create instances that implement `IDisposable<'T>`.

We need to write some code in the main function to call our `readFile` function and output the data to the output window;

```fsharp
[<EntryPoint>]
let main argv =
    let path = Path.Combine(__SOURCE_DIRECTORY__, "resources", "customers.csv")
    let data = readFile path
    data |> Seq.iter (fun x -> printfn "%s" x)
    0
```

You must leave the `0` at the end of the main function as this tells dotnet that the process has been completed successfully.

We are using a built-in constant, `__SOURCE_DIRECTORY__`, to determine the current source code directory.

The Seq module has a wide range of functions available, similar to the List and Array modules. The `Seq.iter` function will iterate over the sequence, perform an action, and return *unit*.

The code in *Program.fs* should now look like this:

```fsharp
open System
open System.IO

// string -> seq<string>
let readFile path = 
    seq { 
        use reader = new StreamReader(File.OpenRead(path))
        while not reader.EndOfStream do
            reader.ReadLine() 
    }

[<EntryPoint>]
let main argv =
    let path = Path.Combine(__SOURCE_DIRECTORY__, "resources", "customers.csv")
    let data = readFile path
    data |> Seq.iter (fun x -> printfn "%s" x)
    0
```

Run the code by typing `dotnet run` in the Terminal.

To handle potential errors from loading a file, we are going to add some error handling to the *readFile* function:

```fsharp
// string -> Result<seq<string>,exn>
let readFile path = 
    try
        seq { 
            use reader = new StreamReader(File.OpenRead(path))
            while not reader.EndOfStream do
                reader.ReadLine() 
        }
        |> Ok
    with
    | ex -> Error ex
```

To handle the change in the signature of the readFile function, we will introduce a new function:

```fsharp
let import path =
    match path |> readFile with
    | Ok data -> data |> Seq.iter (fun x -> printfn "%A" x)
    | Error ex -> printfn "Error: %A" ex.Message
```

Replace the code in the main function with:

```fsharp
[<EntryPoint>]
let main argv =
    let path = Path.Combine(__SOURCE_DIRECTORY__, "resources", "customers.csv")
    import path
    0
```

We should simplify this code whilst we're here:

```fsharp
[<EntryPoint>]
let main argv =
    Path.Combine(__SOURCE_DIRECTORY__, "resources", "customers.csv")
    |> import
    0
```

Run the program to check that it still works by typing `dotnet run` in the Terminal.

Sadly, we have introduced a subtle bug here when we have an exception; the exception will not get caught because `seq {...}` is lazily evaluated. If you want to test it out, try a path that does not exist. Thankfully, it is very easy to fix:

```fsharp
// string -> Result<seq<string>,exn>
let readFile path = 
    try
        File.ReadLines(path)
        |> Ok
    with
    | ex -> Error ex
```

## Parsing Data

Now that we can load the data, we should try to convert each line to an F# record:

```fsharp
type Customer = {
    CustomerId : string
    Email : string
    IsEligible : string
    IsRegistered : string
    DateRegistered : string
    Discount : string
}
```

It is good practice to put types definitions above the let bindings in a code file.

Create a function that takes a `seq<string>` as input, parses the input data, and returns a `seq<Customer>` as output:

```fsharp
// seq<string> -> seq<Customer>
let parse (data:string seq) = 
    data
    |> Seq.skip 1 // Ignore the header row
    |> Seq.map (fun line -> 
        match line.Split('|') with
        | [| customerId; email; eligible; registered; dateRegistered; discount |] -> 
            Some { 
                CustomerId = customerId
                Email = email
                IsEligible = eligible
                IsRegistered = registered
                DateRegistered = dateRegistered
                Discount = discount
            }
        | _ -> None
    )
    |> Seq.choose id // Ignore None and unwrap Some
```

There are some new features in this function:

The `Seq.skip` function ignores a number of lines. In this case, the first item in the sequence is a header row and not a Customer, so it should be ignored. There is a potential bug here as `Seq.skip` is a partial function that will raise an exception if the input sequence is empty. We will fix this later in the chapter.

The `Split` function creates an array of strings from a string. We then pattern match the array and get the data which we then use to populate a Customer. If you aren't interested in all of the data, you can use the wildcard (_) for those parts. The number of columns in the array must match the number of identifiers we have used. In this case, we have defined six identifiers as we expect the row to be split into six parts.

We have now met the three primary collection types in F#: List (`[..]`), Seq (`seq {..}`), and Array (`[|..|]`).

The `Seq.choose` function will ignore any item in the sequence that is None and will unwrap the Some items to return a sequence of Customers. Remember that the *id* keyword is the identity function, a built-in value that is equivalent to the anonymous function `(fun x -> x)`.

We need to modify the import function to use the new parse function:

```fsharp
let import path =
    match path |> readFile with
    | Ok data -> data |> parse |> Seq.iter (fun x -> printfn "%A" x)
    | Error ex -> printfn "Error: %A" ex.Message
```

The next stage is to extract the code from the `Seq.map` in the `parse` function into its own function:

```fsharp
// string -> Customer option
let parseLine (line:string) : Customer option =
    match line.Split('|') with
    | [| customerId; email; eligible; registered; dateRegistered; discount |] -> 
        Some { 
            CustomerId = customerId
            Email = email
            IsEligible = eligible
            IsRegistered = registered
            DateRegistered = dateRegistered
            Discount = discount
        }
    | _ -> None
```

Modify the `parse` function to use the new `parseLine` function:

```fsharp
let parse (data:string seq) =
    data
    |> Seq.skip 1
    |> Seq.map (fun x -> parseLine x)
    |> Seq.choose id
```

We can simplify this function by removing the lambda, just like a Method Group in C#:

```fsharp
let parse (data:string seq) =
    data
    |> Seq.skip 1
    |> Seq.map parseLine
    |> Seq.choose id
```

## Testing the Code

Whilst we have improved the code a lot, it is difficult to test without having to load a file. The signature of the `readFile` function is `string -> Result<seq<string>>,exn>` which means that we could easily convert it to use a webservice or a fake service for testing rather than a path to a file on disk.

To make this testable and extensible, we can make import a higher-order function by taking a function as a parameter with the same signature as `readFile`. We should set the name of the input parameter to *dataReader* to reflect the possibility of different sources of data than a file:

```fsharp
let output data =
    data 
    |> Seq.iter (fun x -> printfn "%A" x)

let import (dataReader:string -> Result<string seq,exn>) path =
    match path |> dataReader with
    | Ok data -> data |> parse |> output
    | Error ex -> printfn "Error: %A" ex.Message
```

We can now pass any function with this signature `string -> Result<string seq,exn>`into the import function.

This signature is quite simple but they can get quite complex. We can create a Type Abbreviation with the same signature and use that in the parameter instead:

```fsharp
type DataReader = string -> Result<string seq,exn>
```

Replace the function signature in `import` with it;

```fsharp
let import (dataReader:DataReader) path =
    match path |> dataReader with
    | Ok data -> data |> parse |> output
    | Error ex -> printfn "Error: %A" ex.Message
```

We can use the type abbreviation like an Interface with exactly one function in the *readFile* function but it does mean modifying our code a little to use an alternate function style:

```fsharp
//string -> Result<string seq,exn>
let readFile : DataReader =
    fun path ->
        try
            File.ReadLines(path)
            |> Ok
        with
        | ex -> Error ex
```

We need to make a small change to our call in main to tell it to use the readFile function as we have changed the signature of the import function:

```fsharp
[<EntryPoint>]
let main argv =
    Path.Combine(__SOURCE_DIRECTORY__, "resources", "customers.csv")
    |> import readFile
    0
```

If we use import with *readFile* regularly, we can use partial application to create a new function that does that for us:

```fsharp
let importWithFileReader = import readFile
```

To use it we would simply call:

```fsharp
Path.Combine(__SOURCE_DIRECTORY__, "resources", "customers.csv")
|> importWithFileReader 
```

The payoff for the work we have done to use higher order functions and  type abbreviations is that we can easily pass in a fake function with known data for testing:

```fsharp
let fakeDataReader : DataReader =
    fun _ ->
        seq {
            "CustomerId|Email|Eligible|Registered|DateRegistered|Discount"
            "John|john@test.com|1|1|2015-01-23|0.1"
            "Mary|mary@test.com|1|1|2018-12-12|0.1"
            "Richard|richard@nottest.com|0|1|2016-03-23|0.0"
            "Sarah||0|0||"
        }
        |> Ok

import fakeDataReader "_"
```

You can use any function that satisfies the *DataReader* function signature.

## Final Code

What we have ended up with is the following;

```fsharp
open System.IO

type Customer = {
    CustomerId : string
    Email : string
    IsEligible : string
    IsRegistered : string
    DateRegistered : string
    Discount : string
}

type DataReader = string -> Result<string seq,exn>

let readFile : DataReader =
    fun path ->
        try
            File.ReadLines(path)
            |> Ok
        with
        | ex -> Error ex

let parseLine (line:string) : Customer option =
    match line.Split('|') with
    | [| customerId; email; eligible; registered; dateRegistered; discount |] -> 
        Some { 
            CustomerId = customerId
            Email = email
            IsEligible = eligible
            IsRegistered = registered
            DateRegistered = dateRegistered
            Discount = discount
        }
    | _ -> None

let parse (data:string seq) =
    data
    |> Seq.skip 1
    |> Seq.map parseLine
    |> Seq.choose id

let output data =
    data 
    |> Seq.iter (fun x -> printfn "%A" x)

let import (dataReader:DataReader) path =
    match path |> dataReader with
    | Ok data -> data |> parse |> output
    | Error ex -> printfn "Error: %A" ex.Message

[<EntryPoint>]
let main argv =
    Path.Combine(__SOURCE_DIRECTORY__, "resources", "customers.csv")    
    |> import readFile
    0
```

In a later chapter, we will extend this code by adding data validation but first, we have some tidying up to do to make our code more succinct and robust.

## Summary

In this chapter, we have looked at how we can import data using some of the most useful functions on the Seq module, sequence expressions, and function types. We have also seen how we can use higher order functions to allow us to easily extend functions to take different functions as parameters at runtime and for tests.

In the next chapter, we will look at another exciting F# feature, Active Patterns, to help make our pattern matching more readable.