# 1 - A simple Domain Modelling Exercise

This book will introduce you to the world of programming in F#. Rather than begin with a simple Hello World example or even some theory, I thought that I'd start with a typical business problem and look at how we can use some of the features of F# to solve it.

> **F# Interactive**
>
> We will be using F# Interactive to compile and run our code in this chapter. If you are not aware of how to use this, please read the section on **Setting up your environment** in the Introduction.

Using VS Code, create a new folder to store the code from this chapter. Add a new file called *part1.fsx*. The .fsx file is an F# script file. We are going to use script files and run the code using F# Interactive rather than creating and running a console application via the dotnet CLI.

## The Problem

This problem comes from a post by [Chris Roff](https://medium.com/@bddkickstarter/functional-bdd-5014c880c935) where he looks at using F# and [Behaviour Driven Development](https://cucumber.io/docs/bdd/) together.

```markdown
Feature: Applying a discount
Scenario: Eligible Registered Customers get 10% discount when they spend £100 or more

Given the following Registered Customers
|Customer Id|Is Eligible|
|John       |true       |
|Mary       |true       |
|Richard    |false      |

When [Customer Id] spends [Spend]
Then their order total will be [Total]

Examples:
|Customer Id|   Spend|   Total|
|Mary       |   99.00|   99.00|
|John       |  100.00|   90.00|
|Richard    |  100.00|  100.00|
|Sarah      |  100.00|  100.00|

Notes:
Sarah is not a Registered Customer
Only Registered Customers can be Eligible
```

Along with some examples showing how you can verify that your code is working correctly, there are a number of domain-specific words and concepts that we may want to represent in our code. We will start with a simple but naive solution and then we'll see how F#'s types can help us make it much more domain-centric and as an added benefit, be less susceptible to bugs.

## Getting Started

Along with simple datatypes like string, decimal, and boolean, F# has a powerful Algebraic Type System (ATS). Think of these types as simple data structures that you can use to compose larger data structures. We define a type using the `type` keyword as shown with this `Tuple` type:

```fsharp
type Customer = string * bool * bool
```

This definition states that a `Customer` type is a `Tuple` that consists of a `string` and two `booleans`. This means that a `Tuple` is defined as an `AND` type. Tuples in F# do not have named parts, so a consumer of your type would have to guess the meaning of the data. Note the use of the capital letter at the start of the name of the type, commonly referred to as `Pascal Case`. Most things other than type definitions use `Camel Case`, where the first letter is in lower case.

We create an instance of the `Customer` type using the `let` keyword:

```fsharp
// string * bool * bool
let fred = ("Fred", true, true)
```

The parentheses in this example are optional. Notice the difference between the definition which uses `*` and the usage which makes use of `,` to separate the parts of the *tuple*.

Using `let`, we have bound the data `("Fred", true, true)` to the identifier `fred`. By default, all data is immutable in F#, so you shouldn't think of `fred` as a variable.

We can explicitly state the type of data that the `fred` identifier is bound to by adding it to our *let binding*:

```fsharp
// Customer
let fred : Customer = ("Fred", true, true)
```

We can also use the `let` keyword to decompose the tuple:

```fsharp
let (id, isEligible, isRegistered) = fred
```

We have bound the identifiers `id`, `isEligible`, and `isRegistered` to the data in the `fred` binding.

Rather than use a *tuple*, we will create a *Record* type that solves the property name issue. We can define our initial customer record type like this:

```fsharp
type Customer = { Id:string; IsEligible:bool; IsRegistered:bool }
```

The record type, like the tuple, is an *AND* type, so in this example, a *Customer* consists of an Id which is a string value, and two boolean values called *IsEligible* and *IsRegistered*. Record types, like most of the types we will see through this book, are immutable, that is they cannot be changed once created. This means that all of the data to create a *Customer* record must be supplied when an instance is created.

Instead of defining the record type on a single line, we can also put each field on a separate line.

> WARNING: Tabs are not supported by F#, so your IDE/editor needs to be able to convert tabs to spaces, which most do including VS code.

```fsharp
type Customer = {
    Id : string
    IsEligible : bool
    IsRegistered : bool
}
```

Notice how the properties are aligned. F# makes use of *Significant Whitespace*, so alignment is important for scope. Alignment issues are one of the first things you should look at if the compiler warns you of an error in your code.

If you put the named values on separate lines, you no longer need to use the semi-colon separator. Spaces between the name value's label and type are allowed and they do not need to be consistent.

To create an instance of a customer we would write the following **below the type definition**:

```fsharp
let fred = { Id = "Fred"; IsEligible = true; IsRegistered = true } // Customer
```

The F# compiler has inferred the type of the value bound to the *fred* identifier to be a Customer. By using the let keyword, we have bound the identifier *fred* to that instance of a Customer. 

Another option that we have is to use the following style which is better when you have more properties to make your code easier to read:

```fsharp
let fred = { 
    Id = "Fred"
    IsEligible = true
    IsRegistered = true 
}
```

Note that we no longer need to use semi-colons to separate the property definitions.

You can annotate the type on the value binding if you like:

```fsharp
let fred : Customer = { Id = "Fred"; IsEligible = true; IsRegistered = true }
```

If there are multiple record types defined with the same structure, the compiler will not be able to infer the type, so we will need to annotate the identifier with the type.

We are not going to use the *fred* binding, so you can delete that line of code.

If you have used languages like C# or Java, you may have been surprised by the strict ordering of the code. The F# compiler works from the top of the file, downwards. It might seem a little odd at first but you soon get used to it. It has some really nice advantages for how we write and verify our code as well as making the compiler's job easier. This ordering applies at the project level too, so your source code (*.fs*) files need to be ordered in the same way, not alphabetically.

Below the *Customer* type, we need to create a function to calculate the total that takes a *Customer* and a *Spend* (decimal) as input parameters and returns the *Total* (decimal) as output:

```fsharp
// Customer -> decimal -> decimal
let calculateTotal (customer:Customer) (spend:decimal) : decimal =
    let discount = 
        if customer.IsEligible && spend >= 100.0M 
        then (spend * 0.1M) else 0.0M   
    let total = spend - discount
    total
```

The M suffix at the end of a number tells the compiler that the number is a decimal.

There are a few things to note about functions:

- We have used ```let``` again to define the function and inside the function to define the discount and total bindings.
- The discount and total bindings are scoped to the function and are not visible outside the function.
- There is no container, such as a class, because functions are first-class citizens.
- The return type is to the right of the input arguments.
- No return keyword is needed as the last line is returned automatically.
- Nesting using significant whitespace is used to define scope. Tabs are not allowed.
- We used an ```if``` expression that must return the same type for true and false routes.

> **Expressions** are used throughout F# programming since they always return an output. This makes them easy to compose with other expressions and easy to test. In contrast, statements do not return an output, so are not composable.

The comment above the function definition shows the function signature, which is **Customer -> decimal -> decimal**. The item at the end of the signature, after the last arrow, is the return type of the function. This function signature reads as *this function takes a Customer and a decimal as inputs and returns a decimal as output*. This is not strictly true as you will discover in the next chapter. Your IDE will normally show you the function signature or you can see it by hovering your mouse over the function name.

> **Function Signatures** are **very important**, as you will discover in the next chapter; get used to looking at them.

The input parameters of the *calculateTotal* function are in curried form. Curried parameters means that there is more than one of them. If a function signature has more than one arrow, then you have curried parameters. We can also arrange them in tupled form:

```fsharp
// (Customer * decimal) -> decimal
let calculateTotal (customer:Customer, spend:decimal) : decimal =
    let discount = 
        if customer.IsEligible && spend >= 100.0M 
        then (spend * 0.1M) else 0.0M   
    let total = spend - discount
    total
```

Note the change in the function signature from **Customer -> decimal -> decimal** to **(Customer * decimal) -> decimal**. The body of the function remains the same, it's just the input parameters that have changed and become a single tupled parameter. There are significant benefits to using the curried form for most F# functions, so that is the style that you will mostly see used in this book. We will discover some of the benefits and where the name came from in the next chapter.

The F# Compiler supports a feature called **Type Inference** which means that most of the time it can determine types through usage without you needing to explicitly define them. As a consequence, we can re-write the function as:

```fsharp
// Customer -> decimal -> decimal
let calculateTotal customer spend =
    let discount = 
        if customer.IsEligible && spend >= 100.0M then spend * 0.1M 
        else 0.0M   
    spend - discount
```

The IDE should still display the function signature as **Customer -> decimal -> decimal**. I also removed the total binding as I don't think it adds anything to the readability of the function.

> Don't forget to highlight the code you've written so far and press ALT + ENTER to run it in F# Interactive (FSI).

Whilst type inference works extremely well, there are some situations where it doesn't and you will need to provide a type to a parameter. One of the cases is where you are using a .NET function like ```DateTime.TryParse```. The problem here is that that function has two overrides and the compiler can't determine which one it needs to use.

Now that we have finished making changes to the function, we need to create a customer from our specification below the *calculateTotal* function and run in FSI:

```fsharp
let john = { Id = "John"; IsEligible = true; IsRegistered = true }
```

Rather than write a formal test, we can use FSI to run simple verifications for us. We will look at writing proper unit tests in chapter 4. Write the following after the *john* binding:

```fsharp
let assertJohn = (calculateTotal john 100.0M = 90.0M)
```

Highlight all of the code and press ALT + ENTER to run the code in FSI. What you should see is the following:

```fsharp
val assertJohn : bool = true
```

Add in the other users and test cases from the specification in the same way:

```fsharp
let john = { Id = "John"; IsEligible = true; IsRegistered = true } // Customer
let mary = { Id = "Mary"; IsEligible = true; IsRegistered = true } // Customer
let richard = { Id = "Richard"; IsEligible = false; IsRegistered = true } // Customer
let sarah = { Id = "Sarah"; IsEligible = false; IsRegistered = false } // Customer

let assertJohn = (calculateTotal john 100.0M = 90.0M) // bool
let assertMary = (calculateTotal mary 99.0M = 99.0M) // bool
let assertRichard = (calculateTotal richard 100.0M = 100.0M) // bool
let assertSarah = (calculateTotal sarah 100.0M = 100.0M) // bool
```

> **Multi-purpose ```=``` operator**
>
> There is no such thing as ```==``` or even ```===``` in F#. We use ```=``` for binding, setting property values, and equality. The one place where we don't use it is for updating values of mutable bindings. To update mutable bindings, we use the `<-` operator:
>
> ```fsharp
> let mutable myInt = 0
> myInt = 1 // false [equality check]
> myInt <- 1 // myInt now holds the value 1 [assignment]
> myInt = 1 // true [equality check]
> ```
>
> Notice that we have to explicitly define a binding as mutable.

Highlight the new code and press ALT + ENTER. You should see the following in FSI.

```fsharp
val assertJohn : bool = true
val assertMary : bool = true
val assertRichard : bool = true
val assertSarah : bool = true
```

We don't actually need to use the parentheses:

```fsharp
let assertJohn = calculateTotal john 100.0M = 90.0M
let assertMary = calculateTotal mary 99.0M = 99.0M
let assertRichard = calculateTotal richard 100.0M = 100.0M
let assertSarah = calculateTotal sarah 100.0M = 100.0M
```

To add clarity, we could add a new function that would perform the equality check for us:

```fsharp
// 'a -> 'a -> bool (' means generic)
let areEqual expected actual =
	actual = expected

let assertJohn = areEqual 90.0M (calculateTotal john 100.0M)
let assertMary = areEqual 99.0M (calculateTotal mary 99.0M)
let assertRichard = areEqual 100.0M (calculateTotal richard 100.0M)
let assertSarah = areEqual 100.0M (calculateTotal sarah 100.0M) 
```

The *areEqual* function is generic because the compiler has determined that any two values of the same type could be compared using the equality operator.

Using the first version of the asserts, our code should now look like this:

```fsharp
type Customer = {
    Id : string
    IsEligible : bool
    IsRegistered : bool
}

// Customer -> decimal -> decimal
let calculateTotal customer spend =
    let discount = 
        if customer.IsEligible && spend >= 100.0M then spend * 0.1M
        else 0.0M   
    spend - discount

let john = { Id = "John"; IsEligible = true; IsRegistered = true }
let mary = { Id = "Mary"; IsEligible = true; IsRegistered = true }
let richard = { Id = "Richard"; IsEligible = false; IsRegistered = true }
let sarah = { Id = "Sarah"; IsEligible = false; IsRegistered = false }

let assertJohn = (calculateTotal john 100.0M = 90.0M)
let assertMary = (calculateTotal mary 99.0M = 99.0M)
let assertRichard = (calculateTotal richard 100.0M = 100.0M)
let assertSarah = (calculateTotal sarah 100.0M = 100.0M)
```

Whilst this code works, having boolean properties representing domain concepts is not the most robust approach. In addition, it is possible to be in an invalid state where a customer could be eligible but not registered. To prevent this, we could have added a check for ```customer.IsRegistered = true``` but it is easy to forget to do this. Instead, we will take advantage of the F# type system and make domain concepts like *Registered* and *Unregistered* explicit.

## Making the Implicit Explicit

Create a new file called 'part2.fsx' and copy the final code from part1.fsx into it. We are going to modify the code in part2.fsx in this section.

Firstly, we create specific record types for *Registered* and *Unregistered* customers.

```fsharp
type RegisteredCustomer = {
    Id : string
    IsEligible : bool
}

type UnregisteredCustomer = {
	Id : string
}
```

To represent the fact that a customer can be either *Registered* or *Unregistered*, we will use another of the built-in types in the Algebraic Type System: the Discriminated Union (DU). We define the Customer type like this:

```fsharp
type Customer =
    | Registered of RegisteredCustomer
    | Guest of UnregisteredCustomer
```

This reads as "a customer can either be *Registered* of type *RegisteredCustomer* or a *Guest* of type *UnregisteredCustomer*". The items are called Union Cases and consist of a Case Identifier, in this example *Registered*/*Guest*, and some optional Case Data. You can optionally attach any type or mixture of types to a union case. 

Discriminated unions are closed sets, so only the cases described in the type definition are available. The only place that cases can be defined is in the type definition.

The easiest way to understand a discriminated union is to use them! We have to make changes to the users that we have defined. Firstly the *UnregisteredCustomer*:

```fsharp
// Guest of UnregisteredCustomer
let sarah = Guest { Id = "Sarah" } // Customer
```

In this example, *sarah* is still a Customer but you cannot create a Customer, you have to be one of the cases: Registered or Guest. 

Look at how the definition in the discriminated union compares to the case definition.

Now let's make the required changes to the *RegisteredCustomer* bindings:

```fsharp
// Registered of RegisteredCustomer
let john = Registered { Id = "John"; IsEligible = true }
let mary = Registered { Id = "Mary"; IsEligible = true }
let richard = Registered { Id = "Richard"; IsEligible = false }
```

Changing the *Customer* type to a discriminated union from a record also has an impact on the *calculateTotal* function. We will modify the discount calculation using another F# feature: Pattern Matching using a match expression:

```fsharp
let calculateTotal customer spend =
    let discount = 
        match customer with
        | Registered c -> 
            if c.IsEligible && spend >= 100.0M then spend * 0.1M else 0.0M
        | Guest _ -> 0.0M
    spend - discount
```

We are using a match expression to determine the case of the *Customer*. 

> **Expressions** produce an output value. Pretty much everything in F# including values, functions, and control flows is an expression. This allows them to be easily composed.

To understand what the pattern match is doing, compare the match ```Registered c``` with how we constructed the users: ```Registered { Id = "John"; IsEligible = true }```. In this case, ```c``` is a placeholder for the *RegisteredCustomer* case data. The underscore in the *Guest* pattern match is a wildcard that implies that we don't need access to that case data. Pattern matching against discriminated unions is exhaustive; that is, every case must be handled by the match expression. If you don't handle every case, you will get a warning from the compiler saying 'incomplete pattern match'.

We can simplify the logic with a guard clause but it does mean that we need to account for non-eligible Registered customers otherwise the match is incomplete:

```fsharp
let calculateTotal customer spend =
    let discount = 
        match customer with
        | Registered c when c.IsEligible && spend >= 100.0M -> spend * 0.1M
        | Registered _ -> 0.0M
        | Guest _ -> 0.0M
    spend - discount
```

We can simplify the last two matches using a wildcard like this:

```fsharp
let calculateTotal customer spend =
    let discount = 
        match customer with
        | Registered c when c.IsEligible && spend >= 100.0M -> spend * 0.1M
        | _ -> 0.0M
    spend - discount
```

> Be careful when using the wildcard like this because it may prevent the compiler from warning you of additions to the discriminated union and could result in unexpected outcomes.

The tests don't need to change.

This is much better than the naive version we had before. It's easier to understand the logic and you can no longer create data in an invalid state. Does it get better if we make eligibility explicit as well? Let's see!

## Going Further

Create a new file called *part3.fsx* and copy the final code from part2 into it. We are going to modify the code in *part3.fsx* in this section.

Now we are going to make eligibility a real domain concept. We remove the *IsEligible* flag from *RegisteredCustomer* and add *Eligible* to the *Customer* type.

```fsharp
type RegisteredCustomer = {
    Id : string
}

type UnregisteredCustomer = {
    Id : string
}

type Customer =
    | Eligible of RegisteredCustomer
    | Registered of RegisteredCustomer
    | Guest of UnregisteredCustomer
```

We need to make a change to our function.

```fsharp
let calculateTotal customer spend =
    let discount = 
        match customer with
        | Eligible _ when spend >= 100.0M -> spend * 0.1M
        | _ -> 0.0M
    spend - discount
```

We no longer need to test for IsEligible and we also no longer need access to the instance, so we can replace the 'c' with an underscore (wildcard).

We need to make some minor changes to two of our customer bindings:

```fsharp
let john = Eligible { Id = "John" }
let mary = Eligible { Id = "Mary" }
```

Run your code in FSI to check all is still OK.

The state of our code after all of our improvements is:

```fsharp
type RegisteredCustomer = {
	Id : string
}

type UnregisteredCustomer = {
    Id : string
}

type Customer =
    | Eligible of RegisteredCustomer
    | Registered of RegisteredCustomer
    | Guest of UnregisteredCustomer

let calculateTotal customer spend =
    let discount = 
        match customer with
        | Eligible _ when spend >= 100.0M -> spend * 0.1M
        | _ -> 0.0M
    spend - discount

let john = Eligible { Id = "John" }
let mary = Eligible { Id = "Mary" }
let richard = Registered { Id = "Richard" }
let sarah = Guest { Id = "Sarah" }

let assertJohn = calculateTotal john 100.0M = 90.0M
let assertMary = calculateTotal mary 99.0M = 99.0M
let assertRichard = calculateTotal richard 100.0M = 100.0M
let assertSarah = calculateTotal sarah 100.0M = 100.0M
```

I think that this is a worthwhile improvement over where we started as readability has improved and we have prevented getting into invalid states. However, we can still do better by replacing primitives with domain concepts; we will revisit this in a later chapter. Another improvement we will make is to add unit testing with XUnit where we can make use of the helpers and assertions we've already written.

Now that the *RegisteredCustomer* and *UnregisteredCustomer* records are really simple, we could remove them and add the string Id value directly to the case values in the Customer discriminated union:

```fsharp
type Customer =
    | Eligible of Id:string
    | Registered of Id:string
    | Guest of Id:string

let calculateTotal customer spend =
    let discount = 
        match customer with
        | Eligible _ when spend >= 100.0M -> spend * 0.1M
        | _ -> 0.0M
    spend - discount

let john = Eligible "John"
let mary = Eligible "Mary"
let richard = Registered "Richard"
let sarah = Guest "Sarah"

let assertJohn = calculateTotal john 100.0M = 90.0M
let assertMary = calculateTotal mary 99.0M = 99.0M
let assertRichard = calculateTotal richard 100.0M = 100.0M
let assertSarah = calculateTotal sarah 100.0M = 100.0M
```

I chose not to do this as it's likely that the *RegisteredCustomer* case data would contain more data than we currently see. That doesn't stop you from doing it only to the *UnregisteredCustomer* record type.

This was one approach to modelling this simple domain but we can do it in a few other styles as well.

## Alternative Approaches (1 of 2)

Create a new file called *part4.fsx* and copy the final code from *part3.fsx* into it. We are going to modify the code in *part4.fsx* in this section.

An alternative approach would be to use a discriminated union as a type in a property of a record type:

```fsharp
type CustomerType =
    | Registered of IsEligible:bool
    | Guest

type Customer = { Id:string; Type:CustomerType }

// Customer -> decimal -> decimal
let calculateTotal customer spend =
    let discount =
        match customer.Type with
        | Registered (IsEligible = isEligible) when isEligible && spend >= 100.0M -> spend * 0.1M
        | _ -> 0.0M
    spend - discount

let john = { Id = "John"; Type = Registered (IsEligible = true) }
let mary = { Id = "Mary"; Type = Registered (IsEligible = true) }
let richard = { Id = "Richard"; Type = Registered (IsEligible = false) }
let sarah = { Id = "Sarah"; Type = Guest }

let assertJohn = calculateTotal john 100.0M = 90.0M
let assertMary = calculateTotal mary 99.0M = 99.0M
let assertRichard = calculateTotal richard 100.0M = 100.0M
let assertSarah = calculateTotal sarah 100.0M = 100.0M
```

This is perfectly valid but it means introducing language that isn't in our current understanding of the domain. 

We can simplify the filter like this:

```fsharp
let calculateTotal customer spend =
    let discount =
        match customer.Type with
        | Registered (IsEligible = true) when spend >= 100.0M -> spend * 0.1M
        | _ -> 0.0M
    spend - discount

```

It solves the illegal state issue nicely but I don't like having to invent names. I much prefer to make my code as domain-centric as possible.

## Alternative Approaches (2 of 2)

Remember when we discounted tuples earlier as they don't allow the individual parts to be named? Discriminated unions have their own data structures that solve that. Although they look like tuples, the data structures of discriminated unions are not tuples.

Create a new file called *part5.fsx* and copy the final code from *part2.fsx* into it. We are going to modify the code in *part5.fsx* in this section. 

Let's start by modifying the *Customer* type and the *calculateTotal* function:

```fsharp
type Customer =
    | Registered of Id:string * IsEligible:bool
    | Guest of Id:string

// Customer -> decimal -> decimal
let calculateTotal customer spend =
    let discount =
        match customer with
        | Registered (id, isEligible) when isEligible && spend >= 100.0M -> spend * 0.1M
        | _ -> 0.0M
    spend - discount
```

You don't need to touch the asserts but we do need to change the let bindings to support the new Customer data structure:

```fsharp
let john = Registered (Id = "John", IsEligible = true)
let mary = Registered (Id = "Mary", IsEligible = true)
let richard = Registered (Id = "Richard", IsEligible = false)
let sarah = Guest (Id = "Sarah")
```

Highlight all of the code in this script file and run it in FSI to verify that the asserts all return true.

If we don't need the value of the Id property, we can use a wildcard to ignore it:

```fsharp
// Customer -> decimal -> decimal
let calculateTotal customer spend =
    let discount =
        match customer with
        | Registered (_, isEligible) when isEligible && spend >= 100.0M -> spend * 0.1M
        | _ -> 0.0M
    spend - discount

```

It is also possible to simplify the filter as we have done in previous versions:

```fsharp
// Customer -> decimal -> decimal
let calculateTotal customer spend =
    let discount =
        match customer with
        | Registered (IsEligible = true) when spend >= 100.0M -> spend * 0.1M
        | _ -> 0.0M
    spend - discount

```

If you need the value of the Id property, we can ask for it at the same time as applying the `IsEligible = true` filter:

```fsharp
// Customer -> decimal -> decimal
let calculateTotal customer spend =
    let discount =
        match customer with
        | Registered (Id = id; IsEligible = true) when spend >= 100.0M -> spend * 0.1M
        | _ -> 0.0M
    spend - discount
```

If you are looking at some of the filters we've written and are wondering if they could be extracted out to some kind of re-usable code, then you are in luck! The feature you are looking for is called Active Patterns. We will spend a whole chapter looking at them later in the book but this is an example of one being used in this version of our code:

```fsharp
// Customer -> unit option
let (|IsEligible|_|) customer =
    match customer with
    | Registered (IsEligible = true) -> Some ()
    | _ -> None

// Customer -> decimal -> decimal
let calculateTotal customer spend =
    let discount =
        match customer with
        | IsEligible when spend >= 100.0M -> spend * 0.1M
        | _ -> 0.0M
    spend - discount
```

You will meet Option, Some, None, and Unit in chapter 3 and Active Patterns in chapter 7. I just wanted to give you a sneak preview of some of the goodies you are going to encounter later in the book.

The guiding principle for domain modelling is to be as domain-centric as possible, whilst [making illegal states unrepresentable](https://fsharpforfunandprofit.com/posts/designing-with-types-making-illegal-states-unrepresentable/). Modelling is an inexact task at best. Try out a number of versions to see if there is a better way than you first thought of.

## Summary

In this chapter, we have combined types in many possible ways to solve a simple business problem. The types available in the F# Algebraic Type System, despite being simple AND (record and tuple) and OR (discriminated union) structures, can be combined together to build complicated data structures. Even more importantly, it can build incredibly expressive structures that model almost any kind of domain.

Despite the relative simplicity of the code we have created, we have covered quite a lot in this chapter:

- F# Interactive (FSI)
- Algebraic Type System
  - Tuples
  - Record Types
  - Discriminated Union
  - Type Composition
- Pattern Matching
  - Match Expression
  - Guard clause
- Let bindings
- Functions
- Function Signatures

In the next chapter, we will start to look at function composition: Composing bigger functions out of smaller ones.

## Postscript

To illustrate the portability of the functional programming concepts we have covered in this post, one of my colleagues, Daniel Weller, wrote a Scala version of one of the solutions:

```scala
sealed trait Customer

case class Registered(id : String) extends Customer
case class EligibleRegistered(id : String) extends Customer
case class Guest(id: String) extends Customer

def calculateTotal(customer: Customer)(spend: Double) = {
    val discount = customer match {
        case EligibleRegistered(_) if spend >= 100.0 => spend * 0.1
        case _ => 0.0
    }
    spend - discount
}

val john = EligibleRegistered("John")
val assertJohn = (calculateTotal (john) (100.0)) == 90.0
```

As you can see, it is fairly easy to see our solution in this unfamiliar Scala code.

You can find his code here -> <https://gist.github.com/frehn>