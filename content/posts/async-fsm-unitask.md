---
title: "Async FSM using UniTask"
tags: ['unity', 'c#', 'async', 'unitask', 'fsm']
date: 2022-04-22T00:02:06+07:00
author: ["Jussi Tuomi"]
draft: false
---

## Introduction

In this post I'm going through steps to implement an asynchronous finite-state machine (FSM) in Unity, using async/await library [UniTask](https://github.com/Cysharp/UniTask). In the end you'll have a nice modular state machine with all the usual stuff you would expect to find in a FMS. We'll also take a look on how we can run update loops independently of monobehaviours / gameobjects.

You can follow along or hop directly to my GitHub to explore the [repository](https://github.com/jushii/AsyncFSM) which contains the full project.

### Requirements
- Unity 2020.2+
- UniTask

I recommend installing UniTask [via git URL](https://github.com/Cysharp/UniTask#install-via-git-url) using Unity's package manager.

## States

Alright, let's start with the states. We create an interface `IState` and three abstract classes: `State`, `State<T>` and `Options`. `State` and `State<T>` are the base classes for all state implementations and `Options` can be used as a container for custom properties when transitioning to a new state.

```cs
public interface IState
{
}
```

```cs
public abstract class State : State<Options>
{
}

public abstract class State<T> : IState where T : Options
{
}
```

```cs
public abstract class Options
{
}
```

The `IState` interface contains the blueprint of our state. States know which state machine they belong to and they'll use that reference to request state transitions. The `OnEnter` and `OnExit` methods will both return `UniTask` struct which makes them awaitable.

The `SetOptions` method will be called each time we do a state transition. The state machine supports states with and without options.

To keep things simple we add in only one update method: `OnUpdate`. Later I'll introduce you to the _true power_ 😱 of UniTask and show how easily you can hook in to different timings in Unity's player loop. You can even inject your own player loop timings to the state machine.

```cs
using Cysharp.Threading.Tasks;

public interface IState
{
    StateMachine StateMachine { get; set; }
    UniTask OnEnter();
    UniTask OnExit();
    void SetOptions(Options options);
    void OnUpdate();
}
```

Next, let's implement the members of `IState` to the `State<T>` class. If you're wondering about `await UniTask.Yield()`, it's the UniTask's replacement for `yield return null`. We'll make the methods virtual so our derived states can override only the methods they need.

```cs
using Cysharp.Threading.Tasks;

public abstract class State : State<Options>
{
}

public abstract class State<T> : IState where T : Options
{
    public T Options { get; private set; }

    public virtual async UniTask OnEnter()
    {
        await UniTask.Yield();
    }

    public virtual async UniTask OnExit()
    {
        await UniTask.Yield();
    }

    public void SetOptions(Options options)
    {
        if (options is T stateOptions)
        {
            Options = stateOptions;
        }
    }

    public virtual void OnUpdate()
    {
    }
}
```
## Transitions

For handling state transitions we'll create two classes: `Transition` and `Transition<T>`.

```cs
public class Transition : Transition<Options>
{
}

public abstract class Transition<T> where T : Options
{
}
```

A transition needs the state `Type` so we know to which state we want to transition in to. We can also provide `Options` which can be used for setting up state properties before the state's `OnEnter()` method is called.

```cs
using System;

public class Transition : Transition<Options>
{
    public Transition(Type type, Options options) : base(type, options)
    {
    }
}

public abstract class Transition<T> where T : Options
{
    public Type Type { get; }
    public T Options { get; }

    protected Transition(Type type, T options)
    {
        Type = type;
        Options = options;
    }
}
```

## Implementing the state machine

Now let's create the `StateMachine` class and add in some members.

```cs
using System;
using System.Collections.Generic;
using Cysharp.Threading.Tasks;

public class StateMachine
{
    private IState _currentState;
    private IState _previousState;
    private readonly Dictionary<Type, IState> _states = new();
    private readonly Queue<Transition> _pendingTransitions = new();
}
```

We create a dictionary to keep track of registered states. If you dislike using `Type` as the dictionary key, you could as well use a `string` or an `enum` of your choice. We'll also create a queue for keeping track of the requested state transitions.

Now it's time to add in some methods. First add in a method for registering new states.

```cs
public void RegisterState(IState state)
{
    state.StateMachine = this;
    
    _states.Add(state.GetType(), state);
}
```

We create two methods for requesting transitions. When a transition is requested it will added into a queue. The queue will be processed in our state machine's update loop.

```cs
public void RequestTransition(Type stateType)
{
    _pendingTransitions.Enqueue(new Transition(stateType, null));
}

public void RequestTransition<T>(Type stateType, T options) where T : Options
{
    _pendingTransitions.Enqueue(new Transition(stateType, options));
}
```

The `ChangeTo` method will be responsible of handling a transition. We'll again return a `UniTask` struct so we can await for it's completion.

```cs
private async UniTask ChangeTo<T>(Type stateType, T options) where T : Options
{
}
```

We'll implement a typical FSM-style transition where we first exit the current state (if any) and then enter the next state. Keeping track of the current state as we do this. You'll probably want to include better validation for certain things. For now let's just throw an exception if we try to transition to a state which is not registered to our state machine.

```cs
private async UniTask ChangeTo<T>(Type stateType, T options) where T : Options
{
    if (_currentState != null)
    {
        _previousState = _currentState;
        await _previousState.OnExit();
        _currentState = null;
    }

    if (_states.TryGetValue(stateType, out IState nextState))
    {
        nextState.SetOptions(options);
        _currentState = nextState;
        await nextState.OnEnter();
    }
    else
    {
        throw new Exception($"State: {stateType.Name} is not registered to state machine.");
    }
}

```
## The async update loop

This is where things get somewhat interesting. [Async enumerables](https://docs.microsoft.com/en-us/archive/msdn-magazine/2019/november/csharp-iterating-with-async-enumerables-in-csharp-8) is a C# 8.0 feature which, using UniTask, allows us to use a new update notation that lets us to inject our code directly into Unity's `PlayerLoop`, breaking us free from the shackles of `MonoBehaviour`.

```cs
private async void Update()
{
    await foreach (var _ in UniTaskAsyncEnumerable.EveryUpdate())
    {
    }
}
```

Using `UniTaskAsyncEnumerable` we can emulate the `Update()` method of the `MonoBehaviour` component in a pure C# class. This is very powerful and useful for many other things besides this state machine. You can read more about async enumerables in the context of Unity and UniTask in the [UniTask repository](https://github.com/Cysharp/UniTask#asyncenumerable-and-async-linq).

By default `EveryUpdate` uses `PlayerLoopTiming.Update` but you can easily change it to something else (for example to `PlayerLoopTiming.FixedUpdate`). For this example we're only going to implement a standard `Update` loop.

Everything looks good so far, but before we continue with the update loop let's discuss how we can actually start and stop the update. Let's create methods called `Run()` and `Stop()`.

```cs
public void Run()
{
}

public void Stop()
{
}
```

To stop the state machine from running we need a `CancellationToken`. So let's add in a field for cancellation token source to our state machine class. We provide this token to the `UniTaskAsyncEnumerable` in our update loop. The token is used to request the cancellation of the enumerator.

```cs
using System;
using System.Collections.Generic;
using System.Threading;
using Cysharp.Threading.Tasks;
using Cysharp.Threading.Tasks.Linq;

public class StateMachine
{
    private IState _currentState;
    private IState _previousState;
    private readonly Dictionary<Type, IState> _states = new();
    private readonly Queue<Transition> _pendingTransitions = new();
    private CancellationTokenSource _cancellationTokenSource;
    ...
```

When we call `Run` to start the state machine we create a new `CancellationTokenSource` and fire up our async `Update()` method. To actually use the token we have to provide it using the `WithCancellation` method.

```cs
public void Run()
{
    _cancellationTokenSource = new();
    Update();
}
```
```cs
private async void Update()
{
    await foreach (var _ in UniTaskAsyncEnumerable
    .EveryUpdate()
    .WithCancellation(_cancellationToken.Token))
    {
    }
}
```

To stop the state machine from running we call `Cancel` on the cancellation token source to request a cancellation of the enumerator. We will also dispose the cancellation token source.

```cs
public void Stop()
{
    _cancellationTokenSource.Cancel();
    _cancellationTokenSource.Dispose();
}
```

Now the only things remaining are to actually process the transition queue and calling the `OnUpdate()` method of the current state.
```cs
private async void Update()
{
    await foreach (var _ in UniTaskAsyncEnumerable
    .EveryUpdate()
    .WithCancellation(_cancellationToken.Token))
    {
        while (_pendingTransitions.Count > 0)
        {
            var transition = _pendingTransitions.Dequeue();
            await ChangeTo(transition.Type, transition.Options);
        }

        _currentState?.OnUpdate();
    }
}
```

## Examples

Let's put the state machine to test by implementing a very basic example. We will create a state machine that has two states and we'll also test out a transition between those states. In this example our state machine is running independently of `MonoBehaviour`. You can of course have the state machine be a member of a class that is derived from monobehaviour or use it in any other way you like.

Create the following classes: `Example`, `ExampleState`, `ExampleStateWithOptions` and `ExampleStateOptions`.

`ExampleState` is a basic state with no options. We'll request a transition after a 2 second delay. You'll notice that `OnUpdate` never gets called because we request the transition already inside the `OnEnter` method.

```cs
using Cysharp.Threading.Tasks;
using UnityEngine;

public class ExampleState : State
{
    public override async UniTask OnEnter()
    {
        Debug.Log("Entering ExampleState! Waiting 2 seconds before changing state.");

        await UniTask.Delay(2000);

        var options = new ExampleStateOptions
        {
            text = "Hello world!"
        };

        StateMachine.RequestTransition(typeof(ExampleStateWithOptions), options);
    }

    public override async UniTask OnExit()
    {
        Debug.Log("Exiting ExampleState!");

        await UniTask.Yield();
    }

    public override void OnUpdate()
    {
        // This is never called because we request a transition in OnEnter.       
        Debug.Log("Calling OnUpdate in ExampleState!");
    }
}
```

`ExampleStateWithOptions` uses a custom options container. You'll notice that the options are already initialized when we enter the `OnEnter` method. Options are a great way for some state-specific initialization which you might want to run before the `OnUpdate` method is called.

```cs
using Cysharp.Threading.Tasks;
using UnityEngine;

public class ExampleStateOptions : Options
{
    public string text;
}

public class ExampleStateWithOptions : State<ExampleStateOptions>
{
    public override async UniTask OnEnter()
    {
        Debug.Log($"Entering ExampleStateWithOptions. Here's our options text: {Options.text}");

        await UniTask.Yield();
    }

    public override void OnUpdate()
    {
        Debug.Log($"realTimeSinceStartup: {Time.realtimeSinceStartup}, frameCount:{Time.frameCount}");
    }
}
```

In `Example` class we'll create a new state machine, create and register 2 states, request a transition to the initial state and call `Run` to start the state machine.

```cs
using UnityEngine;

public static class Example
{
    private static StateMachine _stateMachine;

    [RuntimeInitializeOnLoadMethod]
    private static void Initialize()
    {
        _stateMachine = new StateMachine();
        _stateMachine.RegisterState(new ExampleState());
        _stateMachine.RegisterState(new ExampleStateWithOptions());
        _stateMachine.RequestTransition(typeof(ExampleState));
        _stateMachine.Run();
    }
}
```

Now, if you hit play in Unity and take a look at the console, you should see the state machine in action!