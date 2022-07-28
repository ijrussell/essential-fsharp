# 4 - Organising Code and Testing

In this chapter, we will be looking at how we can use organise our code using .NET Solutions, Projects, Namespaces, and F# Modules. In addition, we'll be writing our first real unit tests in F# with XUnit.

## Getting Started

Follow the instructions in the Appendix of this book to create a new .NET solution containing two projects, a console for the code and an XUnit one for the tests.

Once completed, you should see a message like the following in the tests terminal:

```plaintext
Passed!  - Failed:     0, Passed:     1, Skipped:     0, Total:     1, Duration: < 1 ms - MyProjectTests.dll (net5.0)
```

Open a second Terminal window and navigate to the MyProject folder.

Execute the following command:

```plaintext
dotnet run
```

You should see a message like the following in the Terminal:

```plaintext
Hello world from F#
```

## Solutions and Projects

We have created a Solution to which we added two directories; src for code projects and test for test projects. The test project has to reference the code project to be able to test it.

## Adding a Source File

The last thing we need to do is add a new file called *Customer.fs*. We can do this in two ways. You will find it beneficial to know them both when working on F# in VS Code. We can use either the Explorer view (top icon in the toolbar) or the F# Explorer (Look for the F# logo).

**Using Explorer:**

If you manually add a file, you have to manually include it to the .fsproj file as well so that F# recognises it:

1. Select the *MyProject.fsproj* file.
1. Click on the New File icon and name the file *Customer.fs*.
1. Click on the *MyProject.fspro*j file to edit it. Copy the ```<Compile Include="Program.fs" />``` entry and paste it above or below the existing entry.
1. Change the file name of the uppermost entry to *Customer.fs* from *Program.fs*. 

The *MyProject.fsproj* project file should now contain this:

```xml
  <ItemGroup>
    <Compile Include="Customer.fs" />
    <Compile Include="Program.fs" />
  </ItemGroup>
```

> **Ordering is important**
>
> ```The strict ordering of code within a file is important and so is the ordering of files within a project. Files are compiled from the top down, just as code within a file is.```

**Using F# Solution Explorer (ionide):**

1. Right-click on Program.fs in the MyProject folder.
1. Select the Add File Above menu item and name the file Customer.fs.

Once your new file is created, run 'dotnet build' on the project or solution from the terminal and you will get an error like this:

```plaintext
Files in libraries or multiple-file applications must begin with a namespace or module declaration, e.g. 'namespace SomeNamespace.SubNamespace' or 'module SomeNamespace.SomeModule'. Only the last source file of an application may omit any declaration.
```

## Namespaces and Modules

Modules are how we logically organise our code in an F# project. Namespaces are used to reduce the chances of naming collisions when referencing code from other projects.

### Namespaces

- Can only contain type declarations, import declarations, or modules.
- Span across multiple files.
- Module names must be unique within a namespace even across multiple files.

### Modules

- Can contain anything apart from namespaces. You can even nest modules.
- You can have a top-level module instead of a namespace but it must be unique across all files in the project.

To fix our build issue, we are going to go add a namespace to the *Customer.fs* file in the *MyProject* project. Add the namespace declaration to the top of the file:

```fsharp
namespace MyProject
```

The name can be anything you like, it doesn't have to match the project name. If you run `dotnet build` in the Terminal again, the error has been fixed.

Let's look at some of the options when using namespaces and modules:

```fsharp
namespace MyProject.Customer // Namespace.Namespace

open System

type Customer = {
    Name : string
}

module Domain =

    // string -> Customer
    let create (name:string) =
        { Name = name }

module Db =

    open System.IO

    // Customer -> bool
    let save (customer:Customer) =
        // Imagine this talks to a database
        ()
```

The Customer record type is used by the two modules, so it is defined in the scope of the namespace. The modules have access to the types in the namespace without having to add an import declaration.

There are other combinations of namespace and modules available. Firstly, separate the namespace and module definition:

```fsharp
namespace MyApplication // Namespace

open System

module Customer = // Module
    
    type Customer = {
        Name : string
    }

    module Domain =

        // string -> Customer
        let create (name:string) =
            { Name = name }

    module Db =

        open System.IO

        // Customer -> unit
        let save (customer:Customer) =
            // Imagine this talks to a database
            ()
```

Notice the nesting required for the sub-modules.

We can also use the module keyword at the top level. In this case, *MyApplication* is a namespace and *Customer*, a module:

```fsharp
module MyApplication.Customer // Namespace.Module

open System

type Customer = {
    Name : string
}

module Domain =

    // string -> Customer
    let create (name:string) =
        { Name = name }

module Db =

    open System.IO

    // Customer -> unit
    let save (customer:Customer) =
        // Imagine this talks to a database
        ()
```

A good starting point is to add a namespace to the top of each file using the project name and use modules for everything else. 

Do not feel pressured into creating a file per module. It's generally better to keep as much code that changes together in the same file. This means organising your code by feature/domain concept rather than by technical concept as happens in MVC for instance.

## Writing Tests

Have a look at the *Tests.fs* file that was created when we generated the test project:

```fsharp
module Tests

open System
open Xunit

[<Fact>]
let ``My test`` () =
    Assert.True(true)
```

One of the really nice things about F# for naming things, particularly tests, is that we can use readable sentences if we use the double backticks. When you have test errors, the name of the module and the test appear in the output. 

There is generally no need for namespaces in test files as the code is not deployed.

> **WARNING:** If you use the double backtick naming style, do not use any odd chars like [,|-;.]. These used in test names causes the ionide extension to crash. [Correct: 2022-05-20]

Create a new file in the test project called *CustomerTests.fs* and add the following code to it:

```fsharp
namespace MyProjectTests

open System
open Xunit

module ``I can group my tests in a module and run`` =

    [<Fact>]
    let ``My first test`` () =
        Assert.True(true)

    [<Fact>]
    let ``My second test`` () =
        Assert.True(true)
```

Run the tests using the Terminal:

```plaintext
dotnet test
```

You can also use the test runner built into VS Code.

Change the second test to make it fail. 

```fsharp
    [<Fact>]
    let ``My second test`` () =
        Assert.True(false)
```

Run the tests and look at the output.

## Writing Real Tests

We are going to use the code from chapter 2 to write tests against. 

Write the following code into *Customer.fs*:

```fsharp
module MyProject.Customer

type Customer = {
    Id : int
    IsVip : bool
    Credit : decimal
}

// Customer -> (Customer * decimal)
let getPurchases customer =
    let purchases = if customer.Id % 2 = 0 then 120M else 80M
    (customer, purchases)

// (Customer * decimal) -> Customer
let tryPromoteToVip purchases =
    let (customer, amount) = purchases
    if amount > 100M then { customer with IsVip = true }
    else customer

// Customer -> Customer
let increaseCreditIfVip customer =
    let increase = if customer.IsVip then 100M else 50M
    { customer with Credit = customer.Credit + increase }

// Customer -> Customer
let upgradeCustomer customer = 
    customer
    |> getPurchases
    |> tryPromoteToVip
    |> increaseCreditIfVip
```

We can use the asserts from chapter 2 as the basis of our new tests:

```fsharp
let customerVIP = { Id = 1; IsVip = true; Credit = 0.0M }
let customerSTD = { Id = 2; IsVip = false; Credit = 100.0M }

let assertVIP =
	upgradeCustomer customerVIP = {Id = 1; IsVip = true; Credit = 100.0M }
let assertSTDtoVIP =
	upgradeCustomer customerSTD = {Id = 2; IsVip = true; Credit = 200.0M }
let assertSTD =
	upgradeCustomer { customerSTD with Id = 3; Credit = 50.0M } = {Id = 3; IsVip = false; Credit = 100.0M }
```

We should make some minor changes to help make the asserts easier to use as the basis of our tests:

```fsharp
let customerVIP = { Id = 1; IsVip = true; Credit = 0.0M }
let customerSTD = { Id = 2; IsVip = false; Credit = 100.0M }

let areEqual expected actual =
	actual = expected

let assertVIP =
	let expected = {Id = 1; IsVip = true; Credit = 100.0M }
	areEqual expected (upgradeCustomer customerVIP) 
let assertSTDtoVIP =
	let expected = {Id = 2; IsVip = true; Credit = 200.0M }
	areEqual expected (upgradeCustomer customerSTD) 
let assertSTD =
	let expected = {Id = 3; IsVip = false; Credit = 100.0M }
	areEqual expected (upgradeCustomer { Id = 3; IsVip = false; Credit = 50.0M })
```

Delete the code from *CustomerTests.fs* and write the following code into the new file:

```fsharp
namespace MyProjectTests

open Xunit
open MyProject.Customer

module ``When upgrading customer`` =

    let customerVIP = { Id = 1; IsVip = true; Credit = 0.0M }
    let customerSTD = { Id = 2; IsVip = false; Credit = 100.0M }
    
    // let assertVIP =
    //     let expected = {Id = 1; IsVip = true; Credit = 100.0M }
    //     areEqual expected (upgradeCustomer customerVIP) 
    [<Fact>]
    let ``should give VIP customer more credit`` () =
        let expected = { customerVIP with Credit = customerVIP.Credit + 100M }
        let actual = upgradeCustomer customerVIP
        Assert.Equal(expected, actual)    
    
    // let assertSTDtoVIP =
    //     let expected = {Id = 2; IsVip = true; Credit = 200.0M }
    //     areEqual expected (upgradeCustomer customerSTD) 
    // let assertSTD =
    //     let expected = {Id = 3; IsVip = false; Credit = 100.0M }
    //     areEqual expected (upgradeCustomer { customerSTD with Id = 3; Credit = 50.0M })
```

Run the tests by running `dotnet test` in the test project Terminal. They should pass.

Add the test replacements for the remaining two asserts.

```fsharp
namespace MyProjectTests

open Xunit
open MyProject.Customer

module ``When upgrading customer`` =

    let customerVIP = { Id = 1; IsVip = true; Credit = 0.0M }
    let customerSTD = { Id = 2; IsVip = false; Credit = 100.0M }
    
    // let assertVIP =
    //     let expected = {Id = 1; IsVip = true; Credit = 100.0M }
    //     areEqual expected (upgradeCustomer customerVIP) 
    [<Fact>]
    let ``should give VIP customer more credit`` () =
        let expected = { customerVIP with Credit = customerVIP.Credit + 100M }
        let actual = upgradeCustomer customerVIP
        Assert.Equal(expected, actual)    
    
    // let assertSTDtoVIP =
    //     let expected = {Id = 2; IsVip = true; Credit = 200.0M }
    //     areEqual expected (upgradeCustomer customerSTD)
    [<Fact>]
    let ``should convert eligible STD customer to VIP`` () =
        let expected = {Id = 2; IsVip = true; Credit = 200.0M }
        let actual = upgradeCustomer customerSTD
        Assert.Equal(expected, actual)
    
    // let assertSTD =
    //     let expected = {Id = 3; IsVip = false; Credit = 100.0M }
    //     areEqual expected (upgradeCustomer { customerSTD with Id = 3; Credit = 50.0M })
    [<Fact>]
    let ``should not upgrade ineligible STD customer to VIP`` () =
        let expected = {Id = 3; IsVip = false; Credit = 100.0M }
        let actual = upgradeCustomer { customerSTD with Id = 3; Credit = 50.0M }
        Assert.Equal(expected, actual)
```

Run the tests by running `dotnet test` in the test project Terminal. They should pass.

We are using Xunit as our test framework. There are other testing and assertion libraries available that you may prefer. 

## Using FsUnit for Assertions

This is an example of how an assertion would look if we used *FsUnit* instead of *XUnit*:

```fsharp
open FsUnit

upgraded |> should equal expected
```

Now we can replace our *XUnit* asserts with *FsUnit* ones:

```fsharp
    [<Fact>]
    let ``should give VIP customer more credit`` () =
        let expected = { customerVIP with Credit = customerVIP.Credit + 100M }
        let actual = upgradeCustomer customerVIP
        actual |> should equal expected

	[<Fact>]
    let ``should convert eligible STD customer to VIP`` () =
        let customer = { Id = 2; IsVip = false; Credit = 200M }
        let expected = { customer with IsVip = true; Credit = customer.Credit + 100M }
        let actual = upgradeCustomer customer
        actual |> should equal expected

    [<Fact>]
    let ``should not upgrade eligible STD customer to VIP`` () =
        let customer = { Id = 3; IsVip = false; Credit = 50M }
        let expected = { customer with Credit = customer.Credit + 50M }
        let actual = upgradeCustomer customer
        actual |> should equal expected
```

Run the tests by running `dotnet test` in the test project Terminal or the Solution Terminal. They should pass.

## Summary

The features we used in this chapter are important if we want to make working on larger F# codebases manageable.

- Solutions, projects, namespaces, and modules
- Unit testing with *XUnit*
- Assertions with *FsUnit*

In the next chapter, we will have an initial look at collections.