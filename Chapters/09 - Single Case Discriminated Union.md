# 9 - Single-Case Discriminated Union

In this chapter, we are going to improve the robustness and readability of our code by increasing our use of domain concepts and reducing our use of primitives. The primary approach adopted by the F# community for this task is the `Single-Case Discriminated Union`.

## Setting Up

We are going to improve some of the code we wrote in Chapter 1.

Create a new folder for the code in this chapter and open the new folder in VS Code.

Add a new file called *code.fsx*.

## Solving the Problem

This is where we left the code from the first chapter of this book:

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

// Customer -> decimal -> decimal
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

// 'a -> 'a -> bool
let isEqualTo expected actual =
    actual = expected

let assertJohn = calculateTotal john 100.0M |> isEqualTo 90.0M
let assertMary = calculateTotal mary 99.0M |> isEqualTo 99.0M
let assertRichard = calculateTotal richard 100.0M |> isEqualTo 100.0M
let assertSarah = calculateTotal sarah 100.0M |> isEqualTo 100.0M
```

Copy the code into *code.fsx*.

It's nice but we still have some primitives where we should have domain concepts (Spend and Total). I would like the signature of the `calculateTotal` function to change from `Customer -> decimal -> decimal` to `Customer -> Spend -> Total`. The easiest way to achieve this is to use `Type Abbreviations`, which are simple aliases:

```fsharp
type Spend = decimal
type Total = decimal
```

To use the new types, we simply decorate the `calculateTotal` function input parameters and output with them:

```fsharp
// Customer -> Spend -> Total
let calculateTotal (customer:Customer) (spend:Spend) : Total =
    let discount = 
        match customer with
        | Eligible _ when spend >= 100.0M -> spend * 0.1M
        | _ -> 0.0M
    spend - discount
```

An alternative would be to create a type abbreviation for the function signature:

```fsharp
type CalculateTotal = Customer -> Spend -> Total

//Customer -> Spend -> Total
let calculateTotal : CalculateTotal =
    fun customer spend ->
        let discount = 
            match customer with
            | Eligible _ when spend >= 100.0M -> spend * 0.1M
            | _ -> 0.0M
        spend - discount
```

Note the change in the way that input parameters are used with the type abbreviation.

Either approach gives us the signature we want. Both styles are useful to know, but for the rest of this chapter, we will use the first style without the type abbreviation for the function signature.

There is a potential problem with type abbreviations; if the underlying type matches, I can use anything for the input, not just the limited range of values expected for `Spend`. There is nothing stopping you from supplying either an invalid value or the wrong value to a parameter as shown in this example code with GPS coordinates:

```fsharp
type Latitude = decimal
type Longitude = decimal

type GpsCoordinate = { Latitude: Latitude; Longitude: Longitude }

// Latitude -90° to 90°
// Longitude -180° to 180°
let badGps : GpsCoordinate = { Latitude = 1000M; Longitude = -345M }

let latitude = 46M
let longitude = 15M

// Swap latitude and longitude
let badGps2 : GpsCoordinate = { Latitude = longitude; Longitude = latitude }
```

We can write tests to prevent this from happening but that is additional work. Instead, we want the type system to help us to write safer code and prevent this from happening. This is the role of features like the `Single-Case Discriminated Union`.

Let's define one for `Spend` by replacing the existing code with:

```fsharp
type Spend = Spend of decimal
```

If you hover your mouse over the first spend you will see the following tooltip:

```fsharp
union Spend =
    | Spend of decimal
```

It is the convention to omit the bar (`|`) for unions that contain only a single case.

You will notice that the `calculateTotal` function now has errors. We can fix that by deconstructing the `Spend` parameter value in the function using pattern matching:

```fsharp
let calculateTotal (customer:Customer) (spend:Spend) : Total =
    let (Spend value) = spend
    let discount = 
        match customer with
        | Eligible _ when value >= 100.0M -> value * 0.1M
        | _ -> 0.0M
    value - discount
```

If the `Spend` type had been defined as `type Spend = xSpend of decimal`, the deconstructor would have been `xSpend value` rather than `Spend value`.

We need to change the asserts to use the new type constructor:

```fsharp
let assertJohn = calculateTotal john (Spend 100.0M) |> isEqualTo 90.0M
let assertMary = calculateTotal mary (Spend 99.0M) |> isEqualTo 99.0M
let assertRichard = calculateTotal richard (Spend 100.0M) |> isEqualTo 100.0M
let assertSarah = calculateTotal sarah (Spend 100.0M) |> isEqualTo 100.0M
```

The `calculateTotal` function is now safer but less readable. There is a simple fix for this, we can deconstruct the value in the function parameter directly:

```fsharp
// Customer -> decimal -> decimal
let calculateTotal customer (Spend spend) = 
    let discount = 
        match customer with
        | Eligible _ when spend >= 100.0M -> spend * 0.1M
        | _ -> 0.0M
    spend - discount
```

If you replace all of your primitives and type abbreviations with single-case discriminated unions, you cannot supply the wrong parameter as the compiler will stop you. Of course, it doesn't stop the determined from abusing our code but it is another hurdle they must overcome.

The next improvement is to restrict the range of values that the `Spend` type can accept since very few domain values will be unbounded. We will restrict `Spend` to between 0.0M and 1000.0M. To support this, we are going to add a `ValidationError` type and prevent the direct use of the `Spend` constructor:

```fsharp
type ValidationError = 
    | InputOutOfRange of string

type Spend = private Spend of decimal
    with
        member this.Value = this |> fun (Spend value) -> value
        static member Create input = 
            if input >= 0.0M && input <= 1000.0M then
                Ok (Spend input)
            else
                Error (InputOutOfRange "You can only spend between 0 and 1000")
```

The use of the private accessor prevents code outside the containing module from directly accessing the type constructor. This means that to create an instance of `Spend`, we need to use the `Create` static member function defined on the type.

To extract the value, we use the `Value` member property defined on the instance. Although it is common to use `this` to describe the instance, it has no meaning in F#; `this` is just an identifier and we could have easily used `x` or `s`. If we hadn't used the identifier in the logic for extracting the value, we could have used the underscore `_`, although again it has no real meaning in this situation; it's just an identifier.

We need to make some changes to get the code to compile. Firstly we change the `calculateTotal` function to use the new type member functions:

```fsharp
// Customer -> 'a -> decimal
let calculateTotal customer spend =
    let discount = 
        match customer with
        | Eligible _ when spend.Value >= 100.0M -> spend.Value * 0.1M
        | _ -> 0.0M
    spend.Value - discount
```

Sadly, the compiler is unable to determine the type of the `spend` parameter using type inference, so we need to add the type abbreviation to it:

```fsharp
// Customer -> Spend -> decimal
let calculateTotal customer (spend:Spend) =
    let discount = 
        match customer with
        | Eligible _ when spend.Value >= 100.0M -> spend.Value * 0.1M
        | _ -> 0.0M
    spend.Value - discount
```

Which type annotations you apply to function parameters depends on how much the code is likely to change and whether it's important to retain the same signature. Type inference works very well most of the time and you should learn to rely on it but sometimes, it makes sense to specify the types.

We also need to fix the asserts since we are now returning a `Result<Spend,ValidationError>` instead of a `Spend`:

```fsharp
let isEqualTo expected actual =
    expected = actual

let assertEqual customer spent expected =
    Spend.Create spent
    |> Result.map (fun spend -> calculateTotal customer spend)
    |> isEqualTo (Ok expected)

let assertJohn = assertEqual john 100.0M 90.0M
let assertMary = assertEqual mary 99.0M 99.0M
let assertRichard = assertEqual richard 100.0M 100.0M
let assertSarah = assertEqual sarah 100.0M 100.0M
```

We have done some extra work to the `Spend` type but by doing so we have ensured that an instance of it can never be invalid.

As a piece of homework, think about how you would use the features we have covered in this chapter on the code from the last chapter.

## Adding Members to Types

Another change that we can make is to move the discount rate from the `calculateTotal` function to be with the `Customer` type definition. The primary reason for doing this would be for re-use:

```fsharp
type Customer =
    | Eligible of RegisteredCustomer
    | Registered of RegisteredCustomer
    | Guest of UnregisteredCustomer
    with 
        member this.Discount =
            match this with
            | Eligible _ -> 0.1M
            | _ -> 0.0M
```

This also allows us to simplify the `calculateTotal` function:

```fsharp
let calculateTotal (customer:Customer) (spend:Spend) =
    let discount = 
        if spend.Value >= 100.0M then spend.Value * customer.Discount 
        else 0.0M
    spend.Value - discount
```

Whilst this looks nice, it has broken the link between the `Customer` type and the `Spend` value. Remember, the rule is a 10% discount if an eligible customer spends 100.0 or more. Let's have another go:

```fsharp
type Customer =
    | Eligible of RegisteredCustomer
    | Registered of RegisteredCustomer
    | Guest of UnregisteredCustomer
    with 
        member this.CalculateDiscountPercentage(spend:Spend) =
            match this with
            | Eligible _ -> 
                if spend.Value >= 100.0M then 0.1M else 0.0M
            | _ -> 0.0M
```

> **WARNING** If the compiler shows an error, you will need to move the Spend type to before the Customer type declaration in the file as it is currently defined after it.

We now need to modify the `calculateTotal` function to use our new member function:

```fsharp
let calculateTotal (customer:Customer) (spend:Spend) =
    let discount = spend.Value * customer.CalculateDiscountPercentage(spend)
    spend.Value - discount
```

To run this code, you will need to load all of the code back into FSI using `ALT+ENTER` as we have changed quite a lot of code. Your tests should still pass.

A final change would be to simplify the `calculateTotal` function:

```fsharp
let calculateTotal customer spend =
    spend.Value * (1.0M - customer.CalculateDiscountPercentage(spend))
```

Having done all of this work, I'm going to suggest that we try to avoid using this approach and instead try to maintain a clean separation between data and behaviour. This isn't a rule, just strong guidance. Thankfully, there are other approaches that we can use instead.

## Using Modules

The last approach we will look at is to use a module with the same name as the type. This style is used throughout F#. Let's move our function to a new module:

```fsharp
type Customer =
    | Eligible of RegisteredCustomer
    | Registered of RegisteredCustomer
    | Guest of UnregisteredCustomer

module Customer =
    let calculateDiscountPercentage (spend:Spend) customer =
        match customer with
        | Eligible _ -> if spend.Value >= 100.0M then 0.1M else 0.0M
        | _ -> 0.0M
```

You may have noticed when dealing with other modules like `List` or `Option` that it is the convention to pass in the instance of the defining type as the last parameter.

We again need to modify our `calculateTotal` function:

```fsharp
let calculateTotal customer spend =
    let discountPercentage = 
        customer 
        |> Customer.calculateDiscountPercentage spend
    spend.Value * (1.0M - discountPercentage)
```

An alternative style would be to do the following, which some may find more readable:

```fsharp
let calculateTotal customer spend =
    customer 
    |> Customer.calculateDiscountPercentage spend
    |> fun discountPercentage -> spend.Value * (1.0M - discountPercentage)
```

It might also be a good idea to add the `calculateTotal` function to the *Customer* module:

```fsharp
type Customer =
    | Eligible of RegisteredCustomer
    | Registered of RegisteredCustomer
    | Guest of UnregisteredCustomer

module Customer =

    let calculateDiscountPercentage spend customer =
        match customer with
        | Eligible _ -> if Spend.Value spend >= 100.0M then 0.1M else 0.0M
        | _ -> 0.0M

    let calculateTotal spend customer =
        customer 
        |> calculateDiscountPercentage spend
        |> fun discountPercentage -> spend.Value * (1.0M - discountPercentage)
```

You can also move the functions for the `Spend` type to a module as well.

```fsharp
type Spend = private Spend of decimal

module Spend =
    
    let value input = input |> fun (Spend value) -> value
    
    let create input = 
        if input >= 0.0M && input <= 1000.0M then
            Ok (Spend input)
        else
            Error (InputOutOfRange "You can only spend between 0 and 1000")
```

We need to make a few small changes to our code to support these changes:

```fsharp
module Customer =

    let calculateDiscountPercentage spend customer =
        match customer with
        | Eligible _ -> if Spend.value spend >= 100.0M then 0.1M else 0.0M
        | _ -> 0.0M

    let calculateTotal spend customer =
        customer 
        |> calculateDiscountPercentage spend
        |> fun discountPercentage -> Spend.value spend * (1.0M - discountPercentage)
```

We also need to make one change to the assertion helper functions as we have moved the `calculateTotal` function to the Customer module:

```fsharp
let assertEqual customer spent expected =
    Spend.create spent
    |> Result.map (fun spend -> customer |> Customer.calculateTotal spend)
    |> isEqualTo (Ok expected)
```

If you run all of the code through FSI, your asserts will now pass again.

## A Few Minor Improvements

There are a few minor improvements that we can implement quite easily. Firstly, we will replace the string used in the `Customer` record types with a single-case discriminated union called `CustomerId`. We will add type abbreviations for `Total` and `DiscountPercentage`. Finally, we need to fix the compile errors. This is what we end up with:

```fsharp
type CustomerId = CustomerId of string

type RegisteredCustomer = {
    Id : CustomerId
}

type UnregisteredCustomer = {
    Id : CustomerId
}

type ValidationError = 
    | InputOutOfRange of string

type Spend = private Spend of decimal

module Spend =

    let value input = input |> fun (Spend value) -> value

    let create input = 
        if input >= 0.0M && input <= 1000.0M then
            Ok (Spend input)
        else
            Error (InputOutOfRange "You can only spend between 0 and 1000")

type Total = decimal
type DiscountPercentage = decimal

type Customer =
    | Eligible of RegisteredCustomer
    | Registered of RegisteredCustomer
    | Guest of UnregisteredCustomer

module Customer =

    // Spend -> Customer -> DiscountPercentage
    let calculateDiscountPercentage spend customer : DiscountPercentage =
        match customer with
        | Eligible _ -> if Spend.value spend >= 100.0M then 0.1M else 0.0M
        | _ -> 0.0M

    // Customer -> Spend -> Total
    let calculateTotal spend customer : Total =
        customer 
        |> calculateDiscountPercentage spend
        |> fun discountPercentage -> Spend.value spend * (1.0M - discountPercentage)

let john = Eligible { Id = CustomerId "John" }
let mary = Eligible { Id = CustomerId "Mary" }
let richard = Registered { Id = CustomerId "Richard" }
let sarah = Guest { Id = CustomerId "Sarah" }

let isEqualTo expected actual =
    expected = actual

let assertEqual customer spent expected =
    Spend.create spent
    |> Result.map (fun spend -> customer |> Customer.calculateTotal spend)
    |> isEqualTo (Ok expected)

let assertJohn = assertEqual john 100.0M 90.0M
let assertMary = assertEqual mary 99.0M 99.0M
let assertRichard = assertEqual richard 100.0M 100.0M
let assertSarah = assertEqual sarah 100.0M 100.0M
```

We now have quite a bit more code than we started with but it is much more robust and domain-centric which was our original goal for this chapter.

## Using a Record Type

You can also use a record type to serve the same purpose as the single-case discriminated union:

```fsharp
type Spend = private { Spend : decimal }

module Spend =
    
    let value input = input.Spend
    
    let create input = 
        if input >= 0.0M && input <= 1000.0M then
            Ok { Spend = input }
        else
            Error (InputOutOfRange "You can only spend between 0 and 1000")
```

## Summary

We have covered quite a lot in this chapter but reducing our usage of raw primitives and adding more domain-centric types will improve the quality of our code as well as making it more readable.

In this post we have learned about single-case discriminated unions. They allow us to restrict the range of values that a type can accept compared to raw primitives. We have also seen that we can extend types by adding helper functions and properties to them and the use of modules to do the same.

In the next chapter, we will look at object programming in F#.
