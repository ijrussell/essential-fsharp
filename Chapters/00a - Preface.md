# Preface

This book is targeted at folks wanting to learn F# and assumes that the reader has no real knowledge of F# or functional programming. Some programming experience, particularly in C# or VB.NET, may be useful but not absolutely necessary.

Although F# has often been portrayed as a Functional Programming language, it isn't, and neither is this book purely about Functional Programming. You are not going to learn about Category Theory, Lambda Calculus, or even Monads. You will learn things about programming in a functional-first style as a by-product of how we solve problems with F#. You will discover how F#'s functional-first approach to programming [empowers everyone to write succinct, robust, and performant code](<https://fsharp.org/>). You will learn a lot of new terminology as you read this book but only enough to understand the features of F# that allow us to solve real-world problems with clearly expressed and concise code. What you will take away most from this book is an understanding of how F# supports us by making it easier for us to write code the F# way than it is to be a Functional Programming purist.

For many historical reasons, Functional Programming has been viewed as difficult to learn and not very practical by most .NET developers. Functional Programming is too often regarded as being too steeped in mathematics and academia to be of any use for writing the everyday line of business applications that most of us work on. This book is designed to help dispel those impressions. So the obvious question to ask is:

> What is F# good for?

Whilst it's great for many things like web programming, cloud programming, Machine Learning, AI, and Data Science, the most accurate answer comes from a long-standing F# Community member [Dave Thomas](<https://twitter.com/7sharp9_>):

>  "F# is good for programming"

F# is a general-purpose language for the .NET platform along with C# and VB.NET. F# has been bundled with Visual Studio, and now the .NET SDK, since 2010. The language has been quite stable since then. Code written in 2010 is not only still valid today but is also likely to be stylistically similar to current standards.

Why would you choose F# over another language? I noticed a question from [cancelerx](<https://twitter.com/ndy40>) on Twitter:

> "I would like to do a lightning talk on F#. My team is primarily PHP and Python devs. What should I showcase?  I want to light a spark."

My response was:

> The expressive type system, composition with the forward pipe operator, pattern matching, collections, and the REPL.

I've been thinking about my answer and whilst I still think that the list is a good one, I think that I should have said that it is how the various features work together that makes F# so special, not the individual features themselves. It feels like a well-thought-out language that doesn't have features just because other languages have them.

F# supports many paradigms, such as Imperative and Object-Oriented Programming but we will concentrate on how it encourages functional-first programming. My view of the choices the designers of F# has made in the language can be summed up quite easily:

> I enjoy functional programming but I like programming in F# even more.

I'm not alone in feeling like this about F#. I hope that this book is a useful step on a similar journey to the one that I've been on.

## Contents

This book covers the core features and practices that a developer needs to know to work effectively on the types of F# Line of Business (LOB) applications we work on at Trustbit. It starts with the basics of types, function composition, pattern matching, and testing, and ends with an example of a simple website and API built using the wonderful [Giraffe](<https://github.com/giraffe-fsharp/Giraffe>) library.

The original source for this book is two series of blog posts that I wrote on the [Trustbit blog](<https://trustbit.tech/blog>) in 2020/21. They have been significantly re-written to clarify and expand the explanations, to ensure that the code works on VS Code using the ionide F# extension, and to take advantage of some new F# features introduced in versions 5 and 6. 

During the course of reading this ebook, you are going to be introduced to a wide range of features that are the essentials of F#:

#### Chapter 1 - A Simple Domain-Modelling Exercise

In the opening chapter, we take a typical business problem and look at how we can use some of the features of F# to solve it.

#### Chapter 2 - Functions

Chapter 2 is all about functions - What they are and how we use composition to build more useful functions.

#### Chapter 3 - Null and Exception Handling

In this chapter, we look at the features F# offers that can eliminate null errors and reduce the need to throw exceptions.

#### Chapter 4 - Organising Code and Testing

In Chapter 4, we will look at how we should structure our code and files plus we get our first look at testing in F#.

#### Chapter 5 - Introduction to Collections

F# has amazing support for handling collections of data. In this chapter, we will get an introduction to some of the features it offers and see how easy it is to compose collection functions into pipelines to transform data from one format to another.

#### Chapter 6 - Reading Data From a File

A short chapter where we see how to load data from a CSV file and parse the data into F# types.

#### Chapter 7 - Active Patterns

Pattern matching is a very powerful tool for the F# developer and active patterns extend that power by allowing us to build custom matchers that can be used to simplify our code and make it more readable.

#### Chapter 8 - Functional Validation

In this chapter, we will use the tools from the previous chapter to add validation to the imported data from Chapter 6. We will also be introduced to one of the more unique features of F#, the computation expression.

#### Chapter 9 - Single-Case Discriminated Unions

In this chapter, we will see how we can reduce our reliance on primitive values in data structures and make our code more domain-centric.

#### Chapter 10 - Object Programming

F# is a functional-first language but it does have excellent support for object programming. This chapter will introduce you to some of the features available.

#### Chapter 11 - Recursion

Recursion is a powerful technique. In this chapter, we will see some of the ways that recursion can help us in our F# journey.

#### Chapter 12 - Computation Expressions

In this chapter, we look more deeply into computation expressions, which are the generic way that F# handles working with effects like Option, Result, and Async.

#### Chapter 13 - Introduction to Web Programming With Giraffe

In this chapter, we will go from nothing to having an API route and a web page using a really nice F# library called Giraffe and its partner, the Giraffe View Engine.

#### Chapter 14 - Creating an API With Giraffe

In this chapter, we will extend the API part of the website.

#### Chapter 15 - Creating Web Pages With Giraffe

A short chapter where we will look at more of the features offered by the Giraffe View Engine.

## F# Software Foundation

You should join the [F# Software Foundation](<https://foundation.fsharp.org/join>); It's free. By joining, you will be able to access the dedicated Slack channel where there are excellent tracks for beginners and beyond.

## About the author

Ian Russell has over 25 years of experience as a software developer in the UK. He has held many technical roles over the years but made the decision many years ago that he could do the most good by remaining 'just a software developer'. Ian works remotely from the UK for Trustbit, a software solutions provider based in Vienna, on a cloud-based GPS aggregator for a logistics company written mostly in F#. Ian's .NET journey started with C# 1.1 in 2003 and he started playing in his own time with F# in 2010. He has been a regular speaker at UK user groups and conferences since 2009.
