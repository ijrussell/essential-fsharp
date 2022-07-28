# 14 - Creating an API with Giraffe

In this chapter, we'll be creating a simple API with Giraffe. We are going to add new functionality to the project we created in the last chapter.

## Getting Started

You will need a tool to run HTTP calls (GET, POST, PUT, and DELETE). I use [Postman](<https://www.postman.com/>) but any tool including those available in VS Code will work.

Open the code from the last chapter in VS Code.

## Our Task

We are going to create a simple API that we can view, create, update and delete Todo items.

## Sample Data

Rather than work against a real data store, we are going to create a simple store with a dictionary and use that in our handlers.

Create a new file above *Program.fs* called *TodoStore.fs* and add the following code to it:

```fsharp
module GiraffeExample.TodoStore

open System
open System.Collections.Concurrent

type TodoId = Guid

type NewTodo = {
    Description: string
}

type Todo = {
    Id: TodoId
    Description: string
    Created: DateTime
    IsCompleted: bool
}

type TodoStore() =
    let data = ConcurrentDictionary<TodoId, Todo>()

    let get id =
        let (success, value) = data.TryGetValue(id)
        if success then Some value else None
    
    member _.Create(todo) = data.TryAdd(todo.Id, todo)
    member _.Update(todo) = data.TryUpdate(todo.Id, todo, data[todo.Id])
    member _.Delete(id) = data.TryRemove id
    member _.Get(id) = get id
    member _.GetAll() = data.Values |> Seq.toArray
```

*TodoStore* is a simple class type that wraps a concurrent dictionary that we can use to test our API out without needing to connect to a database. This means that it will not persist between runs. You could pass the initial state through the TodoStore constructor if you need data.

Add a reference to the module import declarations in *Program.fs*:

```fsharp
open GiraffeExample.TodoStore
```

To be able to use the *TodoStore*, we need to make a change to the `configureServices` function in *Program.fs*:

```fsharp
let configureServices (services : IServiceCollection) =
    services
        .AddRouting()
        .AddGiraffe()
        .AddSingleton<TodoStore>(TodoStore())
    |> ignore
```

If you're thinking that this looks like dependency injection, you would be correct; we are using the one provided by ASP.NET Core. We add the TodoStore as a singleton as we only need one instance to exist. 

## Routes

We saw in the last chapter that Giraffe uses individual route handlers, so we need to think about how to add our new routes. The routes we need to add are:

```plaintext
GET     /api/todo       // Get a list of todos
GET     /api/todo/id    // Get one todo
POST    /api/todo       // Create a todo
PUT     /api/todo/id    // Update a todo
DELETE  /api/todo/id    // Delete a todo
```

Let's create a new handler with the correct HTTP verbs just above the `endpoints` value:

```fsharp
let apiTodoRoutes =
    [
        GET [
            routef "/%O" viewTodoHandler
            route "" viewTodosHandler
        ]
        POST [ 
            route "" createTodoHandler
        ]
        PUT [
            routef "/%O" updateTodoHandler
        ]
        DELETE [
            routef "/%O" deleteTodoHandler
        ]
    ]
```

We will create the missing handlers after we have plugged the new handler into our `endpoints` value as a subroute:

```fsharp
let endpoints =
    [
        GET [
            route "/" (htmlView indexView)
        ]
        subRoute "/api/todo" apiTodoRoutes
        subRoute "/api" apiRoutes
    ]
```

There are lots of ways of arranging the routes. It's up to you to find an efficient approach. I like this style with all of the route strings in one place. This makes them easy to change.

Next, we have to implement the new handlers.

## Handlers

Rather than create the new handlers in *Program.fs*, we are going to create them in a new file. We will also move the `apiTodoRoutes` handler there as well.

Create a new file between *Program.fs* and *TodoStore.fs* called *Todos.fs*. Copy the following code into the new file:

```fsharp
module GiraffeExample.Todos

open System
open System.Collections.Generic
open Microsoft.AspNetCore.Http
open Giraffe
open Giraffe.EndpointRouting
open GiraffeExample.TodoStore
```

Move (Cut & Paste) the `apiTodoRoutes` handler function to the new *Todos.fs* file.

You will need to fix the error in the `endpoints` binding by adding an open declaration to *Project.fs*:

```fsharp
open GiraffeExample
```

Add the subroute:

```fsharp
subRoute "api/todo" Todos.apiTodoRoutes
```

Now we can concentrate on adding the handlers for our new routes to a new Handlers module above `apiTodoRoutes` in *Todos.fs*. We'll start with the two GET requests:

```fsharp
module Handlers =

    let viewTodosHandler =
        fun (next : HttpFunc) (ctx : HttpContext) ->
            let store = ctx.GetService<TodoStore>()
            store.GetAll()
            |> ctx.WriteJsonAsync

    let viewTodoHandler (id:Guid) =
        fun (next : HttpFunc) (ctx : HttpContext) ->
            task {
                let store = ctx.GetService<TodoStore>()
                return!
                    (match store.Get(id) with
                    | Some todo -> json todo
                    | None -> RequestErrors.NOT_FOUND "Not Found") next ctx
            }
```

We are using the *HttpContext* instance (`ctx`) to gain access to the *TodoStore* instance we set up earlier using service location, a simple form of dependency injection. 

Let's add the handlers for POST and PUT in the Handlers module:

```fsharp
let createTodoHandler =
    fun (next : HttpFunc) (ctx : HttpContext) ->
        task {
            let! newTodo = ctx.BindJsonAsync<NewTodo>()
            let store = ctx.GetService<TodoStore>()
            let created = 
                { 
                    Id = Guid.NewGuid()
                    Description = newTodo.Description
                    Created = DateTime.UtcNow
                    IsCompleted = false 
                }
                |> store.Create
            return! json created next ctx
        }

let updateTodoHandler (id:Guid) =
    fun (next : HttpFunc) (ctx : HttpContext) ->
        task {
            let! todo = ctx.BindJsonAsync<Todo>()
            let store = ctx.GetService<TodoStore>()
            return!
                (match store.Update(todo) with
                | true -> json true
                | false -> RequestErrors.GONE "Gone") next ctx
        }
```

The most interesting thing here is that we use a built-in Giraffe function to gain strongly-typed access to the request body passed into the handler via the *HttpContext* (`ctx`).

Finally, we handle the Delete route:

```fsharp
let deleteTodoHandler (id:Guid) =
    fun (next : HttpFunc) (ctx : HttpContext) ->
        task {
            let store = ctx.GetService<TodoStore>()
            return!
                (match store.Get(id) with
                | Some existing -> 
                    let deleted = store.Delete(KeyValuePair<TodoId, Todo>(id, existing))
                    json deleted
                | None -> RequestErrors.GONE "Gone") next ctx
        }
```

Finally, we need to fix the errors in the `apiTodoRoutes` handler by adding the module name to the handler calls:

```fsharp
let apiTodoRoutes =
    [
        GET [
            routef "/%O" Handlers.viewTodoHandler
            route "" Handlers.viewTodosHandler
        ]
        POST [ 
            route "" Handlers.createTodoHandler
        ]
        PUT [
            routef "/%O" Handlers.updateTodoHandler
        ]
        DELETE [
            routef "/%O" Handlers.deleteTodoHandler
        ]
    ]
```

One last thing that we probably should do is to remove the `Handler` suffix from the handler names since they are in a module called *Handlers*. You could also argue that the *Todo* in the names is redundant as well. I'll leave these as tasks for the reader!

We should now be able to use the new *Todo* API.

## Using the API

Run the app and use a tool like Postman to work with the API.

To get a list of all Todos, we call `GET /api/todo`. This should return an empty JSON array because we don't have any items in our store currently.

We create a *Todo* by calling `POST /api/todo` with a JSON request body like this:

```json
{ "Description": "Finish blog post" }
```

You will receive a response of true or false. If you now call the list again, you will receive a JSON response containing the newly created item:

```json
[
    {
        "key": "ff5a1d35-4573-463d-b9fa-6402202ab411",
        "value": {
            "id": "ff5a1d35-4573-463d-b9fa-6402202ab411",
            "description": "Finish blog post",
            "created": "2021-03-12T13:47:39.3564455Z",
            "isCompleted": false
        }
    }
]
```

I'll leave the other routes for you to investigate. 

## Summary

We have only scratched the surface of what is possible with Giraffe for creating APIs such as content negotiation and model validation. I highly recommend reading the [Giraffe documentation](<https://github.com/giraffe-fsharp/Giraffe/blob/master/DOCUMENTATION.md>) to get a fuller picture of what is possible.

In the final chapter of the book, we will dive deeper into HTML views with the Giraffe View Engine.