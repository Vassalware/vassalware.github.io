---
title: "Running C# in an in-game Developer Console with Roslyn"
description: "Supercharge your in-game developer console with C# scripting! Using the Roslyn Scripting API, we'll turn a string of C# code into an invocable delegate in just one line. Then, we'll go further and compile our C# commands asynchronously, then invoke them on the update thread."
date: 2018-03-04
image: images/console_anim.gif
tags: ["C#", "game-dev", "roslyn", "scripting"]
draft: false
---



![Example image](/images/console_anim.gif#center-border)

## Introduction

When testing things during game development, it's common to write temporary code that executes on a button press. (e.g., _"When I press F1, call ```SpawnEnemy(Input.MousePosition);```."_) This can be difficult to parameterize, however. If your ```SpawnEnemy``` method takes in more than just a location, such as an enum for your enemy type or an int for its level, you'll need to either change the hard-coded values in the method call (hopefully with [**edit-and-continue**](https://www.youtube.com/watch?v=mfJEug3Y8ss)) or pass in variables and create a UI to change them at runtime. 

Fast-forward along in development and you may have dozens of these utility functions to manipulate your levels or game objects to test things out, and it can become increasingly inconvenient to maintain a way of calling them all at runtime from different key-presses.

<br/>

## Developer Console

You've probably seen plenty of developer consoles in games, usually designed to mimic standard command-line interfaces. You execute commands by writing the name of the command, followed by any parameters it may take, separated by spaces.

```
// Spawn a level 4 Zombie at (200, 200)
> SpawnEnemy Zombie 200 200 4

// or spawn a level 4 Zombie at the mouse position
> SpawnEnemyAtMouse Zombie 4
``` 

These are pretty simple to implement: split the input line along the ```' '``` character, and search a ```Dictionary<string, IConsoleCommand>``` using your first element in your array of splits. The rest of those elements are your command's parameters. If your command has some way of defining what types it expects its parameters as, you can even parse the strings into those types before sending them to the command! 

###### _There's a good implementation of this type of console [**here**](https://github.com/ellisonch/monogameconsole/blob/master/MonoGameConsole/IConsoleCommand.cs), but it seems to leave the parameter string parsing up to the individual commands._

It's a simple implementation, but a robust one. It's easy to add a command: you create a class that implements an ```IConsoleCommand``` interface that defines a Name, its parameters, and an ```Execute``` method, and all your operations and logic go inside of that method. Then you just add an instance to the dictionary.

But what if we could write operations and logic _directly into the console itself?_

<br/>

## Expressions

I was initially looking for a way to execute commands that could resolve expressions in their parameters. I wanted to be able to write ```SpawnEnemy Zombie (20 * 10) (20 * (5 + 5)) 4```. I thought _"Okay, maybe I can use [**Eto.Parse**](https://github.com/picoe/Eto.Parse), write a simple grammar to give me a syntax tree that I can use to resolve the expression. But if I'm writing a grammar, I could change up the syntax. Maybe something like a method call..."_

I slowly realized that my ideal format for writing commands in an in-game console was **still C#**. Luckily, this is possible with Roslyn. Better yet, it's even more simple than the aforementioned console implementation. 

<br/>

## Roslyn

Roslyn's scripting APIs let us execute arbitrary C# code as a 'script'. This means that, like typical scripting languages, we can execute statements without having to place them inside of a method. We can feed C# code from our console into the API and execute it relatively quickly.

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

Thankfully, there's an easy fix. We can just load the assemblies on another thread while our game is still loading. If it takes 1500 ms to load the assemblies we need, and our game already takes 2000 ms to load, then it's highly likely that they'll be finished loading by the time our game finishes loading. We won't need to invoke the script, just create it and grab its delegate on another thread. Here's the code I use for this:

```
Task.Run(() => {
    CSharpScript.Create("System.Console.WriteLine();").CreateDelegate();
});
```

<br/>

## References and Globals

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

The ```globalsType``` is a type that we'll be creating. It's members will be globals that we can access directly from our script. Let's make a simple one for now.

```
public class ConsoleGlobals
{
    public int Foo = 100;
}
```

Let's create a script that uses this type, passing in the ScriptOptions that we created earlier:

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

As you can see, this can be a pretty powerful tool for changing variables on-the-fly.

<br />

## Hooking it up to an in-game Console

To hook this up to a developer console in-game, we need to parse the input string as a script whenever we submit it. For this I have a ```CommandProcessor``` class that looks something like this:

```
public class CommandProcessor
{
	public DebugConsole DebugConsole { get; }

	public CommandProcessor(DebugConsole debugConsole)
	{
		DebugConsole = debugConsole;
	}

	public void ParseInput(string input)
	{
		DebugConsole.Write($">{input}\n", textColor: Color.LightGreen, userInput: true);

		// Here is where we'll run our input as a script.
	}
}
```

The ```CommandProcessor``` has a reference to the ```DebugConsole``` that created it. ```DebugConsole``` is the class that my in-game developer console is implemented in. Whenever we get input to parse (```DebugConsole``` calling our ```ParseInput``` method when we press 'enter'), we'll output it back to the console in a light green color before processing it. The 'userInput' bool is used to tell when a line written to the console was user input or not (so that the 'up' arrow can traverse previous lines of input.) How ```DebugConsole``` is implemented is unimportant; we're just focusing on processing console commands given to us by it.

At this point, we could just put a call to ```CSharpScript.Create()``` with our input string as the code, whatever references and imports we want, and a globals type, but we'd like to create the script and delegate on a separate thread so that it doesn't block our update thread. We'll still be outputting our input string in light green immediately, which will help the console feel responsive despite some commands taking ~200 ms to compile and execute. We'll create a ```ScriptCompilationResult``` class that our ```Task``` will return:

```
public class ScriptCompilationResult
{
    public bool Success { get; }
    public CompilationErrorException Exception { get; }
    public ScriptRunner<object> Script { get; }

    public ScriptCompilationResult(ScriptRunner<object> script)
    {
        Success = true;
        Exception = null;
        Script = script;
    }

    public ScriptCompilationResult(CompilationErrorException exception)
    {
        Success = false;
        Exception = exception;
        Script = null;
    }
}
```

Then, we'll store a ```Task<ScriptCompilationResult>``` private field called ```_compilationResult``` in our ```CommandProcessor``` class, and use it in our ```ParseInput()``` method.

```
public void ParseInput(string input)
{
    DebugConsole.Write($">{input}\n", Color.LightGreen, userInput: true);

    if (_compilationResult != null)
	{
		// We'll make this method in a moment.
		_compilationResult = CompileScript(input);
	}
	else
	{
		DebugConsole.Write("Error: Compilation in progress.\n", Color.Red);
	}
}
```

If our result already exists (which can only mean that we tried to enter another command while one was still compiling), we'll write an error to the console saying that the compilation is already in progress. A queue would be more robust, but I want to _notice_ if any commands are so slow to compile that I was able to write a second one before the first one finished compiling. Now let's look at the ```CompileScript()``` method:

_(Note that_ ```ManaGame``` _is not a typo, but the name of my engine's_ ```Game``` _subclass.)_

```
private Task<ScriptCompilationResult> CompileScript(string code)
{
    return Task.Run(() =>
    {
        try
        {
            var script = CSharpScript.Create(
                code: code,
                options: ScriptOptions.Default
                    .WithReferences(
                        typeof(Game).Assembly,					  // MonoGame Assembly
                        typeof(ManaGame).Assembly,				  // My Assembly
                        typeof(RuntimeBinderException).Assembly)  // Required for 'dynamic'
                    .WithImports(
                        "System",
                        "Microsoft.Xna.Framework"),
                globalsType: typeof(ManaGlobals))
                    .CreateDelegate();

            return new ScriptCompilationResult(script);
        }
        catch (CompilationErrorException e)
        {
            return new ScriptCompilationResult(e);
        }
    });
}
```

We create and start the task at the same time. Within it, we create the script and delegate in a try-catch block. If it succeeds, we return a new ```ScriptCompilationResult``` with the delegate passed to its constructor. If we got a ```CompilationErrorException```, we pass the exception into the constructor instead.

Now, we'll check to see if the task is done in an ```Update()``` method in our ```CommandProcessor``` class:

```
public void Update()
{
	if (_compilationResult == null) return;

	if (_compilationResult.IsCompleted)
	{
		var scriptResult = _compilationResult.Result;
		
		if (scriptResult.Success)
		{
			scriptResult.Script.Invoke(ManaGlobals.Instance);
		}
		else
		{
			DebugConsole.Write($"Error: {scriptResult.Exception.Message}\n", Color.Red);
		}

		_compilationResult = null;
	}
}

```

We exit early if the ```Task``` doesn't exist. If it does, we check if it's completed. If it is, we grab the result, and either invoke the script on our update thread or write the error to the ```DebugConsole```, then set ```_compilationResult``` to null so that we can send more commands. I'm using a singleton for my globals object, so I pass ```ManaGlobals.Instance``` into the delegate invocation. 

Here's my globals class:

```
public class ManaGlobals
{
	public static ManaGlobals Instance => _instance ?? (_instance = new ManaGlobals());
	private static ManaGlobals _instance;

	public dynamic Variables { get; } = new ExpandoObject();
}
```

I use an ```ExpandoObject``` for a field called 'Variables'. ```ExpandoObject``` is an object that can have members dynamically added at runtime. This means that we can write something like:

```
Variables.SomeText = "Hello, World!";
```

and then, on a subsequent call, use the variable:

```
Console.WriteLine(Variables.SomeText); // Output: Hello, World!
```

You can set initial values for members of this ```Variables``` object anywhere in your code. If you're tweaking parameters for your player's jump mechanics, you can initialize some values on ```Variables``` when your player is first created, and then reference them in your jump code. Then you're free to change the values in your console and watch the changes get reflected instantly.

<br/>

There's one final thing you might want. If you want to be able to output the returned value of your script (if there is one) like a REPL, our ```Script.Invoke()``` call returns a ```Task<object>```. We can just store that task's result and write it to the console if it's not null:

```
var returnedValue = scriptResult.Script.Invoke(ManaGlobals.Instance).Result;

if (returnedValue != null)
{
	Console.WriteLine(returnedValue.ToString());
}
```

<br/>

## Conclusion

Thanks for making it this far! This is my first post, so there's a decent change that I've explained too much of what's obvious and not enough of what isn't. Feel free to comment down below or email me if you have any feedback.

Here's that gif from the top again:

![Example image](/images/console_anim.gif#center-border)

Final notes:

* In this gif, I had a ```Color``` field on my global type called ```BackgroundColor```. This was before I thought of using an ```ExpandoObject``` instead.

* I also have a custom ```TextWriter``` set for ```System.Console``` so that calling ```Console.WriteLine``` writes to my in-game developer console.

* Check out the [**Roslyn Scripting API samples**](https://github.com/dotnet/roslyn/wiki/Scripting-API-Samples) on github. This page has enough information to teach you how to expand our implementation to be able to declare variables in your console e.g. ```var myInt = 10;``` and retain them in subsequent calls.

* The Scripting API does add quite a few dependencies to your project. If you're only using the console as a dev-tool and don't plan on shipping it with your game, you'll want to only reference the dependencies in the ```DEBUG``` configuration, and use ```#if DEBUG``` when you use them.

* The font I'm using is called [**Nintendo DS BIOS**](https://www.dafont.com/nintendo-ds-bios.font). It's free for personal use.