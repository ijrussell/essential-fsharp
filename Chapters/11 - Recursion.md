# 11 - Recursion

In this chapter, we are going to look at recursive functions, that is functions that call themselves in a loop. We will start with a naive implementation and then make it more efficient using an accumulator. As a special treat, I will show you a way of writing FizzBuzz and an elegant quicksort algorithm using this technique.

## Setting Up

Create a new folder for the code in this chapter.

All of the code in this chapter can be run using FSI from .fsx files that you create.

## Solving The Problem

We are going to start with a naive implementation of the factorial function (!):

```plaintext
5! = 5 * 4 * 3 * 2 * 1 = 120
```

To create a recursive function, we use the *rec* keyword and we would create a function like this:

```fsharp
// int -> int
let rec fact n = 
    match n with
    | 1 -> 1
    | n -> n * fact (n-1)
```

You'll notice that we have two cases: a base case, in this example where n  equals 1, at which point the recursion ends, and a general case of greater than 1 which is recursive. If we were to write out what happens as we run the recursion, it would look like this:

```text
fact 5 = 5 * fact 4
       = 5 * (4 * fact 3)
       = 5 * (4 * (3 * fact 2))
       = 5 * (4 * (3 * (2 * fact 1)))
       = 5 * (4 * (3 * (2 * 1)))
       = 5 * (4 * (3 * 2))
       = 5 * (4 * 6)
       = 5 * 24
       = 120
```

This is a problem because you can't perform any calculations until you've completed all of the iterations but you also need to store all of the parts as well. This means that the larger n gets, the more memory you need to perform the calculation and it can also lead to stack overflows. We can solve this problem with **Tail Call Optimisation**.

## Tail Call Optimisation

There are a few possible approaches available but we are going to use an accumulator. The accumulator is passed around the recursive function on each iteration:

```fsharp
let fact n =
    let rec loop n acc =
        match n with
        | 1 -> acc
        | _ -> loop (n-1) (acc * n)
    loop n 1
```

We leave the public interface of the function intact but create an enclosed function to do the recursion. We also need to add a line at the end of the function to start the recursion and return the result. You'll notice that we've added an additional parameter `acc` which holds the accumulated value. As we are multiplying, we need to initialise the accumulator to 1. If the accumulator used addition, we would set it to 0 initially.

Again we have two cases to cover; a base case when n = 1 that returns the accumulated value and anything else where the recursion continues with the input value being decremented by 1 and the accumulator being multiplied by the input value. If we write out what happens when we run this function as we did the previous version in pseudocode, we see this:

```plaintext
fact 5 = loop 5 1
       = loop 4 5
       = loop 3 20
       = loop 2 60
       = loop 1 120
       = 120
```

This is much simpler than the previous example, requires very little memory, and is efficient.

Now that we've learnt about tail call optimisation using an accumulator, let's look at a harder example, the Fibonacci Sequence.

## Expanding the Accumulator

The Fibonacci Sequence is a simple list of numbers where each value is the sum of the previous two:

```text
0, 1, 1, 2, 3, 5, 8, 13, 21, ...
```

Let's start with the naive example to calculate the nth item in the sequence:

```fsharp
let rec fib (n:int64) =
    match n with
    | 0L -> 0L
    | 1L -> 1L
    | s -> fib (s-1L) + fib (s-2L)
```

If you want to see just how inefficient this is, try running `fib 50L`. It will take almost a minute on a fast machine! Let's have a go at writing a more efficient version that uses tail call optimisation:

```fsharp
let fib (n:int64) =
    let rec loop n (a,b) =
        match n with
        | 0L -> a
        | 1L -> b
        | n -> loop (n-1L) (b, a+b)
    loop n (0L,1L)
```

Let's write out what happens (ignoring the type annotation):

```plaintext
fib 5L = loop 5L (0L, 1L)
       = loop 4L (1L, 1L+0L)
       = loop 3L (1L, 1L+1L)
       = loop 2L (2L, 1L+2L)
       = loop 1L (3L, 2L+3L)
       = 5L
```

The 5th item in the sequence is `b` which is `2L+3L`. Try running `fib 50L`; It should return almost instantaneously.

Next, we'll now continue on our journey to find as many functional ways of solving FizzBuzz as possible. :)

## Using Recursion to Solve FizzBuzz

We start with a list of mappings that we are going to recurse over:

```fsharp
let mapping = [ (3, "Fizz"); (5, "Buzz") ]
```

Our fizzbuzz function is using tail call optimisation and has an accumulator that will use string concatenation and an initial value of an empty string (`""`).

```fsharp
let fizzBuzz initialMapping n =
    let rec loop mapping acc =
        match mapping with
        | [] -> if acc = "" then string n else acc
        | head::tail -> 
            let value = 
                head |> (fun (div, msg) -> if n % div = 0 then msg else "") 
            loop tail (acc + value)
    loop initialMapping ""
```

The pattern match is asking:

1. If there are no mappings left, then the loop completes and we return either the original input as a string if the accumulator is an empty string specifying no matches or the accumulator.
1. If there are rules left, continue the recursive loop with the tail as the mapping and add the result of the logic to the accumulator.

Finally, we can run the function and print out the results to the terminal:

```fsharp
[ 1 .. 105 ]
|> List.map (fizzBuzz mapping)
|> List.iter (printfn "%s")
```

This is quite a nice extensible approach to the FizzBuzz problem as adding `(7, "Bazz")` is trivial.

```fsharp
let mapping = [ (3, "Fizz"); (5, "Buzz"); (7, "Bazz") ]
```

Having produced a nice solution to the FizzBuzz problem using recursion, can we use the `List.fold` function to solve it? Of course we can!

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

Whilst recursion is the primary mechanism for solving these types of problems in pure functional languages like Haskell, F# has some other approaches that may prove to be simpler to implement and potentially more performant.

Let's have a look at how we can solve another popular algorithm, Quicksort.

## Quicksort using recursion

[Quicksort](<https://www.tutorialspoint.com/data_structures_algorithms/quick_sort_algorithm.htm>) is a nice algorithm to create in F# because of the availability of some very useful collection functions in the List module:

```fsharp
let rec qsort input =
    match input with
    | [] -> []
    | head::tail ->
        let smaller, larger = List.partition (fun n -> head >= n) tail
        List.concat [qsort smaller; [head]; qsort larger]

[5;9;5;2;7;9;1;1;3;5] |> qsort |> printfn "%A"
```

The `List.partition` function splits a list into two based on a predicate function, in this case, items smaller than or equal to the head value and those larger than it. `List.concat` converts a deeply nested sequence of lists into a single list.

## Recursion with Hierarchical Data

Recursion can be used to create and deconstruct hierarchical data structures. We are going to tackle Episode 2.1 of the Trustbit Transport Tycoon challenge:

https://github.com/trustbit/exercises/blob/master/transport-tycoon_21.md

We are given a map and some CSV data detailing the distances between connected locations:

```csv
A,B,km
Cogburg,Copperhold,1047
Leverstorm,Irondale,673
Cogburg,Steamdrift,1269
Copperhold,Irondale,345
Copperhold,Leverstorm,569
Leverstorm,Gizbourne,866
Rustport,Cogburg,1421
Rustport,Steamdrift,1947
Rustport,Gizbourne,1220
Irondale,Gizbourne,526
Cogburg,Irondale,1034
Rustport,Irondale,1302
```

We will assume that the return journeys are the same distance.

Create a new file called *shortest-distance.fsx*.

Create a new folder called *resources*, add a file called *data.csv*, and add the data above to it.

Open the *shortest-distance.fsx* file.

Add the following declaration to the top of the file:

```fsharp
open System.IO
```

We can define a simple generic hierarchical discriminated union like this:

```fsharp
type Tree<'T> =
    | Branch of 'T * Tree<'T> seq
    | Leaf of 'T
```

Add the *Tree* type to the file.

We are going to tackle this problem in three stages:

- Load the data.
- Generate possible routes from start to finish.
- Find the shortest from the possible routes.

Let's get started!

### Load the Data

We need to define a record type to hold the data that we load from the CSV file:

```fsharp
type Connection = { Start:string; Finish:string; Distance:int }
```

We next need to write a function that takes a file path and returns the loaded data:

```fsharp
// string -> Map<string, Connection list>
let loadData path =
    path
    |> File.ReadLines
    |> Seq.skip 1
    |> fun rows -> [
        for row in rows do
            match row.Split(",") with
            | [|start;finish;distance|] -> 
                { Start = start; Finish = finish; Distance = int distance }
                { Start = finish; Finish = start; Distance = int distance }
            | _ -> failwith "Row is badly formed"
    ]
    |> List.groupBy (fun cn -> cn.Start)
    |> Map.ofList
```

Instead of returning a *Connection list*, we are returning *Map<string,Connection list>*. A Map is a read-only dictionary that better suits our needs since the question we will ask of this data is which routes are there from a location?

We need to call this new function with a start and finish location, so we will add that code:

```fsharp
let run start finish = 
    Path.Combine(__SOURCE_DIRECTORY__, "resources", "data.csv") 
    |> loadData
    |> printfn "%A"

let result = run "Cogburg" "Leverstorm"
```

Run the code in FSI. At the moment, you will see the data from the *Map*.

### Find Possible Routes

We are going to define a record type that defines a waypoint on our route. It will include data on where we are, how we got there, and how far we have travelled to get here. Not only does this save time when it comes to calculating the shortest route but it will be very helpful in the next stage. Let's define our *Waypoint* type:

```fsharp
type Waypoint = { Location:string; Route:string list; TotalDistance:int }
```

We need to create a function that determines where we can go to next that we haven't already been to: 

```fsharp
// Connection list -> Waypoint -> Waypoint list
let getUnvisited connections current =
    connections
    |> List.filter (fun cn -> current.Route |> List.exists (fun loc -> loc = cn.Finish) |> not)
    |> List.map (fun cn -> { 
        Location = cn.Finish
        Route = cn.Start :: current.Route
        TotalDistance = cn.Distance + current.TotalDistance })
```

We pass in the current *Waypoint* and a list of possible *Connections* with it as the starting point. We can filter out those possible next locations that we have already been to on that route. Finally, we create a list of *Waypoints* with updated accumulated data.

Now we are ready to generate our Tree structure that defines all of the possible routes between the start and the finish:

```fsharp
// string -> string -> Map<string, Connection list> -> Tree<Waypoint>
let findPossibleRoutes start finish (routeMap:Map<string, Connection list>) =
    let rec loop current =
        let nextRoutes = getUnvisited routeMap[current.Location] current
        if nextRoutes |> List.isEmpty |> not && current.Location <> finish then
            Branch (current, seq { for next in nextRoutes do loop next })
        else 
            Leaf current
    loop { Location = start; Route = []; TotalDistance = 0 }
```

We have two cases where a branch would not continue: when there are no locations that we can go to that we haven't already been and if the location is the finish location.

We need to add the `findPossibleRoutes` function to our `run` function:

```fsharp
let run start finish = 
    Path.Combine(__SOURCE_DIRECTORY__, "resources", "data.csv") 
    |> loadData
    |> findPossibleRoutes start finish 
    |> printfn "%A"

let result = run "Cogburg" "Leverstorm"
```

Execute the code in FSI and you will get a tree structure built with Waypoints.

Now we need to convert our tree structure to a list of Waypoints, so that we can find the shortest route:

```fsharp
// Tree<Waypoint> -> List<Waypoint>
let rec treeToList tree =
    match tree with 
    | Leaf x -> [x]
    | Branch (_, xs) -> List.collect treeToList (xs |> Seq.toList)
```

We are only interested in the leaves because they should be the only place where the finish location exists in the output.

We need to add this function to the end of the `findPossibleRoutes` function. We also need to filter out any Waypoints that do not point at the finish.

```fsharp
// string -> string -> Map<string, Connection list> -> Waypoint list
let findPossibleRoutes start finish (routeMap:Map<string, Connection list>) =
    let rec loop current =
        let nextRoutes = getUnvisited routeMap[current.Location] current
        if nextRoutes |> List.isEmpty |> not && current.Location <> finish then
            Branch (current, seq { for next in nextRoutes do loop next })
        else 
            Leaf current
    loop { Location = start; Route = []; TotalDistance = 0 }
    |> treeToList
    |> List.filter (fun wp -> wp.Location = finish)
```

Execute the code in FSI and you will see that we are getting closer to completing this task.

### Select Shortest Route

Our final task is to determine the shortest route from the output:

```fsharp
let selectShortestRoute routes =
    routes 
    |> List.minBy (fun wp -> wp.TotalDistance)
    |> fun wp -> wp.Location :: wp.Route |> List.rev, wp.TotalDistance
```

We should add the `selectShortestRoute` function to the `run` function:

```fsharp
let run start finish = 
    Path.Combine(__SOURCE_DIRECTORY__, "resources", "data.csv") 
    |> loadData
    |> findPossibleRoutes start finish 
    |> selectShortestRoute
    |> printfn "%A"

let result = run "Cogburg" "Leverstorm"
```

Execute the code in FSI and you will see something like this:

```fsharp
> let result = run "Cogburg" "Leverstorm";;
(["Cogburg"; "Copperhold"; "Leverstorm"], 1616)
```

This was quite a long task but notice that we broke it down into smaller sections to make it easier to resolve.

### Finished Code

```fsharp
open System.IO

type Tree<'T> =
    | Branch of 'T * Tree<'T> seq
    | Leaf of 'T

type Waypoint = { Location:string; Route:string list; TotalDistance:int }

type Connection = { Start:string; Finish:string; Distance:int }

// string -> Map<string, Connection list>
let loadData path =
    path
    |> File.ReadLines
    |> Seq.skip 1
    |> fun rows -> [
        for row in rows do
            match row.Split(",") with
            | [|start;finish;distance|] -> 
                { Start = start; Finish = finish; Distance = int distance }
                { Start = finish; Finish = start; Distance = int distance }
            | _ -> failwith "Row is badly formed"
    ]
    |> List.groupBy (fun cn -> cn.Start)
    |> Map.ofList

// Map<string, Connection list> -> Waypoint -> Waypoint list
let getUnvisited connections current =
    connections
    |> List.filter (fun cn -> current.Route |> List.exists (fun loc -> loc = cn.Finish) |> not)
    |> List.map (fun cn -> { 
        Location = cn.Finish
        Route = cn.Start :: current.Route
        TotalDistance = cn.Distance + current.TotalDistance })

// Tree<Waypoint> -> List<Waypoint>
let rec treeToList tree =
    match tree with 
    | Leaf x -> [x]
    | Branch (_, xs) -> List.collect treeToList (xs |> Seq.toList)

// string -> string -> Map<string, Connection list> -> Waypoint list
let findPossibleRoutes start finish (routeMap:Map<string, Connection list>) =
    let rec loop current =
        let nextRoutes = getUnvisited routeMap[current.Location] current
        if nextRoutes |> List.isEmpty |> not && current.Location <> finish then
            Branch (current, seq { for next in nextRoutes do loop next })
        else 
            Leaf current
    loop { Location = start; Route = []; TotalDistance = 0 }
    |> treeToList
    |> List.filter (fun wp -> wp.Location = finish)

// Waypoint list -> (string list * int)
let selectShortestRoute routes =
    routes 
    |> List.minBy (fun wp -> wp.TotalDistance)
    |> fun wp -> wp.Location :: wp.Route |> List.rev, wp.TotalDistance

// string -> string -> unit
let run start finish = 
    Path.Combine(__SOURCE_DIRECTORY__, "resources", "data.csv") 
    |> loadData
    |> findPossibleRoutes start finish 
    |> selectShortestRoute
    |> printfn "%A"

let result = run "Cogburg" "Leverstorm"
```

## Other Uses Of Recursion

Recursion is great for handling hierarchical data like the file system and XML, converting between flat data and hierarchies, and for event loops or loops where there is no defined finish. 

As always, Scott Wlaschin's excellent [site](<https://fsharpforfunandprofit.com/posts/recursive-types-and-folds-3b/#series-toc>) has many posts on the topic.

## Summary

In this chapter, we have looked at recursion in F#. In particular, we have looked at the basics and how to use accumulators with tail call optimisation to make recursion more efficient. Finally, we used recursion to help us solve a more complex problem involving hierarchical data.

In the next chapter, we will look at a feature we met in Chapter 8 on functional validation: Computation Expressions.
