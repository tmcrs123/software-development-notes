Events in C# allow you to do pub/sub logic just like you would do in Angular using `output()`;

However they are setup in a more verbose way.

## Setup / mental model

Events are tightly coupled with delegates. You need to understand what a delegate is before you understand events.

You will see this written a lot online:

> A delegate is a pointer to a function. 

I think it makes more sense to think of it like a specification for a function. 

A delegate is defined like this (consider an alarm clock as an example):

```C#
public delegate void AlarmHandler(string message);
```

This syntax means the property - `AlarmHandler` - is a specification for a function that takes a string as an argument and returns nothing. 

They weirdness here comes from the fact that you cannot simply assign a function to the delegate. The following won't compile:

```C#
AlarmHandler = (string message) => {
	Console.WriteLine(message) //this won't compile
}
```

So what's the point of this? 

The point is you have now defined a structure for a function that you can provide to other parts of your code. In particular to an `event`

## Using `delegates` in `event`

When you are working with events in any language, you also need to specify what happens when that event is triggered.

This of course differs from language to language and in some of them the syntax makes more sense than in C# (think of `.subscribe()` or `(click)=doThings())`  in Angular). Usually there's a split between the event being fired and the code that runs when the event is fired (think callbacks here).

However in C# there is only one entity that deals with this: the `event` itself.

I think this is where most of confusion comes from.

The way you go about declaring an event is like this:

```C#
public event ??? AlarmFired
```

Notice the `???` which is not a typo. In C# we are talking about a strongly typed language. So unlike with typescript where you could register any function with any parameters as an event handler, in C# the compiler wants you to be more specific. You need to specify *exactly* the structure of your event handler.

Now, at this point it would be really useful if we had a way to define the structure for our handler function and provide it as a parameter to the event definition. And this is where the delegates come into play. You have to use a delegate to tell the event what shape an event handler function has:

```C#
public event AlarmHandler AlarmFired;
```

This bit of code reads as follows: "Declare a property named AlarmFired, which is an event that requires that follows the structure of the AlarmHandler delegate"

With this knowledge you now need to provide the handlers to event.

Again the syntax for this is, imo, a bit weird. This is how you do it:

```C#
AlarmFired += (message) => {
	Console.WriteLine(message)
}
```

Or if you have some class that implements this:

```C#
class Person {
	public void RespondToAlarm(string message){
		Console.WriteLine(message);
	}
}

(...)

AlarmFired += Person.RespondToAlarm
```

The last thing you need to do is to fire the event and your handlers should fire. This is how you do it:

```C#
AlarmFired.Invoke("wake up")
```

Notice that in this case, because we said our handler will take a string parameter, when you call `Invoke` you need to provide that string parameter. The `Invoke` method will always have the same number of parameters as the delegate. The string provided here is the string that will make it's way to the handlers.