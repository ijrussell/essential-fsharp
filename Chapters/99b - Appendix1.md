# Appendix 1

## Creating Solutions and Projects in VS Code

We are going to create a new .NET Solution containing an F# console project and an XUnit test
project using the dotnet CLI.

Open VS Code in a new folder.

Open a new Terminal window. The shortcut for this is CTRL+SHIFT+’.

Create a new file in the folder. It doesn’t matter what it is called, so use ‘setup.txt’.

Copy the following script into the file:

```plaintext
dotnet new sln -o MySolution
cd MySolution
mkdir src
dotnet new console -lang F# -o src/MyProject
dotnet sln add src/MyProject/MyProject.fsproj
mkdir tests
dotnet new xunit -lang F# -o tests/MyProjectTests
dotnet sln add tests/MyProjectTests/MyProjectTests.fsproj
cd tests/MyProjectTests
dotnet add reference ../../src/MyProject/MyProject.fsproj
dotnet add package XUnit
dotnet add package FsUnit
dotnet add package FsUnit.XUnit
dotnet build
dotnet test
```

This script will create a new solution called MySolution, two folders called src and tests, a console app called MyProject in the src folder, and a test project called MyProjectTests in the tests folder. Change the names to suit. In VS Code, CTRL+F2 will allow you to edit all of the instances of the selected word at the same time.

If you want C# projects, omit the '-lang F#' from both project lines and change the file extensions to '.csproj' from '.fsproj'.

Select all of the script text.

To run the script in the Terminal, you can do either of the following:

- Choose the Terminal menu item and then select 'Run Selected Text'.
- Press CTRL+SHIFT+P to open the Command Palette and then type 'TRSTAT'. Select the '**T**erminal: **R**un **S**elected **T**ext in **A**ctive **T**erminal' item.

The script will now execute in the Terminal.

You can delete the file with the script in it if you wish as we no longer need it.
