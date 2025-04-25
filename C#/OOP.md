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