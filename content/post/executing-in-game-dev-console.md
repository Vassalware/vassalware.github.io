---
title: "Running C# in an in-game Developer Console with Roslyn"
date: 2018-03-03
tags: ["C#", "game-dev", "roslyn", "scripting"]
draft: false
---

![Example image](/images/console_anim.gif#center-border)

## Introduction

When testing things during game development, it's common to write temporary code that executes on a button press. (e.g., _"When I press F1, call ```SpawnEnemy(Input.MousePosition);```."_) This can be difficult to parameterize, however. If your ```SpawnEnemy``` method takes in more than just a location, such as an enum for your enemy type or an int for its level, you'll need to either change the hard-coded values in the method call (hopefully with [edit-and-continue](https://www.youtube.com/watch?v=mfJEug3Y8ss)) or pass in variables and create a UI to change them at runtime. 

Fast-forward along in development and you may have dozens of these utility functions to manipulate your levels or game objects to test things out, and it can become increasingly inconvenient to maintain a way of calling them all at runtime from different key-presses.

## Developer Console

You've probably seen plenty of developer consoles in games. These are usually designed to mimic desktop command-line interfaces. You invoke commands by writing the name of the command, followed by any parameters it may take, separated by spaces.

```
// Spawn a level 4 Zombie at (200, 200)
> SpawnEnemy Zombie 200 200 4

// or spawn a level 4 Zombie at my mouse
> SpawnEnemyAtMouse Zombie 4
``` 

These are pretty simple to implement: split the input line along the ```' '``` character, and search your ```Dictionary<string, IConsoleCommand>``` using your first element in your array of splits. The rest of those elements are your command's parameters. If your command has some way of defining what types it expects its parameters as, you can even parse the strings into those types before sending them to the command! 

This is a pretty simple but robust method. You create a class that implements ```IConsoleCommand```, write its name and what parameters it expects, and write your operations and logic inside of an ```Execute``` method. But what if we could write operations and logic _directly into the console itself?_

## Expressions

I was initially looking for a way to execute commands that could resolve expressions in their parameters. I wanted to be able to write ```SpawnEnemy Zombie (20 * 10) (20 * (5 + 5)) 4```. I thought _"Okay, I can use [Eto.Parse](https://github.com/picoe/Eto.Parse), write a simple grammar to give me a syntax tree that I can use to resolve the expression. But if I'm writing a grammar, I could change up the syntax. Maybe something like a method call..."_

I slowly realized that my ideal format for writing commands in a runtime console was **still C#**. Better yet, it's even more simple than the aforementioned console implementation. 

## Enter: Roslyn

Roslyn's scripting APIs let us execute arbitrary C# code as a 'script'. This means that, like typical scripting languages, we can execute C# statements without having to place them inside of a method. We can feed C# code from our console into the API and execute it relatively quickly.

To start off, install the scripting API assembly from NuGet:

```C#
Install-Package Microsoft.CodeAnalysis.CSharp.Scripting
```

After that, we can get started executing C# code from a string with just one statement:

```C#
CSharpScript.Create("System.Console.WriteLine(\"Hello!\");").CreateDelegate().Invoke();
```

That's it! Now, let's break down what's happening:

The ```Create()``` call returns a ```Script<object>``` object. We'll be using another overload of the ```Create()``` method later in order to pass some global variables and methods for our scripts to use. We then call the ```CreateDelegate()``` method on the ```Script<object>```, which compiles the Script and returns a delegate that will run it. At this point, it's functionally equivalent to having an ```Action``` delegate with ```System.Console.WriteLine("Hello!");``` inside.

At this point, if you've tried the code yourself, you may have noticed that it's pretty slow (around 1500 ms on my system). This is because .NET assemblies are lazily loaded, so we're loading _quite a few_ assemblies the first time we call this code. When placed at the start of a fresh console application, this code loads **28** assemblies! While subsequent calls are much faster (around 120 ms on my system), we still don't want our game to lock up for 1500 ms the first time we send a command. Even if we compile the script asynchronously (which we should) so that it doesn't block our game thread, we would still be waiting 1500 ms for our command to be invoked, so this is problematic for us.

Thankfully, there's an easy fix. We can just load the assemblies on another thread while our game is still loading. If it takes 1500 ms to load the assemblies we need, and our game already takes 2000 ms to load, then it's highly likely that they'll be finished loading by the time our game finishes loading. Here's the code I use for this:

```
Task.Run(() => {
	CSharpScript.Create("System.Console.WriteLine();").CreateDelegate();
});
```

[Unfinished]