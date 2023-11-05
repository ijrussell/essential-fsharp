# 15 - Creating Web Pages with Giraffe

In the last chapter, we created a simple API for managing a Todo list. In this post, we are going to expand our journey into HTML views with the Giraffe View Engine.

If you haven't already done so, read the previous two chapters on Giraffe.

## Getting Started

We are going to continue to make changes to the project we updated in the last chapter.

We are going to create a simple HTML view of a *Todo list* and populate it with dummy data from the server.

Rather than rely on my HTML/CSS skills, we are going to start with a pre-built sample: The *ToDo list* example from [w3schools](<https://www.w3schools.com/howto/howto_js_todolist.asp>).

## Configuration

Add a new folder to the project called *wwwroot*, two sub-folders called *css* and *js*, and add a file to each folder: *main.js* and *main.css*. Copy the CSS and JavaScript from Appendix 2 into the relevant files. The *wwwroot* folder will be automatically wired to the project when you perform the next step. 

We have to tell the app to serve static files if requested. We need to add the `UseStaticFiles()` extension method to the `configureApp` function in *Program.fs*:

```fsharp
let configureApp (appBuilder : IApplicationBuilder) =
    appBuilder
        .UseRouting()
        .UseStaticFiles()
        .UseGiraffe(endpoints)
        .UseGiraffe(notFoundHandler)
```

Have a look [here](<https://docs.microsoft.com/en-us/aspnet/core/fundamentals/static-files?view=aspnetcore-6.0>) if you need to configure a different folder than *wwwroot*.

## Adding a Master Page

We saw in Chapter 13 how we can create HTML using the Giraffe View Engine DSL. 

We are going to create a master page and pass the title and content into it.

Create a new file called *Shared.fs* above the other created files.

Let's start with a basic HTML page:

```html
<html>
    <head>
        <title>My Title</title>
        <link rel="stylesheet" href="css/main.css" />
    </head>
    <body />
</html>
```

If we convert the HTML to Giraffe View Engine format, we would get:

```fsharp
// string -> XmlNode list -> XmlNode
let masterPage msg content =
    html [] [
        head [] [
            title [] [ str msg ]
            link [ _rel "stylesheet"; _href "css/main.css" ]
        ]
        body [] content
    ]
```

Most tags have two lists, one for styling and one for content. Some tags, like *input*, only take the style list.

Add the following code to *Shared.fs*:

```fsharp
module GiraffeExample.Shared

open Giraffe.ViewEngine

let masterPage msg content =
    html [] [
        head [] [
            title [] [ str msg ]
            link [ _rel "stylesheet"; _href "css/main.css" ]
        ]
        body [] content
    ]
```

To update the existing view with our new master page, we need to add an open declaration to *Program.fs*:

```fsharp
open GiraffeExample
```

We then need to change `indexView` to use the new master page:

```fsharp
let indexView =
    [
        h1 [] [ str "I |> F#" ]
        p [ _class "some-css-class"; _id "someId" ] [
            str "Hello World from the Giraffe View Engine"
        ]
    ]
    |> Shared.masterPage "Giraffe View Engine Example"
```

This approach is very different to most view engines which rely on search and replace but are primarily still HTML. The primary advantage of the Giraffe View Engine approach is that you get full type safety when creating and generating views and can use the full power of the F# language.

Run the website to prove that you haven't broken anything.

## Creating the Todo List View

The HTML below from the w3schools source has to be converted into Giraffe View Engine format:

```html
<div id="myDIV" class="header">
  <h2>My To Do List</h2>
  <input type="text" id="myInput" placeholder="Title...">
  <span onclick="newElement()" class="addBtn">Add</span>
</div>

<ul id="myUL">
  <li>Hit the gym</li>
  <li class="checked">Pay bills</li>
  <li>Meet George</li>
  <li>Buy eggs</li>
  <li>Read a book</li>
  <li>Organise office</li>
</ul>
```

Create a new module in *Todos.fs* called `Views` and add a new function to generate our `todoView` binding:

```fsharp
module Views =

    open Giraffe.ViewEngine

    let todoView =
        [
            div [ _id "myDIV"; _class "header" ] [
                h2 [] [ str "My To Do List" ]
                input [ _type "text"; _id "myInput"; _placeholder "Title..." ]
                span [ _class "addBtn"; _onclick "newElement()" ] [ str "Add" ]
            ]
            ul [ _id "myUL" ] [
                li [] [ str "Hit the gym" ]
                li [ _class "checked" ] [ str "Pay bills" ]
                li [] [ str "Meet George" ]
                li [] [ str "Buy eggs" ]
                li [] [ str "Read a book" ]
                li [ _class "checked" ] [ str "Organise office" ]
            ]
            script [ _src "js/main.js"; _type "text/javascript" ] []
        ]
        |> Shared.masterPage "My ToDo App" 
```

We only need to tell the endpoints router about our new view. Change the root '/' route in webApp to:

```fsharp
let endpoints =
    [
        GET [
            route "/" (htmlView Todos.Views.todoView)
        ]
        subRoute "/api/todo" apiTodoRoutes
        subRoute "/api" apiRoutes
    ]
```

If you now run the app using dotnet run in the Terminal. Follow the HTTPS link in the Terminal and you will see the *Todo* app running in the browser.

## Loading Data on Startup

Rather than hard code the list of *Todos* into the view, we can load it in on the fly by creating a list of items. Create the following above the *Views* module in *Todos.fs*:

```fsharp
module Data =

    let todoList = [
        { Id = Guid.NewGuid(); Description = "Hit the gym"; Created = DateTime.UtcNow; IsCompleted = false }
        { Id = Guid.NewGuid(); Description = "Pay bills"; Created = DateTime.UtcNow; IsCompleted = true }
        { Id = Guid.NewGuid(); Description = "Meet George"; Created = DateTime.UtcNow; IsCompleted = false }
        { Id = Guid.NewGuid(); Description = "Buy eggs"; Created = DateTime.UtcNow; IsCompleted = false }
        { Id = Guid.NewGuid(); Description = "Read a book"; Created = DateTime.UtcNow; IsCompleted = true }
        { Id = Guid.NewGuid(); Description = "Read Essential F#"; Created = DateTime.UtcNow; IsCompleted = false }
    ]
```

There's a lot of duplicate code, so let's create a helper function to simplify things:

```fsharp
module Data =

    let private create description isCompleted =
        { 
            Id = Guid.NewGuid()
            Description = description
            Created = DateTime.UtcNow
            IsCompleted = isCompleted 
        }

    let todoList = 
        [ 
            ("Hit the gym", false)
            ("Pay bills", true)
            ("Meet George", false)
            ("Buy eggs", false)
            ("Read a book", true)
            ("Read Essential F#", false)
        ]
        |> List.map (fun (todo, isCompleted) -> create todo isCompleted)
```

We need to create a partial view, which is simple helper function, to style each item in the *Todo list*. You should write this function above the `todoView` code in the *Views* module:

```fsharp
// Todo -> XmlNode
let private showListItem (todo:Todo) =
    let style = if todo.IsCompleted then [ _class "checked" ] else []
    li style [ str todo.Description ]
```

Now we can use the `showListItem` partial view function in the `todoView` and we can pass the *Todo* items in as an input parameter:

```fsharp
let todoView items =
    [
        div [ _id "myDIV"; _class "header" ] [
            h2 [] [ str "My ToDo List" ]
            input [ _type "text"; _id "myInput"; _placeholder "Title..." ]
            span [ _class "addBtn"; _onclick "newElement()" ] [ str "Add" ]
        ]
        ul [ _id "myUL" ] [
            for todo in items do 
                showListItem todo
        ]
        script [ _src "js/main.js"; _type "text/javascript" ] []
    ]
    |> masterPage "My ToDo App"
```

We are generating the list of *Todos* in a list comprehension.

We need to change the root route as we are now passing the list of *Todos* into the `todoView` function:

```fsharp
let endpoints =
    [
        GET [
            route "/" (htmlView (Todos.Views.todoView Todos.Data.todoList))
        ]
        subRoute "/api/todo" apiTodoRoutes
        subRoute "/api" apiRoutes
    ]
```

Run the app to see the *Todo* items displayed on the HTML page.

## Summary

We have only scratched the surface of what is possible with Giraffe and the Giraffe View Engine for creating web pages and APIs. I highly recommend reading the [Giraffe View Engine documentation](<https://github.com/giraffe-fsharp/Giraffe.ViewEngine>) to find out about some of the other features available.

If you like the Giraffe View Engine but don't like writing JavaScript, you should investigate the [SAFE Stack](<https://safe-stack.github.io/>) where **everything** is written in F#. The SAFE Stack is also great for creating Single Page Apps with React frontends, all in F#.

That concludes our journey into F# and Giraffe. I hope that I have given you enough of a taste of what F# can offer to encourage you to want to take it further.
