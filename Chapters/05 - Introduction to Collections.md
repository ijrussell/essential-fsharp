# 5 - Introduction to Collections

In this chapter, we will investigate functional collections and the helper modules that F# supplies out of the box.

## Before We Start

Create a new folder for this chapter.

## The Basics

Add a new script file called *lists.fsx* for the example code we are going to write in this section.

F# has a hugely deserved reputation for being great for data-centric use cases like finance and data science due in a large part to the power of its support for data structures and collection handling.

There are a number of types of collections in F# that we can make use of but the three primary ones are:

- *Seq* - A lazily evaluated collection.
- *Array* - Great for numerics/data science. There are built-in modules for 2d, 3d, and 4d arrays.
- *List* - Eagerly evaluated, linked list, with immutable structure and data.

> **F# vs .NET Collections**
>
> F# *Seq* is equivalent to .NET *IEnumerable<'T>*.
>
> The .NET *List<'T>* is called *ResizeArray<'T>* in F#.

Each of these types has a supporting module that contains a wide range of functions including the ability to convert to/from each other.

In this chapter, we are going to concentrate on the *List* type and module.

## Core Functionality

We will be using *lists.fsx* for this section. Remember to highlight the code and run it in F# Interactive (FSI) using ```ALT + ENTER```.

Create an empty list:

```fsharp
let items = []
```

Create a list with five integers:

```fsharp
let items = [2;5;3;1;4]
```

If they were sorted in ascending order, we could also do this:

```fsharp
let items = [1..5]
```

Or we could use a *List Comprehension*:

```fsharp
let items = [ 
    for x in 1..5 do 
        yield x 
]
```

Since F# 5, we have been able to drop the need for the ```yield``` keyword in most cases. We can also define it in one line:

```fsharp
let items = [ for x in 1..5 do x ] 
```

Comprehensions are really powerful. They're also available for the other primary collection types in F#: *Seq* and *Array*, but we are not going to use them in this chapter.

To add an item to a list, we use the cons operator (`::`):

```fsharp
// head :: tail
let extendedItems = 6::items // [6;1;2;3;4;5]
```

The original list remains unaffected by the new item as it is immutable. This is very efficient because *extendedItems* is the new item plus a pointer to the original *items* list.

A non-empty list is made up of a single item called the head and a list of items called the tail which could be an empty list ```[]```. We can pattern match on a list to show this:

```fsharp
let readList items =
    match items with
    | [] -> "Empty list"
    | [head] -> $"Head: {head}" // list containing one item
    | head::tail -> sprintf "Head: %A and Tail: %A" head tail 

let emptyList = readList [] // "Empty list"
let multipleList = readList [1;2;3;4;5] // "Head: 1 and Tail: [2;3;4;5]"
let singleItemList = readList [1] // "Head: 1"
```

The labels *head* and *tail* have no meaning; you could use anything such as *h::t* or *fred::ginger*; they are just labels. We also see three ways of outputing a string; directly, string interpolation using `$`, and `sprintf`.

> **Strings**
>
> There are multiple ways of outputting strings in F#. The primary two ways are `sprintf` and string interpolation:
>
> $"Head: %A{head} and Tail: %A{tail}"
>
> sprintf "Head: %A and Tail: %A" head tail 
>
> The `%A` formats the output using the types `toString()` method. You do not have to use formatting with interpolation.

If we remove the pattern match for the single item list, the code still works:

```fsharp
let readList items =
    match items with
    | [] -> "Empty list"
    | head::tail -> sprintf "Head: %A and Tail: %A" head tail

let emptyList = readList [] // "Empty list"
let multipleList = readList [1;2;3;4;5] // "Head: 1 and Tail: [2;3;4;5]"
let singleItemList = readList [1] // "Head: 1 and Tail: []"
```

We can join (concatenate) two lists together:

```fsharp
let list1 = [1..5]
let list2 = [3..7]
let emptyList = []

let joined = list1 @ list2 // [1;2;3;4;5;3;4;5;6;7]
let joinedEmpty = list1 @ emptyList // [1;2;3;4;5]
let emptyJoined = emptyList @ list1 // [1;2;3;4;5]
```

As lists are immutable, we can re-use them knowing that their values/structure will never change.

We could use the ```List.concat``` function to do the same job as the ```@``` operator:

```fsharp
let joined = List.concat [list1;list2]
```

We can filter a list using a predicate function with the signature `a -> bool` and the ```List.filter``` function:

```fsharp
let myList = [1..9]

let getEvens items =
    items
    |> List.filter (fun x -> x % 2 = 0)

let evens = getEvens myList // [2;4;6;8]
```

We can add up the items in a list using the ```List.sum``` function:

```fsharp
let sum items =
    items |> List.sum

let mySum = sum myList // 45
```

Other aggregation functions are as easy to use but we are not going to look at them here.

Sometimes we want to perform an operation on each item in a list. If we want to return the new list, we use the ```List.map``` function: 

```fsharp
let triple items =
    items
    |> List.map (fun x -> x * 3)

let myTriples = triple [1..5] // [3;6;9;12;15]
```

`List.map` is a higher order function that transforms an `'a list` to a `'b list` using a function that converts `'a ->'b` where `'a` and `'b` could be the same type, like the example shown here which is `int -> int`.

If you have used C#, Select is the closest LINQ equivalent to `List.map`, except it is lazily evaluated. If you need lazy evaluation, you could use `Seq.map` instead.

If we don't want to return a new list, we use the ```List.iter``` function:

```fsharp
let print items =
    items
    |> List.iter (fun x -> (printfn "My value is %i" x))

print myList
```

Let's take a look at a more complicated example using ```List.map``` that changes the structure of the output list. We will use a list of tuples ```(int * decimal)``` which might represent quantity and unit price.

```fsharp
let items = [(1,0.25M);(5,0.25M);(1,2.25M);(1,125M);(7,10.9M)]
```

To calculate the total price of the items, we can use the ```List.map``` function to convert a list of tuples of integer and decimal to a list of decimals and then sum the result:

```fsharp
let sum items =
    items
    |> List.map (fun (q, p) -> decimal q * p)
    |> List.sum
```

Note the explicit conversion of the integer to a decimal. F# is quite strict about types in calculations and doesn't support implicit conversion between types like this. Notice how we can pattern match in the lambda ```(fun (q, p) -> decimal q * p)``` to deconstruct the tuple to gain access to the contained values.

In this particular case, there is an easier way to do the calculation in one step with another of the *List* module functions:

```fsharp
let sum items =
    items
    |> List.sumBy (fun (q, p) -> decimal q * p)
```

## Folding

A very powerful functional concept that we can use to do similar aggregation tasks (and lots more that we won't cover) is the ```List.fold``` function. If you have used C#, Aggregate is the LINQ equivalent:

Our first example is to sum the numbers from one to ten:

```fsharp
[1..10]
|> List.fold (fun acc v -> acc + v) 0
```

`List.fold` is a higher order function that is defined like this:

```fsharp
// folder -> initial value -> input list -> output value
('a -> 'b -> 'a) -> 'a -> 'b list -> 'a
```

The folder function first takes the initial value and stores it in the state which we have called `acc` and then loops through the collection adding each value to the state until there are no items left to process, at which point the state is returned.

This can be simplified as:

```fsharp
[1..10]
|> List.fold (+) 0
```

The product of `[1..10]` using `List.fold` would be:

```fsharp
[1..10]
|> List.fold (fun acc v -> acc * v) 1
```

Notice that we set the initial value to one for multiplication.

This can also be simplified like the sum was:

```fsharp
[1..10]
|> List.fold (*) 1
```

We'll now look at how we would write the total price example we looked at earlier using `List.fold`:

```fsharp
let items = [(1,0.25M);(5,0.25M);(1,2.25M);(1,125M);(7,10.9M)]

let getTotal items =
    items
    |> List.fold (fun acc (q, p) -> acc + decimal q * p) 0M

let total = getTotal items
```

The folder function uses an accumulator and the deconstructed tuple and simply adds the intermediate calculation to the accumulator. The 0M parameter is the initial value of the accumulator. If we were folding using multiplication, the initial value would have been 1M.

An alternative style is to use another of the forward-pipe operators, (||>). This version supports a pair tupled of inputs:

```fsharp
let getTotal items =
    (0M, items) ||> List.fold (fun acc (q, p) -> acc + decimal q * p)

let total = getTotal items
```

There are situations where these additional operators can be useful but in a simple case like this, I prefer the previous version.

Can we use the `List.fold` function to solve FizzBuzz? Of course we can!

```fsharp
let fizzBuzz n =
    [ (3, "Fizz"); (5, "Buzz") ]
    |> List.fold (fun acc (div, msg) -> 
        if n % div = 0 then acc + msg else acc) ""
    |> fun s -> if s = "" then string n else s

[1..105] 
|> List.iter (fizzBuzz >> printfn "%s")
```

We can modify the code to do all of the mapping in the fold function rather than passing the value on to another function:

```fsharp
let fizzBuzz n =
    [ (3, "Fizz"); (5, "Buzz") ]
    |> List.fold (fun acc (div, msg) -> 
        match (if n % div = 0 then msg else "") with
        | "" -> acc
        | s -> if acc = string n then s else acc + s) (string n)
```

Fold is a very nice feature to have but you should try to use simpler, more specific functions like ```List.sumBy``` first.

## Grouping Data and Uniqueness

If we wanted to get the unique numbers from a list, we can do it in many ways. Firstly, we will use the ```List.groupBy``` function to return a tuple for each distinct value:

```fsharp
let myList = [1;2;3;4;5;7;6;5;4;3]

// int list -> (int * int list) list
let gbResult = myList |> List.groupBy (fun x -> x)

// gbResult = [(1,[1]);(2,[2]);(3,[3;3]);(4,[4;4]);(5,[5;5]);(6,[6]);(7,[7])]
```

The `List.groupBy` function returns a list of tuples containing the data you grouped by as the first item and a list of the instances that exist in the original list as the second.

To get the list of unique items from the result list, we can use the ```List.map``` function:

```fsharp
let unique items =
    items
    |> List.groupBy id
    |> List.map (fun (i, _) -> i)
    
let unResult = unique myList // [1;2;3;4;5;6;7]
```

The anonymous function `fun x -> x` can be replaced by the *identity function* `id`.

Using the ```List.groupBy``` function is nice but there is a function called ```List.distinct``` that will do exactly what we want:

```fsharp
let distinct = myList |> List.distinct
```

There is a built-in collection type called *Set* that will do this as well:

```fsharp
let uniqueSet items =
    items
    |> Set.ofList
    
let setResult = uniqueSet myList // [1;2;3;4;5;6;7]
```

Most of the collection types have a way of converting from and to each other.

## Solving a Problem in Many Ways

Thanks to the work of the F# Language Team, there are a wide range of functions available in the collection modules.  This means that there are often multiple ways to solve any given problem. Let's take a simple task like finding the sum of the squares of the odd numbers in a given input list:

```fsharp
// Sum of the squares of the odd numbers
let nums = [1..10]

// Step by step
nums
|> List.filter (fun v -> v % 2 = 1)
|> List.map (fun v -> v * v)
|> List.sum

// Using option and choose
nums
|> List.choose (fun v -> if v % 2 = 1 then Some (v * v) else None) 
|> List.sum

// Do not use reduce for this.
// Firstly, reduce is a partial function, so we need to handle empty list
// More importantly, the first item in the list is not processed by the function,
// So the result will probably be incorrect
match nums with
| [] -> 0
| items -> items |> List.reduce (fun acc v -> acc + if v % 2 = 1 then (v * v) else 0)

// Fold
nums 
|> List.fold (fun acc v -> acc + if v % 2 = 1 then (v * v) else 0) 0

// The recommended version
nums
|> List.sumBy (fun v -> if v % 2 = 1 then (v * v) else 0)
```

Some of the functions are partial functions which means that they do not work for all possible inputs. In the case of `List.reduce`, it will fail on an empty list as it uses the first item as the initial value, so not having one is an issue. Many of the partial functions have a `try` version that returns an option to solve this but `List.reduce` doesn't.

We will now put our new knowledge to good use by looking at a more practical example using something more substantial than a list of integers.

## Working Through a Practical Example

Follow the instructions from the Appendix for creating a solution that we used in the setup of last chapter. The only difference is that you should create a `classlib` project rather than a `console` project.

In this example code, we are going to manage an order that contains an immutable list of items. The functionality we need to add is:

- Add an item
- Increase the quantity of an item
- Remove an item
- Reduce quantity of an item
- Clear all of the items

Create a new file in *MyProject* called *Orders.fs*. You can remove the *Library.fs* file. Add an *OrderTests.fs* file to *MyProjectTests* and add the namespace: 

```fsharp
namespace MyProject.Orders
```

Add the following record types to *Orders.fs*:

```fsharp
type Item = {
    ProductId : int
    Quantity : int
}

type Order = {
    Id : int
    Items : Item list
}
```

Create a module called Domain and put it after the *Order* type definition:

```fsharp
module Domain =
```

We need to create a function to add an item to the order. This function needs to cater for products that exist in the order as well as those that don't. Let's think about how we can do this in pseudocode:

```fsharp
let addItem item order =
    // 1 - Prepend new item to existing order items
    // 2 - Consolidate each product
    // 3 - Sort items in order by productid to make equality simpler
    // 4 - Update order with new list of items
```

The simplest approach will be to perform steps 1 and 4 first:

```fsharp
let addItem item order =
    let items = item::order.Items
    { order with Items = items }
```

Let's create a couple of helpers bindings and some simple asserts to help us test the function in FSI:

```fsharp
let order = { Id = 1; Items = [ { ProductId = 1; Quantity = 1 } ] }
let newItemExistingProduct = { ProductId = 1; Quantity = 1 }
let newItemNewProduct = { ProductId = 2; Quantity = 2 }

addItem newItemNewProduct order = 
    { Id = 1; Items = [ { ProductId = 1; Quantity = 1 };{ ProductId = 2; Quantity = 2 } ] }
addItem newItemExistingProduct order = 
    { Id = 1; Items = [ { ProductId = 1; Quantity = 2 } ] }
```

Run this in FSI and do it for every change that we make. If either of the asserts is false, which in this case the second assert will be, run the code to the left of the equals sign on its own to see what the function is actually returning. You should see something like this:

```plaintext
val it: Order = { Id = 1
                  Items = [{ ProductId = 1
                             Quantity = 1 }; { ProductId = 1
                                               Quantity = 1 }] }
```

To fix this, we need to consolidate the items per product using `List.groupBy`:

```fsharp
// Count how many of each product we have
let addItem item order =
    let items = 
        item :: order.Items
        |> List.groupBy (fun i -> i.ProductId) // (int * Item list) list
    { order with Items = items } // Problem
```

This code won't compile as `List.groupBy` returns a tuple and this is not what `order.Items` is expecting. 

We need to extend the function by mapping the tuple output to the expected *Item* type:

```fsharp
let addItem item order =
    let items = 
        item::order.Items
        |> List.groupBy (fun i -> i.ProductId) // (int * Item list) list
        |> List.map (fun (id, items) -> 
            { ProductId = id; Quantity = items |> List.sumBy (fun i -> i.Quantity) })
    { order with Items = items }
```

Finally, we need to add a sort to enable the built-in list equality mechanism to work:

```fsharp
let addItem item order =
    let items = 
        item::order.Items
        |> List.groupBy (fun i -> i.ProductId) // (int * Item list) list
        |> List.map (fun (id, items) -> 
            { ProductId = id; Quantity = items |> List.sumBy (fun i -> i.Quantity) })
        |> List.sortBy (fun i -> i.ProductId)
    { order with Items = items }
```

Remove the helper bindings we've used for testing as we are going to create real tests in a new *OrderTests.fs* file in the *MyProjectTests* project.

Add the following to the top of the page:

```fsharp
namespace OrderTests

open MyProject.Orders
open MyProject.Orders.Domain
open Xunit
open FsUnit
```

Now we are going to add our first test which verifies that adding items to an empty order works correctly:

```fsharp
module ``Add item to order`` =

    [<Fact>]
    let ``when product does not exist in empty order`` () = 
        let myEmptyOrder = { Id = 1; Items = [] }
        let expected = { Id = 1; Items = [ { ProductId = 1; Quantity = 3 } ] }
        let actual = myEmptyOrder |> addItem { ProductId = 1; Quantity = 3 } 
        actual |> should equal expected
```

Run the test in the Terminal for the test project or the solution using:

```markdown
dotnet test
```

Add the following tests in the same module and run the tests:

```fsharp
    [<Fact>]
    let ``when product does not exist in non empty order`` () = 
        let myOrder = { Id = 1; Items = [ { ProductId = 1; Quantity = 1 } ] }
        let expected = { Id = 1; Items = [ { ProductId = 1; Quantity = 1 };{ ProductId = 2; Quantity = 5 } ] }
        let actual = myOrder |> addItem { ProductId = 2; Quantity = 5 } 
        actual |> should equal expected

    [<Fact>]
    let ``when product exists in non empty order`` () = 
        let myOrder = { Id = 1; Items = [ { ProductId = 1; Quantity = 1 } ] }
        let expected = { Id = 1; Items = [ { ProductId = 1; Quantity = 4 } ] }
        let actual = myOrder |> addItem { ProductId = 1; Quantity = 3 } 
        actual |> should equal expected
```

We have created three tests that cover the main cases when adding an item to an existing order.

We can easily add multiple items to an order by changing the cons operator (::) which adds a single item to the list into the concat operator (@) which joins two lists together:

```fsharp
// Item list -> Order -> Order
let addItems newItems order = 
    let items = 
        newItems @ order.Items 
        |> List.groupBy (fun i -> i.ProductId)
        |> List.map (fun (id, items) -> 
            { ProductId = id; Quantity = items |> List.sumBy (fun i -> i.Quantity) })
        |> List.sortBy (fun i -> i.ProductId)
    { order with Items = items }
```

We should add some tests to *OrderTests.fs* for the `addItems` function:

```fsharp
module ``add multiple items to an order`` =

    [<Fact>]
    let ``when new products added to empty order`` () = 
        let myEmptyOrder = { Id = 1; Items = [] }
        let expected = { Id = 1; Items = [ { ProductId = 1; Quantity = 1 }; { ProductId = 2; Quantity = 5 } ] }
        let actual = myEmptyOrder |> addItems [ { ProductId = 1; Quantity = 1 }; { ProductId = 2; Quantity = 5 } ]
        actual |> should equal expected

    [<Fact>]
    let ``when new products and updated existing to order`` () = 
        let myOrder = { Id = 1; Items = [ { ProductId = 1; Quantity = 1 } ] }
        let expected = { Id = 1; Items = [ { ProductId = 1; Quantity = 2 }; { ProductId = 2; Quantity = 5 } ] }
        let actual = myOrder |> addItems [ { ProductId = 1; Quantity = 1 }; { ProductId = 2; Quantity = 5 } ]
        actual |> should equal expected
```

Run the tests to confirm your new code works.

Let's extract the common functionality in `addItem` and `addItems` into a new function:

```fsharp
// Item list -> Item list
let recalculate items = 
    items
    |> List.groupBy (fun i -> i.ProductId)
    |> List.map (fun (id, items) -> 
        { ProductId = id; Quantity = items |> List.sumBy (fun i -> i.Quantity) })
    |> List.sortBy (fun i -> i.ProductId)

let addItem item order =
    let items = 
        item :: order.Items
        |> recalculate
    { order with Items = items }

let addItems items order =
    let items = 
        items @ order.Items 
        |> recalculate
    { order with Items = items }
```

Run the tests to confirm that the changes have been successful.

Removing a product can be easily achieved by filtering out the unwanted item by the *ProductId*:

```fsharp
let removeProduct productId order =
    let items = 
        order.Items
        |> List.filter (fun x -> x.ProductId <> productId)
        |> List.sortBy (fun i -> i.ProductId)
    { order with Items = items }
```

Again we write some tests to verify our new function works as expected:

```fsharp
module ``Removing a product`` =

    [<Fact>]
    let ``when remove all items of existing productid`` () = 
        let myOrder = { Id = 1; Items = [ { ProductId = 1; Quantity = 1 } ] }
        let expected = { Id = 1; Items = [] }
        let actual = myEmptyOrder |> removeProduct 1 
        actual |> should equal expected

    [<Fact>]
    let ``should do nothing for non existent productid`` () = 
        let myOrder = { Id = 2; Items = [ { ProductId = 1; Quantity = 1 } ] }
        let expected = { Id = 2; Items = [ { ProductId = 1; Quantity = 1 } ] }
        let actual = myOrder |> removeProduct 2 
        actual |> should equal expected
```

Reducing an item quantity is slightly more complex. We reduce the quantity of the item by the specified number, recalculate the items and then filter out any items with a quantity less than or equal to zero:

```fsharp
let reduceItem productId quantity order =
    let items = 
        { ProductId = productId; Quantity = -quantity } :: order.Items
        |> recalculate
        |> List.filter (fun x -> x.Quantity > 0)
    { order with Items = items }
```

Again we write some tests to verify our new function works as expected:

```fsharp
module ``Reduce item quantity`` =

    [<Fact>]
    let ``reduce existing item quantity`` () = 
        let myOrder = { Id = 1; Items = [ { ProductId = 1; Quantity = 5 } ] }
        let expected = { Id = 1; Items = [ { ProductId = 1; Quantity = 2 } ] }
        let actual = myOrder |> reduceItem 1 3
        actual |> should equal expected

    [<Fact>]
    let ``reduce existing item and remove`` () = 
        let myOrder = { Id = 2; Items = [ { ProductId = 1; Quantity = 5 } ] }
        let expected = { Id = 2; Items = [] }
        let actual = myOrder |> reduceItem 1 5
        actual |> should equal expected

    [<Fact>]
    let ``reduce item with no quantity`` () = 
        let myOrder = { Id = 3; Items = [ { ProductId = 1; Quantity = 1 } ] }
        let expected = { Id = 3; Items = [ { ProductId = 1; Quantity = 1 } ] }
        let actual = myOrder |> reduceItem 2 5
        actual |> should equal expected

    [<Fact>]
    let ``reduce item with no quantity for empty order`` () = 
        let myEmptyOrder = { Id = 4; Items = [] }
        let expected = { Id = 4; Items = [] }
        let actual = myEmptyOrder |> reduceItem 2 5
        actual |> should equal expected
```

Clearing all of the items is really simple. We just set Items to be an empty list:

```fsharp
let clearItems order = 
    { order with Items = [] }
```

Write some tests to verify our new function works as expected:

```fsharp
module ``Empty an order of all items`` =

    [<Fact>]
    let ``order with existing item`` () = 
        let myOrder = { Id = 1; Items = [ { ProductId = 1; Quantity = 1 } ] }
        let expected = { Id = 1; Items = [] }
        let actual = myOrder |> clearItems
        actual |> should equal expected

    [<Fact>]
    let ``order with existing items`` () = 
        let myOrder = { Id = 1; Items = [ { ProductId = 1; Quantity = 1 };{ ProductId = 2; Quantity = 5 } ] }
        let expected = { Id = 1; Items = [] }
        let actual = myOrder |> clearItems
        actual |> should equal expected

    [<Fact>]
    let ``empty order is unchanged`` () = 
        let myEmptyOrder = { Id = 2; Items = [] }
        let expected = { Id = 2; Items = [] }
        let actual = myEmptyOrder |> clearItems
        actual |> should equal expected
```

## Summary

In this chapter, we have looked at some of the most useful functions within the *List* module. There are similar functions available in the *Seq* and *Array* modules as we will discover in the coming chapters. We have also seen that it is possible to use immutable data structures to provide important business functionality that is succinct and robust.

We have only scratched the surface of what is possible with collections in F#. For more information on the List module, have a look at the [F# Docs](https://fsharp.github.io/fsharp-core-docs/reference/fsharp-collections-listmodule.html). Thanks to some awesome community assistance, there are code samples for each of the collection module functions.

In the next chapter, we will look at how to handle a stream of data from a CSV source file.
