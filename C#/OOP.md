Classes

Abstract classes:
 - Cannot be static
 - Cannot be instantiated
 - Cannot have abstract fields, only properties (meaning need to have a get, set defined)
 - Can have static methods
 - Can define concrete instance methods
 - Can define abstract instance methods if marked as abstract
 - Allows double execution on instance methods if marked as virtual
 - An abstract class can implement interfaces
 - You cannot inherit from multiple classes, abstract or not

Static:
- A concrete class can have static methods and properties. Why? Think configuration or utility functions

Interfaces:
 - An interface can have properties
 - An interface can define methods
 - An interface can define methods with a body. If you define the body you don't need to implement it in the class that implements the interface
 - An interface can extend another interface
 - Interface methods cannot be static or virtual  

## SOLID

### S - Single Responsibility Principle

A class should only have "one reason to change"/ do one thing:

In the example below, `ReportGenerator` should not be responsible for saving the file. Extract that logic to a new class.
 
```C#
internal class ReportGenerator
{
    public string GenerateReport() => "Create a report";

    public void SaveToFile() => "Saving..." //WRONG
}
```

This should be:

```C#
internal class ReportGenerator
{
    public string GenerateReport() => "Create a report";    
}

internal class ReportSaver
{
	public void SaveToFile() => "Saving..." //WRONG
}
```

## O - Open/Closed principle

"Should be open for extension but closed for modification".

What this means is: "you should avoid touching code that was already working". If you need to add more functionality then extend the class rather than modifying it.

This implicitly assume your "modifications" will introduce new concerns (thus violating the Single Responsibility Principle) and will make your code harder to maintain.

This is a big assumption and needs to be taken with a pinch of salt. If you change your business logic a bit and don't nothing else to an existing class you are unlikely to "add new responsibilities".

The spirit of this principle is more "if it's working don't fix it and think before you add more stuff".

## L - Liskov substitution principle

"If B is a subclass of A, you should be able to replace A with B without breaking the application".

Essentially this means whatever you do you can never break the contract of the base class, you can never do something that breaks its behaviour. 

Your subclass can have more methods and properties than the subclass, but it always needs to respect the contract of the base class.

Considered the following example, which complies with the LSP:

```C#
public class Animal
{
    public virtual void Eat() => Console.WriteLine("Eating");
}

public class Dog : Animal
{
    public void Bark() => Console.WriteLine("Barking");
    public override void Eat() => Console.WriteLine("Dog is eating");
}
```

```C#
Animal animal = new Dog();
animal.Eat(); // ✅ Works fine and does not break the principle
```

Now consider this example that violates the principle because the derived class breaks the expected behaviour of the base class. 

```C#
public class Animal
{
    public virtual void Eat() => Console.WriteLine("Eating");
}

public class Dog : Animal
{
    public void Bark() => Console.WriteLine("Barking");
    public override void Eat() => Throw new Exception(); //Violates the principle! 
}
```
## I - Interface Segregation Principle

Clients should not depend on interfaces they don't need aka "Don't make big interfaces, your interfaces does one thing and one thing only"

Bad example:

```C#
public interface IMachine
{
    void Print();
    void Scan();
    void Fax();
}
```

Good example:

```C#
public interface IPrinter
{
    void Print();
}

public interface IScanner
{
    void Scan();
}

public class MultiFunctionPrinter : IPrinter, IScanner
{
    public void Print() => Console.WriteLine("Printing...");
    public void Scan() => Console.WriteLine("Scanning...");
}
```

## D - Dependency Inversion Principle

"High level modules should not depend on low levels modules" aka "Your classes should not depend on other classes, it should depend on abstractions"

Bad example:

```C#
public class EmailService
{
    public void Send(string message) => Console.WriteLine("Sending: " + message);
}

public class Notification
{
    private EmailService _emailService = new EmailService();
    public void SendNotification(string message) => _emailService.Send(message);
}
```

Good example:

```C#
public interface IMessageSender
{
    void Send(string message);
}

public class EmailService : IMessageSender
{
    public void Send(string message) => Console.WriteLine("Sending: " + message);
}

public class Notification
{
    private readonly IMessageSender _sender;
    public Notification(IMessageSender sender) => _sender = sender;

    public void SendNotification(string message) => _sender.Send(message);
}
```

This also ties a lot to the "composition over inheritance" argument.

You should always prefer composition (meaning a "has a" relationship) because it's more flexible, it's easier to swap components and makes for a more decoupled code. Also it makes code much easier to test. One argument in favour of composition is that it hides the implementation details (meaning you need to implement the interface the way you need to), unlike inheritance because it exposes behaviour to the base class (assuming the is NOT abstract).

So when should you prefer abstract classes? When you have a clear hierarchical relationship and you are confident the base class will remain stable and general. But even then prefer abstract base classes and interfaces and only inherit if truly necessary.