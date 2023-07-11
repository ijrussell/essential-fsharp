# 13 - Introduction to Web Programming with Giraffe

In this chapter, we will start investigating web programming in F#.

We are going to use [Giraffe](<https://github.com/giraffe-fsharp/Giraffe>). Giraffe is "an F# micro web framework for building rich web applications" and was created by [Dustin Moris Gorski](<https://twitter.com/dustinmoris>). It is a thin functional wrapper around ASP.NET Core. Both APIs and web pages are supported by Giraffe and we will cover both in the last three chapters of this book.

Giraffe comes with excellent [documentation](<https://github.com/giraffe-fsharp/Giraffe/blob/master/DOCUMENTATION.md>). I highly recommend that you spend some time looking through it to see what features Giraffe offers.

If you want something more opinionated or want to use F# everywhere including to create JavaScript, have a look at [Saturn](<https://saturnframework.org/>) and the [Safe Stack](<https://safe-stack.github.io/>).

## Getting Started

Create a new folder called *GiraffeExample* and open it in VS Code.

Using the Terminal in VS Code, type in the following command to create an empty ASP.NET Core web app project:

```plaintext
dotnet new web -lang F#
```

Open *Program.fs*. If you've done any ASP.NET Core development before, it will look very familiar, even though it is in F#:

```fsharp
open System
open Microsoft.AspNetCore.Builder
open Microsoft.Extensions.Hosting

[<EntryPoint>]
let main args =
    let builder = WebApplication.CreateBuilder(args)
    let app = builder.Build()

    app.MapGet("/", Func<string>(fun () -> "Hello World!")) |> ignore

    app.Run()

    0 // Exit code
```

Run the code using the Terminal:

```markdown
dotnet run
```

You will see something like this:

```markdown
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: https://localhost:7274
info: Microsoft.Hosting.Lifetime[14]
      Now listening on: http://localhost:5264
info: Microsoft.Hosting.Lifetime[0]
      Application started. Press Ctrl+C to shut down.
info: Microsoft.Hosting.Lifetime[0]
      Hosting environment: Development
info: Microsoft.Hosting.Lifetime[0]
      Content root path: D:\Code\GiraffeExample\
```

Use `CTRL+CLICK` on one of the URLs. Your browser should display:

```markdown
Hello World!
```

To shut the running website down, press ```CTRL+C``` in the Terminal window where the code is running. You should be back to the command prompt.

## Using Giraffe

Firstly, add the following NuGet packages to your project from the Terminal:

```plaintext
dotnet add package Giraffe
dotnet add package Giraffe.ViewEngine
```

Replace the code in *Program.fs* with the following:

```fsharp
open System
open Microsoft.AspNetCore.Builder
open Microsoft.Extensions.Hosting
open Microsoft.Extensions.DependencyInjection
open Microsoft.AspNetCore.Http
open Giraffe
open Giraffe.EndpointRouting
open Giraffe.ViewEngine

let endpoints =
    [
        GET [
            route "/" (text "Hello World from Giraffe")
        ]
    ]

let notFoundHandler =
    "Not Found"
    |> text
    |> RequestErrors.notFound

let configureApp (appBuilder : IApplicationBuilder) =
    appBuilder
        .UseRouting()
        .UseGiraffe(endpoints)
        .UseGiraffe(notFoundHandler)

let configureServices (services : IServiceCollection) =
    services
        .AddRouting()
        .AddGiraffe()
    |> ignore

[<EntryPoint>]
let main args =
    let builder = WebApplication.CreateBuilder(args)
    configureServices builder.Services

    let app = builder.Build()

    if app.Environment.IsDevelopment() then
        app.UseDeveloperExceptionPage() |> ignore
    
    configureApp app
    app.Run()
    0
```

There is a lot more code but anyone who has configured ASP.NET Core applications will know that there is a lot that you will probably need to configure that isn't in the minimal template. The version for Giraffe looks similar to the pre-minimal template code with a few exceptions like the new endpoint routing and the `notFoundHandler`.

### Mutability

One of the requirements of configuration for ASP.NET Core is the ability to set properties on non-F# objects. Up to this point in the book, we have not used mutability directly. We need to define a binding as `mutable` to be able to modify it:

```fsharp
let mutable x = 1

x = 2 // false
```

We can't use the equality operator (`=`) to update the value; instead, it will compare x to 2, which is false.

To update the value of `x`, we use the assignment operator (`<-`):

```fsharp
x <- 2

x = 2 // true
```

You need to use the same operator to set properties of .NET objects. You do not need to mark .NET objects as mutable as they already are.

If you accidentally use the equality operator, the compiler will give you a warning that you need to ignore the boolean result. If you run your code, it will not do what you are expecting but will not report an error.

### Endpoints

Endpoint routing was actually introduced in Giraffe 5.x and is designed to integrate with the ASP.NET Core endpoint routing APIs. You can read more about Giraffe endpoint routing at https://github.com/giraffe-fsharp/Giraffe/blob/master/DOCUMENTATION.md#endpoint-routing.

The endpoint routing allows us to make use of existing HttpHandlers and new custom ones that we are going to write.

### 404 - Not Found Handler

We have added a handler for when a route cannot be found, which is represented by an HTTP response status code of 404. If you run the web app and add a route like `/api` to your request, you will get a response of 404 and a payload of `Not Found`.

We should run the new code to verify that it works. In the Terminal, type the following to run the project:

```plaintext
dotnet run
```

Use `CTRL+CLICK` as before and you will see the following in your browser:

```markdown
Hello World from Giraffe
```

The message is generated by the `text` HttpHandler that Giraffe supplies. In addition, it will return a 200 status code.

Shut the running website down by pressing ```CTRL+C``` in the Terminal window where the code is running. You should be back to the command prompt.

## Creating an API Route

We are going to add a new route '/api' to our list of endpoints and respond with some JSON using another built-in Giraffe HttpHandler, *json*:

```fsharp
let endpoints =
    [
        GET [
            route "/" (text "Hello World from Giraffe")
            route "/api" (json {| Response = "Hello world!!" |})
        ]
    ]
```

The payload for the JSON response on the `/api` route is an anonymous record, designated by the `{|..|}` structure. You define the structure and data of the anonymous record in place rather than defining a type and creating an instance. Anonymous records are great for lightweight serialization structures, particularly for generating JSON for an HTTP endpoint like the one we have created. If you want to learn more about them, have a look at https://www.compositional-it.com/news-blog/5-tips-for-working-with-f-anonymous-records/.

Now run the code and add `/api` to the Url in the browser. You should see a JSON response:

```markdown
{"response":"Hello world!!"}
```

## Creating a Custom HttpHandler

There are lots of built-in HttpHandlers but it is simple to create your own because they are functions with the signature `HttpFunc -> HttpContext -> HttpFuncResult`. 

Let's add a new route in our endpoints for the API that takes a string value in the request Url.  The request will be handled by a custom handler that we haven't written yet:

```fsharp
let endpoints =
    [
        GET [
            route "/" (text "Hello World from Giraffe")
            route "/api" (json {| Response = "Hello world!!" |})
            routef "/api/%s" sayHelloNameHandler
        ]
    ]
```

**[routef]**

Note that the Url value is automatically bound to the handler function if it has a matching input parameter of the same type in the handler, so the `sayHelloNameHandler` needs to take a string input parameter. Create the custom handler above the `endpoints` binding:

```fsharp
let sayHelloNameHandler (name:string) : HttpHandler =
    fun (next:HttpFunc) (ctx:HttpContext) ->
        task {
            let msg = $"Hello {name}, how are you?"
            return! json {| Response = msg |} next ctx
        }
```

This function could be rewritten as the following but it is convention to use the style above:

```fsharp
let sayHelloNameHandler (name:string) (next:HttpFunc) (ctx:HttpContext) : HttpHandler =
    task {
        let msg = $"Hello {name}, how are you?"
        return! json {| Response = msg |} next ctx
    }
```

The `task {...}` is a Computation Expression of type `System.Threading.Tasks.Task<'T>` and is the equivalent to *async/await* in C#. In addition, F# has a different asynchronous mechanism called *async* but we won't use that as Giraffe has chosen to work directly with *Task* instead.

You will see a lot more of the *task* Computation Expression in the next chapter as we expand the API side of our website.

Now run the code and then try the `api/{yourname}` route in the browser. If you replace the placeholder `{yourname}` with your name, you should see a JSON response asking how you are.

Giraffe has some built-in extensions for *HttpContext* which can simplify some handlers:

```fsharp
let sayHelloNameHandler (name:string) : HttpHandler =
    fun (next:HttpFunc) (ctx:HttpContext) ->
        {| Response = $"Hello {name}, how are you?" |}
        |> ctx.WriteJsonAsync
```

The `WriteJsonAsync` function serializes the anonymous record to JSON and writes the output to the body of the HTTP response. In addition, the function ensures that the HTTP *Content-Type* header to `application/json` and then correctly sets the *Content-Length* header. The JSON serializer can be configured in the ASP.NET Core startup code by registering a custom class of type `Json.ISerializer`. For details on this, consult the Giraffe documentation.

Giraffe is excellent for creating APIs but it is equally suited to server-side rendered web pages using the Giraffe View Engine.

## Creating a View

The Giraffe View Engine is a Domain-Specific Language (DSL) for generating HTML using Giraffe. This is what a simple HTML page looks like using the DSL:

```fsharp
let indexView =
    html [] [
        head [] [
            title [] [ str "Giraffe Example" ]
        ]
        body [] [
            h1 [] [ str "I |> F#" ]
            p [ _class "some-css-class"; _id "someId" ] [
                str "Hello World from the Giraffe View Engine"
            ]
        ]
    ]
```

Each element of html has the following structure:

```fsharp
// element [attributes] [sub-elements/data]
html [] [] 
```

Each element has two supporting lists, the first is for attributes and the second for sub-elements or data. You may wonder why we need a DSL to generate HTML, but it helps prevent badly formed structures. All of the features you need like master pages, partial views and model binding are included. We will have a deeper look at views in the final chapter of this book.

Add the `indexView` code above the `webApp` handler and replace the root (`"/"`) route with the following using the Giraffe `htmlView` handler function:

```fsharp
route "/" (htmlView indexView)
```

Run the web app and you will see the text from the indexView in your browser.

## Adding Subroutes

We have some duplicate code in our routes already, so let's try to simplify them. This is where we are currently:

```fsharp
let endpoints =
    [
        GET [
            route "/" (htmlView indexView)
            route "/api" (json {| Response = "Hello world!!" |})
            routef "/api/%s" sayHelloNameHandler
        ]
    ]

```

We can extract routes to another HttpHandler like we do with the '/api' route here:

```fsharp
// Endpoint list
let apiRoutes = 
    [
        route "" (json {| Response = "Hello world!!" |})
        routef "/%s" sayHelloNameHandler
    ]

// Endpoint list
let endpoints =
    [
        GET [
            route "/" (htmlView indexView)
            subRoute "/api" apiRoutes
        ]
    ]
```

Note the change from 'route' to 'subRoute' if we extract the '/api' route to its own handler.

Run the app and try the routes out in your browser or by using a simple HTTP client like Postman or Insomnia.

It is likely that the `apiRoutes` endpoint handler will need to deal with other HTTP verbs like POST, etc. We can modify the code to reflect this possibility:

```fsharp
let apiRoutes = 
    [
        GET [
            route "" (json {| Response = "Hello world!!" |})
            routef "/%s" sayHelloNameHandler
        ]
    ]

let endpoints =
    [
        GET [
            route "/" (htmlView indexView)
        ]
        subRoute "/api" apiRoutes
    ]
```

We've moved the *subRoute* out of the *GET* section in `endpoints` and added a *GET* section to the routes in `apiRoutes`. In the next chapter, we will use multiple HTTP verbs in a route handler.

Finally, run the app again and try the routes out in your browser or by using a simple HTTP client.

## Reviewing the Code

The finished code for this chapter is:

```fsharp
open System
open Microsoft.AspNetCore.Builder
open Microsoft.Extensions.Hosting
open Microsoft.Extensions.DependencyInjection
open Microsoft.AspNetCore.Http
open Giraffe
open Giraffe.EndpointRouting
open Giraffe.ViewEngine

let indexView =
    html [] [
        head [] [
            title [] [ str "Giraffe Example" ]
        ]
        body [] [
            h1 [] [ str "HTML Generated by F#" ]
            p [ _class "some-css-class"; _id "someId" ] [
                str "Hello World from the Giraffe View Engine"
            ]
        ]
    ]
    
let sayHelloNameHandler (name:string) : HttpHandler =
    fun (next:HttpFunc) (ctx:HttpContext) ->
        {| Response = $"Hello {name}, how are you?" |}
        |> ctx.WriteJsonAsync

let apiRoutes = 
    [
        GET [
            route "" (json {| Response = "Hello world!!" |})
            routef "/%s" sayHelloNameHandler
        ]
    ]

let endpoints =
    [
        GET [
            route "/" (htmlView indexView)
        ]
        subRoute "/api" apiRoutes
    ]

let notFoundHandler =
    "Not Found"
    |> text
    |> RequestErrors.notFound

let configureApp (appBuilder : IApplicationBuilder) =
    appBuilder
        .UseRouting()
        .UseGiraffe(endpoints)
        .UseGiraffe(notFoundHandler)

let configureServices (services : IServiceCollection) =
    services
        .AddRouting()
        .AddGiraffe()
    |> ignore

[<EntryPoint>]
let main args =
    let builder = WebApplication.CreateBuilder(args)
    configureServices builder.Services

    let app = builder.Build()

    if app.Environment.IsDevelopment() then
        app.UseDeveloperExceptionPage() |> ignore
    
    configureApp app
    app.Run()
    0
```

## Summary

We have only scratched the surface of what is possible with Giraffe. In this chapter we had an introduction to using Giraffe to build an API and how to use the Giraffe View Engine to create HTML pages. **We also learnt about the importance of HttpHandlers in Giraffe routing and we're introduced to a number of the built-in ones Giraffe offers and we have seen that we can create our own handlers.**

In the next chapter, we will expand the API side of the application.