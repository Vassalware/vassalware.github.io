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

<br/>

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

<br/>

## Expressions

I was initially looking for a way to execute commands that could resolve expressions in their parameters. I wanted to be able to write ```SpawnEnemy Zombie (20 * 10) (20 * (5 + 5)) 4```. I thought _"Okay, maybe I can use [Eto.Parse](https://github.com/picoe/Eto.Parse), write a simple grammar to give me a syntax tree that I can use to resolve the expression. But if I'm writing a grammar, I could change up the syntax. Maybe something like a method call..."_

I slowly realized that my ideal format for writing commands in an in-game console was **still C#**. Luckily, this is possible with Roslyn. Better yet, it's even more simple than the aforementioned console implementation. 

<br/>

## Enter: Roslyn

Roslyn's scripting APIs let us execute arbitrary C# code as a 'script'. This means that, like typical scripting languages, we can execute C# statements without having to place them inside of a method. We can feed C# code from our console into the API and execute it relatively quickly.

To start off, install the scripting API assembly from NuGet:

```C#
Install-Package Microsoft.CodeAnalysis.CSharp.Scripting
```

<br/>

After that, we can get started executing C# code from a string with just one statement:

```C#
CSharpScript.Create("System.Console.WriteLine(\"Hello!\");").CreateDelegate().Invoke();
```

That's it! Now, let's break down what's happening:

<br/>

The ```Create()``` call returns a ```Script<object>``` object. We'll be using another overload of the ```Create()``` method later in order to pass some global variables and methods for our scripts to use. We then call the ```CreateDelegate()``` method on the ```Script<object>```, which compiles the Script and returns a delegate that will run it. At this point, it's functionally equivalent to having an ```Action``` delegate with ```System.Console.WriteLine("Hello!");``` inside.

<br/>

At this point, if you've tried the code yourself, you may have noticed that it's pretty slow (around 1500 ms on my system). This is because .NET assemblies are lazily loaded, so we're loading _quite a few_ assemblies the first time we call this code. When placed at the start of a fresh console application, this code loads **28** assemblies! While subsequent calls are much faster (around 120 ms on my system), we still don't want our game to lock up for 1500 ms the first time we send a command. Even if we compile the script asynchronously (which we should) so that it doesn't block our game thread, we would still be waiting 1500 ms for our command to be invoked, so this is problematic for us.

Thankfully, there's an easy fix. We can just load the assemblies on another thread while our game is still loading. If it takes 1500 ms to load the assemblies we need, and our game already takes 2000 ms to load, then it's highly likely that they'll be finished loading by the time our game finishes loading. Here's the code I use for this:

```
Task.Run(() => {
	CSharpScript.Create("System.Console.WriteLine();").CreateDelegate();
});
```

<br/>

## Hooking it up to an in-game Console

Now that we know how to turn a string into an executable C# delegate, we can look at one of the other overloads for ```CSharpScript.Create()```:

```
Create(string code, ScriptOptions options, Type globalsType);
```

We know what the ```code``` parameter is, so let's look at the others.

### Options

```ScriptOptions``` is a class that (obviously) contains the options that the script will be run with. This is where we'll specify which assemblies to reference and what namespaces to import. It has an elegant fluent API that we can use:

```
var scriptOptions = ScriptOptions.Default
	.WithReferences(typeof(RuntimeBinderException).Assembly)
	.WithImports("System");
```

Here we're creating a ScriptOptions object that is a modified version of ```ScriptOptions.Default```. We modify it to add a reference to the assembly that contains the ```RuntimeBinderException``` type (Microsoft.CSharp) so that we can use ```dynamic``` objects in our scripts (more on this later). We also import the "System" namespace so that we can simply write ```Console.WriteLine``` in our in-game console.

### GlobalsType

The ```globalsType``` is a type that we'll be creating. It's members will be globals that we can access from our script easily. Let's make a simple one for now.

```
public class ConsoleGlobals
{
	public int Foo = 100;
}
```

Let's create a script that uses this type, along with the ScriptOptions that we created earlier:

```
var script = CSharpScript.Create(
	code: "Console.WriteLine(Foo);",
	options: scriptOptions,
	globalsType: typeof(ConsoleGlobals))
		.CreateDelegate();
```

To use our globals, we're going to need to pass in an object of that type to the ```Invoke()``` method on the delegate:

```
script.Invoke(new ConsoleGlobals()); // Output: 100
```

And it works! And since we're passing in a reference to an object of our globalType, it can be modified by our script as well. If we change our code to ```Foo = 200;```, we can see this:

```
var globals = new ConsoleGlobals();
Console.WriteLine(globals.Foo); // Output: 100
script.Invoke(globals);
Console.WriteLine(globals.Foo); // Output: 200
```

<br/>

[Unfinished Draft]