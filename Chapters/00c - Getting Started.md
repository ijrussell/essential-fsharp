# Getting Started

## Setting up your environment

1. Install F#. The best and simplest way to do this is to install the latest [.NET SDK](https://dotnet.microsoft.com/download) (8.0.x at time of publishing of this edition).
1. Install Visual Studio Code (VS Code).
1. Install the ionide F# extension for VS Code. You may need to reload the environment. The installation page for the extension will tell you.
1. Create a folder for the book and then folders for each book chapter to hold the code you will write.

### F# Interactive (FSI)

If you are coming to F# from C#, the F# Interactive window is similar to the C# Interactive Window. The primary difference is that rather than relying on line-by-line debugging, F# developers will use the FSI instead to validate code. I never debug any code with breakpoints when doing F# development.

FSI appears in the VS Code Terminal window. In addition to the .NET Terminal, you will also have access to an F# Interactive window. You can type directly into FSI but we will send code from our files directly to FSI to compile and run.

> We run code in FSI by selecting the code that we want to run and pressing ALT+ENTER. The code will be copied into FSI, compiled, and run. If you don't change that code, you can reuse it later as FSI keeps your compiled code in memory.

If you type directly into FSI, you will need to end your input with two semi-colons `;;` to tell FSI to execute your code.

### Script Files vs Code Files

F# supports two types of files: Script `.fsx` and Code `.fs`. We will use both types throughout this book. The primary differences in VS Code are that only `.fs` files are compiled into the output `.exe` or `.dll` and `.fsx` files are primarily used to try things out with F# Interactive.

F# code from any type of file can be run in F# Interactive by sending it using `ALT + ENTER`.

There is a command-line version of FSI that can be used to run script files but that is outside the scope of this book.

### Getting to know VS Code and Ionide

Whilst the VS Code and ionide environment is not as fully featured as the full IDEs (Visual Studio and Jet Brains Rider), it is not devoid of features. You don't actually need that many features to easily work on F# codebases, even large ones.

[Compositional-IT](https://www.compositional-it.com/) have released a number of videos on YouTube that give an introduction to [VS Code + F# shortcuts](https://www.youtube.com/playlist?list=PLlzAi3ycg2x27lPUwp3Z-M2m44n681cED).

[Ionide](https://ionide.io/index.html) is much more than just the F# extension for VS Code. Take some time to have a look at what they offer to help your F# experience.

[Fantomas]

## One Last Thing

This book contains lots of code. It will be tempting to copy and paste the code from the book but you may learn more by actually typing the code out as you go. I definitely learn more by doing it this way.

The time for talking has finished. Now it is time to dive into some F# code. In Chapter 1, we will work through a common business use case using F#.
