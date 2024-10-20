# 7 - Active Patterns

In this chapter, we will be extending our knowledge of pattern matching by looking at how we can write our own custom matchers with `Active Patterns`. There are a number of different `Active Patterns` types available and we will look at most of them in this chapter.

## Setting Up

Create a folder for the code from this chapter.

All of the code in this chapter can be written in an F# script file (.fsx) and run using F# Interactive.

## Partial Active Patterns

Partial active patterns return an option as output because only a subset of the possible input values will produce a positive match.

One area where Partial Active Patterns are really helpful is for validation and parsing. This is an example of parsing a string to a DateTime using a function rather than an active pattern:

```fsharp
open System

// string -> DateTime option
let parse (input:string) =
    match DateTime.TryParse(input) with
    | true, value -> Some value
    | false, _ -> None

let isDate = parse "2019-12-20" // Some 2019-12-20 00:00:00
let isNotDate = parse "Hello" // None
```

It works but we can't re-use the function body logic directly in a pattern match, except in the guard clause. Let's create a partial active pattern to handle the DateTime parsing for us:

```fsharp
open System

// string -> DateTime option
let (|ValidDate|_|) (input:string) = 
    match DateTime.TryParse(input) with
    | true, value -> Some value
    | false, _ -> None
```

Anytime that you see the banana clips ```(|...|)```, you are looking at an active pattern and if you see ``` (|...|_|)```, then you are looking at a partial active pattern. Remember, the partial part of the name comes from the use of the wildcard in the pattern definition, implying that the active pattern only returns success for a subset of the possible input values. 

Notice that we are returning the parsed value in the success case. We will be able to access that when we use the active pattern.

The logic in this active pattern could be written to use an *if* expression instead of a *match* expression:

```fsharp
open System

// string -> DateTime option
let (|ValidDate|_|) (input:string) = 
    let success, value = DateTime.TryParse(input)
    if success then Some value else None
```

Either version is valid.

Now we can plug it into our parse function:

```fsharp
// string -> unit
let parse input =
    match input with
    | ValidDate dt -> printfn "%A" dt
    | _ -> printfn $"'%s{input}' is not a valid date"

parse "2019-12-20" // 2019-12-20 00:00:00
parse "Hello" // 'Hello' is not a valid date
```

We can access the parsed date for the success case because we returned it from the partial active pattern. 

There's more code but it is much more readable and we have functionality that we can re-use in other pattern matches. In this case, the partial active pattern returned the parsed value as part of the match. If you didn't care about the value being returned, you can return ```Some ()``` instead of ```Some value``` as shown in the following example:

```fsharp
open System

// string -> unit option
let (|IsValidDate|_|) (input:string) = 
    let success, _ = DateTime.TryParse input
    if success then Some () else None
    
// string -> bool
let isValidDate input =
    match input with
    | IsValidDate -> true
    | _ -> false
```

Partial active patterns are used a lot in activities like validation, as we will discover in Chapter 8.

## Parameterized Partial Active Patterns

Parameterized partial active patterns differ from basic partial active patterns by supplying additional input items.

We are going to investigate how an old interview favourite, FizzBuzz, can be implemented using a parameterized partial active pattern. Let's start with the canonical solution:

```fsharp
let calculate i =
    if i % 3 = 0 && i % 5 = 0 then "FizzBuzz"
    elif i % 3 = 0 then "Fizz"
    elif i % 5 = 0 then "Buzz"
    else i |> string

[1..15] |> List.map calculate
```

It works but can we do better with Pattern Matching? How about this?

```fsharp
let calculate i =
    match (i % 3, i % 5) with
    | (0, 0) -> "FizzBuzz"
    | (0, _) -> "Fizz"
    | (_, 0) -> "Buzz"
    | _ -> i |> string
```

Or this?

```fsharp
let calculate i =
    match (i % 3 = 0, i % 5 = 0) with
    | (true, true) -> "FizzBuzz"
    | (true, _) -> "Fizz"
    | (_, true) -> "Buzz"
    | _ -> i |> string
```

Neither of these is any more readable than the original. How about if we could use an active pattern to do something like this?

```fsharp
let calculate i =
    match i with
    | IsDivisibleBy 3 & IsDivisibleBy 5 -> "FizzBuzz"
    | IsDivisibleBy 3 -> "Fizz"
    | IsDivisibleBy 5 -> "Buzz"
    | _ -> i |> string
```

Note the single `&` to apply both parts of the pattern match in the calculate function. There is also a single `|` for the or case.

We can use a parameterized partial active pattern to do this:

```fsharp
// int -> int -> unit option
let (|IsDivisibleBy|_|) divisor n  = 
    if n % divisor = 0 then Some () else None
```

The parameterized name comes from the fact that we can supply additional parameters. The value being tested must always be the last parameter. In this case, we include the divisor parameter as well as the number we are checking.

This is much nicer but what happens if we need to add ```(7, Bazz)``` or even more options into the mix? Look at the code now:

```fsharp
let calculate i =
    match i with
    | IsDivisibleBy 3 & IsDivisibleBy 5 & IsDivisibleBy 7 -> "FizzBuzzBazz"
    | IsDivisibleBy 3 & IsDivisibleBy 5 -> "FizzBuzz"
    | IsDivisibleBy 3 & IsDivisibleBy 7 -> "FizzBazz"
    | IsDivisibleBy 5 & IsDivisibleBy 7 -> "BuzzBazz"
    | IsDivisibleBy 3 -> "Fizz"
    | IsDivisibleBy 5 -> "Buzz"
    | IsDivisibleBy 7 -> "Bazz"
    | _ -> i |> string
```

Maybe not such a good idea? How confident would you be that you had covered all of the permutations? 

An alternative suggested by Isaac Abraham would be to use a list as the input parameter of the active pattern:

```fsharp
let (|IsDivisibleBy|_|) divisors n  = 
    if divisors |> List.forall (fun div -> n % div = 0)
    then Some ()
    else None

let calculate i =
    match i with
    | IsDivisibleBy [3;5;7] -> "FizzBuzzBazz"
    | IsDivisibleBy [3;5] -> "FizzBuzz"
    | IsDivisibleBy [3;7] -> "FizzBazz"
    | IsDivisibleBy [5;7] -> "BuzzBazz"
    | IsDivisibleBy [3] -> "Fizz"
    | IsDivisibleBy [5] -> "Buzz"
    | IsDivisibleBy [7] -> "Bazz"
    | _ -> i |> string
```

In many ways, this is much nicer but it still involves a lot of typing and is potentially prone to errors.

Maybe a different approach would yield a better result?

```fsharp
let calculate n =
    [(3, "Fizz"); (5, "Buzz"); (7, "Bazz")]
    |> List.map (fun (divisor, result) -> if n % divisor = 0 then result else "")
    |> List.reduce (+) // (+) is a shortcut for (fun acc v -> acc + v)
    |> fun input -> if input = "" then string n else input

[1..15] |> List.map calculate    
```

The `List.reduce` function will concatenate all of the strings generated by the `List.map` function. The last line is to cater for numbers that are not divisible by any of the numbers in the initial list.

It is less readable than the original but it's much less prone to errors covering all of the possible permutations. You could even pass the mappings in as a parameter:

```fsharp
let calculate mapping n =
    mapping
    |> List.map (fun (divisor, result) -> if n % divisor = 0 then result else "")
    |> List.reduce (+) 
    |> fun input -> if input = "" then string n else input

[1..15] |> List.map (calculate [(3, "Fizz"); (5, "Buzz")])     
```

This is the type of function that F# developers write all the time.

Back to active patterns! Let's have a look at another favourite interview question: Leap Years. The basic F# function looks like this;

```fsharp
let isLeapYear year =
    year % 400 = 0 || (year % 4 = 0 && year % 100 <> 0)

[2000;2001;2020] |> List.map isLeapYear = [true;false;true]
```

But can parameterized partial active patterns help make it more readable? Let's try:

```fsharp
let (|IsDivisibleBy|_|) divisor n  =
    if n % divisor = 0 then Some () else None

let (|NotDivisibleBy|_|) divisor n  =
    if n % divisor <> 0 then Some () else None

let isLeapYear year =
    match year with
    | IsDivisibleBy 400 -> true
    | IsDivisibleBy 4 & NotDivisibleBy 100 -> true
    | _ -> false
```

There are special pattern matching operators for *and* (`&`) and *or* (`|`) for logic but not for *not*, so we have to write the negative case as well as the *IsDivisibleBy* active pattern. This doesn't apply to a `when` guard clause which uses the standard F# logic operators.

You may be wondering why we wrote two parameterized partial active patterns when it seems like a nice candidate for a multi-case active pattern:

```fsharp
let (|IsDivisibleBy|NotDivisibleBy|) divisor n =
    if n % divisor = 0 then IsDivisibleBy else NotDivisibleBy 
```

Sadly, whilst this does compile, it will not work in situ due to the way that the F# compiler processes multi-case active patterns. 

There's more code than our original version but you could easily argue that it is more readable. We could also do a similar thing to the original with helper functions rather than Active Patterns;

```fsharp
let isDivisibleBy divisor year =
    year % divisor = 0

let notDivisibleBy divisor year =
    not (year |> isDivisibleBy divisor)

let isLeapYear year =
    year |> isDivisibleBy 400 || (year |> isDivisibleBy 4 && year |> notDivisibleBy 100)
```

You could also use a match expression with guard clauses:

```fsharp
let isLeapYear input =
    match input with
    | year when year |> isDivisibleBy 400 -> true
    | year when year |> isDivisibleBy 4 && year |> notDivisibleBy 100 -> true
    | _ -> false
```

This would allow us to remove the *notDivisibleBy* function and pipe the *isDivisible* result to the *not* function:

```fsharp
let isLeapYear input =
    match input with
    | year when year |> isDivisibleBy 400 -> true
    | year when year |> isDivisibleBy 4 && year |> isDivisibleBy 100 |> not -> true
    | _ -> false
```

All of these approaches are valid; it's down to preference which one you choose.

## Multi-Case Active Patterns

Multi-Case Active Patterns are different from Partial Active Patterns in that they always return one of the possible values rather than an option type. The maximum number of choices supported is currently seven although that may increase in a future version of F#.

A simple example of a multi-case active pattern is to determine the colour of a playing card; it can either be red or black. Firstly, we can model a playing card:

```fsharp
type Rank = Ace|Two|Three|Four|Five|Six|Seven|Eight|Nine|Ten|Jack|Queen|King
type Suit = Hearts|Clubs|Diamonds|Spades
type Card = Rank * Suit
```

Rank and suit are discriminated unions with no data attached to any case.

The active pattern needs to take a *Card* as input, determine its suit, and return either Red or Black as output:

```fsharp
let (|Red|Black|) (card:Card) =
    match card with
    | (_, Diamonds) | (_, Hearts) -> Red
    | (_, Clubs) | (_, Spades) -> Black
```

This is not a partial active pattern, so there is no wildcard.

Only cases that are used as output should be included in the active pattern.

We can now use the newly created active pattern in a function that will describe the colour of a chosen card:

```fsharp
let describeColour card =
    match card with 
    | Red -> "red"
    | Black -> "black"
    |> printfn "The card is %s"

describeColour (Two, Hearts) // The card is red
```

## Single-Case Active Patterns

The last type of active pattern that we will look at is the single-case. The purpose of the single-case is to allow you to decompose an input in different ways. We will use a simple string for this.

In this example, we are going to create some rules for creating a password. The rules will be:

- At least 8 characters in length
- Must contain at least one number

Our first action is to create the single-case active patterns for our rules. We will put the rule logic in the number check but not the length check to see what difference that makes to the code that uses the active patterns:

```fsharp
open System

// string -> int
let (|CharacterCount|) (input:string) = 
    input.Length

// string -> bool
let (|ContainsANumber|) (input:string) =
    input
    |> Seq.filter Char.IsDigit
    |> Seq.length > 0
```

A string in F# is also a sequence of characters.

We can use our new active patterns to help perform the password logic in a similar way to how we defined it in the requirements:

```fsharp
let (|IsValidPassword|) input = 
    match input with
    | CharacterCount len when len < 8 -> (false, "Password must be at least 8 characters.")
    | ContainsANumber false -> (false, "Password must contain at least 1 digit.")
    | _ -> (true, "")
```

If we were doing this in production code, we would probably create a discriminated union as the return type rather than use a tuple. This would enable us to remove the need to supply an empty string in the output for false.

Finally, we can make use of the *IsValidPassword* active pattern in our code:

```fsharp
let setPassword input =
    match input with
    | IsValidPassword (true, _) as pwd -> Ok pwd
    | IsValidPassword (false, failureReason) -> Error $"Password not set: %s{failureReason}"

let badPassword = setPassword "password"
let goodPassword = setPassword "passw0rd"
```

You may not use the single-case active pattern very much but when you do, it does make the code much more readable.

Now that we have seen the most commonly used active patterns, we are going to use them to help us solve a little problem.

## Using Active Patterns in a Practical Example

I play, rather unsuccessfully, an online Football (the sport where participants kick a spherical ball with their feet rather than throw an egg-shaped object with their hands) Score Predictor.

The rules are simple;

- 300 points for predicting the correct score (e.g. 2-3 vs 2-3)
- 100 points for predicting the correct result (e.g. 2-3 vs 0-2)
- 15 points per home goal & 20 points per away goal using the lower of the predicted and actual scores

We have some sample predictions, actual scores, and expected calculated points that we can use to validate the code we write;

```fsharp
(0, 0) (0, 0) = 400 // 300 + 100 + (0 * 15) + (0 * 20)
(3, 2) (3, 2) = 485 // 300 + 100 + (3 * 15) + (2 * 20)
(5, 1) (4, 3) = 180 // 0 + 100 + (4 * 15) + (1 * 20)
(2, 1) (0, 7) = 20 // 0 + 0 + (0 * 15) + (1 * 20)
(2, 2) (3, 3) = 170 // 0 + 100 + (2 * 15) + (2 * 20)
```

Firstly we define a simple tuple type to represent a score:

```fsharp
type Score = int * int
```

To determine if we have a correct score, we need to check that the prediction and the actual score are the same. F# uses structural equality to compare tuples, so this is trivial to evaluate with a partial active pattern:

```fsharp
// Score * Score -> option<unit>
let (|CorrectScore|_|) (expected:Score, actual:Score) =
    if expected = actual then Some () else None
```

We don't care about the actual scores, just that they are the same, so we can return unit.

We also need to determine what the result of a Score is. This can only be one of three choices: Draw (Tie), Home win or Away win. We can easily represent this with a multi-case active pattern:

```fsharp
let (|Draw|HomeWin|AwayWin|) (score:Score) =
    match score with
    | (h, a) when h = a -> Draw
    | (h, a) when h > a -> HomeWin
    | _ -> AwayWin
```

Note that this active pattern returns one of the pattern choices rather than an option type. We can now create a new partial active pattern for determining if we have predicted the correct result:

```fsharp
let (|CorrectResult|_|) (expected:Score, actual:Score) =
    match (expected, actual) with
    | (Draw, Draw) -> Some ()
    | (HomeWin, HomeWin) -> Some ()
    | (AwayWin, AwayWin) -> Some ()
    | _ -> None
```

We have to provide the input data as a single structure, so we use a tuple of Scores.

Without the multi-case active pattern for the result of a score, we would have to write something like this:

```fsharp
let (|CorrectResult|_|) (expected:Score, actual:Score) =
    match (expected, actual) with
    | ((h, a), (h', a')) when h = a && h' = a' -> Some ()
    | ((h, a), (h', a')) when h > a && h' > a' -> Some ()
    | ((h, a), (h', a')) when h < a && h' < a' -> Some ()
    | _ -> None
```

I prefer the version using the multi-case active pattern.

Now we need to create a function to work out the points for the goals scored:

```fsharp
let goalsScore (expected:Score) (actual:Score) =
    let (h, a) = expected
    let (h', a') = actual
    let home = [ h; h' ] |> List.min
    let away = [ a; a' ] |> List.min
    (home * 15) + (away * 20)    
```

There are a couple of helper functions for tuples called `fst` and `snd` that we can use to get the first and second parts of the tuple respectively. This can simplify the *goalScore* function:

```fsharp
let goalsScore (expected:Score) (actual:Score) =
    let home = [ fst expected; fst actual ] |> List.min
    let away = [ snd expected; snd actual ] |> List.min
    (home * 15) + (away * 20)    
```

We now have all of the parts to create our function to calculate the total points for each game:

```fsharp
let calculatePoints (expected:Score) (actual:Score) =
    let pointsForCorrectScore = 
        match (expected, actual) with
        | CorrectScore -> 300
        | _ -> 0
    let pointsForCorrectResult =
        match (expected, actual) with
        | CorrectResult -> 100
        | _ -> 0
    let pointsForGoals = goalsScore expected actual
    pointsForCorrectScore + pointsForCorrectResult + pointsForGoals
```

Note how the tuple in the match expression matches the input required by the parameterized active patterns.

There is a bit of duplication/similarity that we will resolve later but firstly we should use our test data to validate our code using FSI:

```fsharp
let assertnoScoreDrawCorrect = 
    calculatePoints (0, 0) (0, 0) = 400
let assertHomeWinExactMatch = 
    calculatePoints (3, 2) (3, 2) = 485
let assertHomeWin = 
    calculatePoints (5, 1) (4, 3) = 180
let assertIncorrect = 
    calculatePoints (2, 1) (0, 7) = 20
let assertDraw = 
    calculatePoints (2, 2) (3, 3) = 170
```

We can simplify the *calculatePoints* function by combining the pattern matching for *CorrectScore* and *CorrectResult* into a new function:

```fsharp
let resultScore (expected:Score) (actual:Score) =
    match (expected, actual) with
    | CorrectScore -> 400
    | CorrectResult -> 100
    | _ -> 0
```

Note that we had to return 400 from *CorrectScore* in this function as we are no longer able to add the *CorrectResult* points later. This allows us to simplify the *calculatePoints* function:

```fsharp
let calculatePoints (expected:Score) (actual:Score) =
    let pointsForResult = resultScore expected actual
    let pointsForGoals = goalScore expected actual
    pointsForResult + pointsForGoals
```

As the *resultScore* and *goalScore* functions have the same signature, we can use a higher order function to remove the duplication:

```fsharp
let calculatePoints (expected:Score) (actual:Score) =
    [ resultScore; goalScore ]
    |> List.sumBy (fun f -> f expected actual)
```

Yes, we can put functions into a List. `List.sumBy` is equivalent to `List.map` followed by `List.sum`.

There are other types of active patterns that we haven't met in this chapter; I recommend that you investigate the F# Documentation to make yourself familiar with them.

## Summary

Active patterns are very useful and powerful features but they are not always the best approach. When used well, they improve readability.

In the next chapter we will be taking some of the features we've met in this chapter to add validation to the code we used in chapter 6.
