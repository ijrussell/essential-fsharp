# 8 - Functional Validation

In this chapter, we are going to look at adding validation to the code that we created in Chapter 6. We will make use of active patterns and we will see how you can easily model domain errors without using exceptions. At the end of the chapter, we will have a quick look at one of the more interesting functional patterns which is perfect for the validation that we are doing.

## Setting Up

Create a new folder for the code in this chapter.

Create a console application by typing the following into a terminal window:

```plaintext
dotnet new console -lang F#
```

Using the Explorer view, create a folder called *resources* in the code folder for this chapter and then create a new file called *customers.csv*. Copy the following data into the new file:

```text
CustomerId|Email|Eligible|Registered|DateRegistered|Discount
John|john@test.com|1|1|2015-01-23|0.1
Mary|mary@test.com|1|1|2018-12-12|0.1
Richard|richard@nottest.com|0|1|2016-03-23|0.0
Sarah||0|0||
```

Replace the code in *Program.fs* with the following which is where we left the code at the end of Chapter 6:

```fsharp
open System
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
            seq { 
                use reader = new StreamReader(File.OpenRead(path))
                while not reader.EndOfStream do
                    reader.ReadLine() 
            }
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

Run it using ```dotnet run``` in the terminal.

## Solving the Problem

We are going to take each unvalidated customer record, validate it, and if it is valid, convert it to a validated customer.

The first thing we need to do is create a new record type that will store our validated data. Add this type below the *Customer* record type definition:

```fsharp
type ValidatedCustomer = {
    CustomerId : string
    Email : string option
    IsEligible : bool
    IsRegistered : bool
    DateRegistered : DateTime option 
    Discount : decimal option
}
```

We need to add a new function to create a *ValidatedCustomer*:

```fsharp
// string -> string option -> bool -> bool -> DateTime option -> decimal option 
// -> ValidatedCustomer
let create customerId email isEligible isRegistered dateRegistered discount =
    {
        CustomerId = customerId
        Email = email
        IsEligible = isEligible
        IsRegistered = isRegistered
        DateRegistered = dateRegistered
        Discount = discount
    }
```

Now we need to think about how we handle validation errors. The first thing that we need to do is to consider what types of errors we expect. The obvious ones are missing data and invalid data. Let's create the validation error type as a discriminated union:

```fsharp
type ValidationError =
    | MissingData of name: string
    | InvalidData of name: string * value: string
```

For missing data, we only need the name of the item but for invalid data, we want the name and the value that failed.

You will need to add an import declaration at the top of the file so that we can use regular expressions:

```fsharp
open System.Text.RegularExpressions
```

We need to create some partial active patterns to handle parsing string data:

```fsharp
let (|ParseRegex|_|) regex str =
    let m = Regex(regex).Match(str)
    if m.Success then Some (List.tail [ for x in m.Groups -> x.Value ])
    else None

let (|IsValidEmail|_|) input =
    match input with
    | ParseRegex ".*?@(.*)" [ _ ] -> Some input
    | _ -> None

let (|IsEmptyString|_|) (input:string) =
    if input.Trim() = "" then Some () else None

let (|IsDecimal|_|) (input:string) = 
    let success, value = Decimal.TryParse input
    if success then Some value else None

let (|IsBoolean|_|) (input:string) =
    match input with 
    | "1" -> Some true 
    | "0" -> Some false
    | _ -> None
    
let (|IsValidDate|_|) (input:string) =
    let (success, value) = input |> DateTime.TryParse 
    if success then Some value else None
```

Now let's create validate functions using our active patterns and the new *ValidationError* discriminated union type:

```fsharp
// string -> Result<string, ValidationError>
let validateCustomerId customerId = 
    if customerId <> "" then Ok customerId
    else Error (MissingData "CustomerId")

// string -> Result<string option, ValidationError>
let validateEmail email = 
    if email <> "" then
        match email with
        | IsValidEmail _ -> Ok (Some email)
        | _ -> Error (InvalidData ("Email", email))
    else
        Ok None

// string -> Result<bool, ValidationError>
let validateIsEligible (isEligible:string) = 
    match isEligible with 
    | IsBoolean b -> Ok b 
    | _ -> Error (InvalidData ("IsEligible", isEligible))

// string -> Result<bool, ValidationError>
let validateIsRegistered (isRegistered:string) = 
    match isRegistered with 
    | IsBoolean b -> Ok b 
    | _ -> Error (InvalidData ("IsRegistered", isRegistered))

// string -> Result<DateTime option, ValidationError>
let validateDateRegistered (dateRegistered:string) = 
    match dateRegistered with
    | IsEmptyString -> Ok None
    | IsValidDate dt -> Ok (Some dt)
    | _ -> Error (InvalidData ("DateRegistered", dateRegistered))

// string -> Result<decimal option, ValidationError>
let validateDiscount discount = 
    match discount with
    | IsEmptyString -> Ok None
    | IsDecimal value -> Ok (Some value)
    | _ -> Error (InvalidData ("Discount", discount))
```

We now need to create a function that uses our new validation functions on the properties of the customer record and creates a *ValidatedCustomer* instance: 

```fsharp
let validate (input:Customer) : Result<ValidatedCustomer, ValidationError list> =
    let customerId = input.CustomerId |> validateCustomerId
    let email = input.Email |> validateEmail
    let isEligible = input.IsEligible |> validateIsEligible
    let isRegistered = input.IsRegistered |> validateIsRegistered
    let dateRegistered = input.DateRegistered |> validateDateRegistered
    let discount = input.Discount |> validateDiscount
    // This won't compile
    create customerId email isEligible isRegistered dateRegistered discount 
```

Notice that we have added the expected return type to the new function. This is good practice with some functions since it forces us to fix any issues to make the code compile.

We can see that we have a problem; the `create` function isn't expecting *Result* types from the validation functions. With the skills and knowledge that we have gained so far, we can solve this.

Firstly, we create a couple of helper functions to extract *Error* and *Ok* data from each validation function:

```fsharp
// Result<'a, ValidationError> -> ValidationError list
let getError input =
    match input with
    | Ok _ -> []
    | Error ex -> [ ex ]

// Result<'a, ValidationError> -> 'a
let getValue input =
    match input with
    | Ok v -> v
    | _ -> failwith "Oops, you shouldn't have got here!"
```

Now we create a list of potential errors using a list comprehension, concatenate them using `List.concat`, and then check to see if the result has any errors. If there are no errors, we can safely call the create function:

```fsharp
let validate (input:Customer) : Result<ValidatedCustomer, ConversionError list> =
    let customerId = input.CustomerId |> validateCustomerId
    let email = input.Email |> validateEmail
    let isEligible = input.IsEligible |> validateIsEligible
    let isRegistered = input.IsRegistered |> validateIsRegistered
    let dateRegistered = input.DateRegistered |> validateDateRegistered
    let discount = input.Discount |> validateDiscount
    let errors = 
        [ 
            customerId |> getError
            email |> getError
            isEligible |> getError
            isRegistered |> getError
            dateRegistered |> getError
            discount |> getError
        ] 
        |> List.concat
    match errors with
    | [] -> Ok (create (customerId |> getValue) (email |> getValue) (isEligible |> getValue) (isRegistered |> getValue) (dateRegistered |> getValue) (discount|> getValue))
    | _ -> Error errors
```

Finally, we need to plug the validation into the pipeline:

```fsharp
// seq<string> -> seq<Result<ValidatedCustomer, ValidationError list>>
let parse (data:string seq) = 
    data
    |> Seq.skip 1
    |> Seq.map parseLine
    |> Seq.choose id
    |> Seq.map validate
```

If you run the code using 'dotnet run' in the terminal, you should get some validated customer data as output.

Add an extra row to the customers.csv file that will fail validation:

```plaintext
|||||
```

If you run the code again, the last item in the output to the terminal will be an error.

## Where are we now?

This is the code we have ended up with:

```fsharp
open System
open System.IO
open System.Text.RegularExpressions

type Customer = {
    CustomerId : string
    Email : string
    IsEligible : string
    IsRegistered : string
    DateRegistered : string
    Discount : string
}

type ValidatedCustomer = {
    CustomerId : string
    Email : string option
    IsEligible : bool
    IsRegistered : bool
    DateRegistered : DateTime option 
    Discount : decimal option
}

type ValidationError =
| MissingData of name: string
| InvalidData of name: string * value: string

type FileReader = string -> Result<string seq, exn>

let readFile : FileReader =
    fun path ->
        try
            seq { 
                use reader = new StreamReader(File.OpenRead(path))
                while not reader.EndOfStream do
                    yield reader.ReadLine() 
            }
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

let (|ParseRegex|_|) regex str =
    let m = Regex(regex).Match(str)
    if m.Success then Some (List.tail [ for x in m.Groups -> x.Value ])
    else None

let (|IsValidEmail|_|) input =
    match input with
    | ParseRegex ".*?@(.*)" [ _ ] -> Some input
    | _ -> None

let (|IsEmptyString|_|) (input:string) =
    if input.Trim() = "" then Some () else None

let (|IsDecimal|_|) (input:string) = 
    let success, value = Decimal.TryParse input
    if success then Some value else None

let (|IsBoolean|_|) (input:string) =
    match input with 
    | "1" -> Some true 
    | "0" -> Some false
    | _ -> None

// string -> Result<string, ValidationError>
let validateCustomerId customerId = 
    if customerId <> "" then Ok customerId
    else Error (MissingData "CustomerId")

// string -> Result<string option, ValidationError>
let validateEmail email = 
    if email <> "" then
        match email with
        | IsValidEmail _ -> Ok (Some email)
        | _ -> Error (InvalidData ("Email", email))
    else
        Ok None

// string -> Result<bool, ValidationError>
let validateIsEligible (isEligible:string) = 
    match isEligible with 
    | IsBoolean b -> Ok b 
    | _ -> Error (InvalidData ("IsEligible", isEligible))

// string -> Result<bool, ValidationError>
let validateIsRegistered (isRegistered:string) = 
    match isRegistered with 
    | IsBoolean b -> Ok b 
    | _ -> Error (InvalidData ("IsRegistered", isRegistered))

// string -> Result<DateTime option, ValidationError>
let validateDateRegistered (dateRegistered:string) = 
    match dateRegistered with
    | IsEmptyString -> Ok None
    | IsValidDate dt -> Ok (Some dt)
    | _ -> Error (InvalidData ("DateRegistered", dateRegistered))

// string -> Result<decimal option, ValidationError>
let validateDiscount discount = 
    match discount with
    | IsEmptyString -> Ok None
    | IsDecimal value -> Ok (Some value)
    | _ -> Error (InvalidData ("Discount", discount))

let getError input =
    match input with
    | Ok _ -> []
    | Error ex -> [ ex ]

let getValue input =
    match input with
    | Ok v -> v
    | _ -> failwith "Oops, you shouldn't have got here!"

let create customerId email isEligible isRegistered dateRegistered discount =
    {
        CustomerId = customerId
        Email = email
        IsEligible = isEligible
        IsRegistered = isRegistered
        DateRegistered = dateRegistered
        Discount = discount
    }

let validate (input:Customer) : Result<ValidatedCustomer, ValidationError list> =
    let customerId = input.CustomerId |> validateCustomerId
    let email = input.Email |> validateEmail
    let isEligible = input.IsEligible |> validateIsEligible
    let isRegistered = input.IsRegistered |> validateIsRegistered
    let dateRegistered = input.DateRegistered |> validateDateRegistered
    let discount = input.Discount |> validateDiscount
    let errors = 
        [ 
            customerId |> getError;
            email |> getError;
            isEligible |> getError;
            isRegistered |> getError;
            dateRegistered |> getError;
            discount |> getError
        ] 
        |> List.concat
    match errors with
    | [] -> Ok (create (customerId |> getValue) (email |> getValue) (isEligible |> getValue) 
    		(isRegistered |> getValue) (dateRegistered |> getValue) (discount|> getValue))
    | _ -> Error errors

let parse (data:string seq) =
    data
    |> Seq.skip 1
    |> Seq.map parseLine
    |> Seq.choose id
    |> Seq.map validate

let output data =
    data 
    |> Seq.iter (fun x -> printfn "%A" x)

let import (fileReader:FileReader) path =
    match path |> fileReader with
    | Ok data -> data |> parse |> output
    | Error ex -> printfn "Error: %A" ex

[<EntryPoint>]
let main argv =
    Path.Combine(__SOURCE_DIRECTORY__, "resources", "customers.csv")    
    |> import readFile
    0
```

This code works well but there is a more idiomatic way of handling this type of problem in F#, and most functional programming languages called Applicatives. It works in a similar way to our version but more elegantly.

## Functional Validation the F# Way

We are going to use the *Computation Expression* support for *Applicatives* that was introduced in F# 5. We are not going to deep dive into computation expressions in this chapter but will do so in Chapter 12. It is enough at this stage to know how they can be used.

> If you want to see an explanation of using *Applicatives* for validation using the idiomatic style prior to F# 5, have a look at the post on [Functional Validation in F# Using Applicatives](https://trustbit.tech/blog/2019/12/09/functional-validation-in-f-using-applicatives) that I wrote for the [2019 F# Advent Calendar](<https://sergeytihon.com/2019/11/05/f-advent-calendar-in-english-2019/>). It's well worth understanding how it works under the covers as it builds on many of the features we have encountered so far.

The first thing that we need to do is to convert each of our individual errors into *ValidationError* lists. We can use the built-in `Result.mapError` function to help us to do this:

```fsharp
let validate (input:Customer) : Result<ValidatedCustomer, ValidationError list> =
    let customerId = 
        input.CustomerId 
        |> validateCustomerId 
        |> Result.mapError (fun ex -> [ ex ])
    let email = 
        input.Email 
        |> validateEmail 
        |> Result.mapError (fun ex -> [ ex ])
    let isEligible = 
        input.IsEligible 
        |> validateIsEligible 
        |> Result.mapError (fun ex -> [ ex ])
    let isRegistered = 
        input.IsRegistered 
        |> validateIsRegistered 
        |> Result.mapError (fun ex -> [ ex ])
    let dateRegistered = 
        input.DateRegistered 
        |> validateDateRegistered 
        |> Result.mapError (fun ex -> [ ex ])
    let discount = 
        input.Discount 
        |> validateDiscount 
        |> Result.mapError (fun ex -> [ ex ])
    // Compile problem
    create customerId email isEligible isRegistered dateRegistered discount 
```

Rather than using `(fun ex -> [ ex ])` we could use another of the functions from the *List* module, `singleton`, which does the same thing:

```fsharp
let discount = 
	input.Discount 
	|> validateDiscount 
	|> Result.mapError List.singleton
```

The other change that we could make is to decide that our individual validation helper functions are only used by this `validate` function, in which case, we could make each one return a list of *ValidationError*. This would simplify the `validate` function:

```fsharp
let validate (input:Customer) : Result<ValidatedCustomer, ValidationError list> =
    let customerId = input.CustomerId |> validateCustomerId 
    let email = input.Email |> validateEmail 
    let isEligible = input.IsEligible |> validateIsEligible 
    let isRegistered = input.IsRegistered |> validateIsRegistered 
    let dateRegistered = input.DateRegistered |> validateDateRegistered 
    let discount = input.Discount |> validateDiscount 
    // Compile problem
    create customerId email isEligible isRegistered dateRegistered discount 
```

Which style you decide to take is not really that important but you should try to stick to one.

Now back to fixing our compile problem. 

We are going to use some code from the FsToolkit.ErrorHandling NuGet package. To install it in our project, we need to add the package via the Terminal:

```plaintext
dotnet add package FsToolkit.ErrorHandling
```

After the package has downloaded, add the following import declaration:

```fsharp
open FsToolkit.ErrorHandling.ValidationCE
```

To finish, we need to plug in the validation computation expression:

```fsharp
let validate (input:Customer) : Result<ValidatedCustomer, ValidationError list> =
    validation {
        let! customerId = 
            input.CustomerId 
            |> validateCustomerId 
            |> Result.mapError (fun ex -> [ ex ])
        and! email = 
            input.Email 
            |> validateEmail 
            |> Result.mapError (fun ex -> [ ex ])
        and! isEligible = 
            input.IsEligible 
            |> validateIsEligible 
            |> Result.mapError (fun ex -> [ ex ])
        and! isRegistered = 
            input.IsRegistered 
            |> validateIsRegistered 
            |> Result.mapError (fun ex -> [ ex ])
        and! dateRegistered = 
            input.DateRegistered 
            |> validateDateRegistered 
            |> Result.mapError (fun ex -> [ ex ])
        and! discount = 
            input.Discount 
            |> validateDiscount 
            |> Result.mapError (fun ex -> [ ex ])
        return create customerId email isEligible isRegistered dateRegistered discount 
    }
```

The key thing to notice is the `and` keyword that is new to F# in version 5.

The bang unwraps the value from the effect, in this case, the value from the *Ok* track of the validated item. Move your mouse over `customerId` and the tooltip will tell you it is a string. Remove the bang and the tooltip will inform you that *customerId* is a `Result<string, ValidationError list>`.

If we had used `let` rather than `and`, the code would have only returned the first error found and then would not have the other lines as it would be on the Error track. The style before `and` was introduced is referred to as *monadic*, as opposed to the *applicative* style we are now using where the use of `and` forces the code to run all of the validation even if there are validation errors reported. The final function to create a *ValidatedCustomer* is only called if there are no errors. It ends up doing exactly what we did initially but much more elegantly!

Computation expressions are the only place in F# where the `return` keyword is required.

Don't worry if you don't fully understand this code as we will be covering computation expressions in detail in Chapter 12 and when we start with web development in Chapter 13.

Check that everything is working by running the code from the terminal:

```plaintext
dotnet run
```

Applicatives are very useful for a range of things, so it's an important pattern to know, even in the old, pre-computation expression style.

## Summary

In this chapter, we have looked at how we can add validation by using active patterns and how easy it is to add additional functionality into the data processing pipeline. We also looked at a more elegant solution than our original one to the validation problem using applicatives with the computation expression support introduced in F# 5.

In the next chapter, we will look at improving the code from the first chapter by using more domain terminology and reducing the use of primitives.