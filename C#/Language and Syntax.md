
## Collections

| Colletion                       | Pros                                                                                        | Cons                                                      |
| ------------------------------- | ------------------------------------------------------------------------------------------- | --------------------------------------------------------- |
| List<T>                         | Dynamic size, fast access                                                                   | Slower inserts, elements need to shift                    |
| LinkedList                      | Efficient inserts/removes in the middle                                                     | High memory consumption, slower access                    |
| SortedList<TKey, TValue>        | Keeps items sorted                                                                          | Slower insertions, needs to sort on insert                |
| Dictionary                      | Fast Lookups, doesn't allow key duplication                                                 | Higher memory consumption, needs to keep an index of keys |
| Sorted Dictionary<TKey, TValue> | Self-balancing tree, better performance for frequent insertions/deletions than `SortedList` | More memory usage                                         |
| Queue                           | FIFO                                                                                        | Sucks for random access (but why would you?)              |
| Stack                           | LIFO                                                                                        | Sucks for random access (but why would you?)              |
| HashSet                         | Uniqueness                                                                                  | No ordering, high memory usage                            |

Note: SortedList is the weird one that looks like a (sorted) dictionary. The difference between the two is that the dictionary under the hood users a Binary Tree. That's why performance is better. for read and write. 

### Time Complexity

| Collection                           | Operation                       | Time Complexity | Explanation                                                                                |
| ------------------------------------ | ------------------------------- | --------------- | ------------------------------------------------------------------------------------------ |
| **`List<T>`**                        | **Add(item)**                   | O(1)            | Appends an item at the end. Occasionally, resizing happens, but this is amortized to O(1). |
|                                      | **Insert(index, item)**         | O(n)            | Inserting at a specific index requires shifting elements.                                  |
|                                      | **Remove(item)**                | O(n)            | Searches for the item and removes it, shifting elements.                                   |
|                                      | **Contains(item)**              | O(n)            | Searches for the item in the list by iterating through it.                                 |
|                                      | **IndexOf(item)**               | O(n)            | Finds the index of the item by iterating through the list.                                 |
|                                      | **Access by index**             | O(1)            | Direct access via index is O(1).                                                           |
| **`Dictionary<TKey, TValue>`**       | **Add(key, value)**             | O(1)            | Average case is O(1) for adding a key-value pair using a hash function.                    |
|                                      | **Remove(key)**                 | O(1)            | Average case is O(1), using the key's hash to find and remove the value.                   |
|                                      | **ContainsKey(key)**            | O(1)            | Average case is O(1), hash lookup for key existence.                                       |
|                                      | **ContainsValue(value)**        | O(n)            | Needs to scan all values, which may involve traversing the entire collection.              |
|                                      | **TryGetValue(key, out value)** | O(1)            | Average O(1), retrieves value for a key.                                                   |
| **`SortedList<TKey, TValue>`**       | **Add(key, value)**             | O(log n)        | Maintains sorted order, requiring O(log n) for binary search insertion.                    |
|                                      | **Remove(key)**                 | O(log n)        | Binary search for the key and removal, O(log n).                                           |
|                                      | **ContainsKey(key)**            | O(log n)        | Binary search to check for key existence.                                                  |
|                                      | **ContainsValue(value)**        | O(n)            | Linear scan of the values, requires O(n).                                                  |
|                                      | **Access by index**             | O(1)            | Index access is O(1) as it uses an internal array.                                         |
| **`SortedDictionary<TKey, TValue>`** | **Add(key, value)**             | O(log n)        | Binary search tree insertion, O(log n).                                                    |
|                                      | **Remove(key)**                 | O(log n)        | Tree structure allows for O(log n) removal.                                                |
|                                      | **ContainsKey(key)**            | O(log n)        | Binary search for key existence.                                                           |
|                                      | **ContainsValue(value)**        | O(n)            | Requires O(n) to search through the values.                                                |
|                                      | **TryGetValue(key, out value)** | O(log n)        | Binary search in a tree, O(log n).                                                         |
| **`Queue<T>`**                       | **Enqueue(item)**               | O(1)            | Adds an item at the end of the queue.                                                      |
|                                      | **Dequeue()**                   | O(1)            | Removes an item from the front of the queue.                                               |
|                                      | **Peek()**                      | O(1)            | Retrieves the front item without removing it.                                              |
|                                      | **Contains(item)**              | O(n)            | Requires a linear scan of the queue.                                                       |
| **`Stack<T>`**                       | **Push(item)**                  | O(1)            | Adds an item to the top of the stack.                                                      |
|                                      | **Pop()**                       | O(1)            | Removes the top item from the stack.                                                       |
|                                      | **Peek()**                      | O(1)            | Retrieves the top item without removing it.                                                |
|                                      | **Contains(item)**              | O(n)            | Requires a linear scan of the stack.                                                       |
| **`HashSet<T>`**                     | **Add(item)**                   | O(1)            | Hash-based insertion, O(1) on average.                                                     |
|                                      | **Remove(item)**                | O(1)            | Hash-based removal, O(1) on average.                                                       |
|                                      | **Contains(item)**              | O(1)            | Hash lookup for item, O(1) on average.                                                     |
| **`LinkedList<T>`**                  | **AddFirst(item)**              | O(1)            | Adds an item to the front of the list.                                                     |
|                                      | **AddLast(item)**               | O(1)            | Adds an item to the end of the list.                                                       |
|                                      | **RemoveFirst()**               | O(1)            | Removes the first item in the list.                                                        |
|                                      | **RemoveLast()**                | O(1)            | Removes the last item in the list.                                                         |
|                                      | **Remove(item)**                | O(n)            | Searches for the item and removes it.                                                      |
|                                      | **Contains(item)**              | O(n)            | Searches for the item by traversing the list.                                              |
|                                      | **Find(item)**                  | O(n)            | Traverses the list to find the item.                                                       |
O(1) -> always the same regardless of collection size
O(n) -> linear to the size of the collection
O(n$^2$) -> exponential to the size of the collection (this is a no no)
O(log(n)) -> increases slowly as the collection gets bigger ( this may be acceptable)

# Delegates

Delegates are essentially function definitions. You create the delegate specifying inputs and outputs and then at some point you need to define it.

```C#
public delegate int AddDelegate(int x, int y);

public static Func<int, int, int> AnotherAddDelegate { get;set; }
```

The code above represents 2 delegates. You can define them in 2 ways:
- Using the delegate type meaning it's a custom delegate
- Using the `FUNC<>` type which is just a generic delegate

With a delegate you can do things such as:

```C#
public static void DoThings()
{
    AddDelegate add = (int a, int b) => a + b;
    AddDelegate add2 = (int a, int b) => a * b;
    AnotherAddDelegate = (int a, int b) => a + b;
}
```

Note: a delegate **is not** a lambda! A lambda is an anonymous function. Is the right side of the equal sign. In the example above the lambda is:
```C#
(int a, int b) => a + b;
```

# IDisposable

It's an interface that exposes a mechanism for releasing unmanaged resources.

What is an unmanaged resource? Anything not picked up by garbage collection. For example:

- File Streams
- Database Connections
- Network Connections


`IDispose` is not magic. In fact you don't really need to implement it, you could just dispose of your resources manually

```C#
_writer.Dispose();
_fileStream.Dispose();
```

The only reason for you to implement the interface is to be able to do `using` so that C# automatically invokes your implementation of `Dispose`

```C#
 internal class FileHandler : IDisposable
 {
     public FileHandler(string path)
     {
         _fileStream = new FileStream(path, FileMode.Create, FileAccess.Write);
         _writer = new StreamWriter(_fileStream);
     }

     public FileStream _fileStream;
     public StreamWriter _writer;
     public bool _disposed = false;

     public void Dispose()
     {
         Console.WriteLine("Disposed invoked");
         _writer.Dispose();
         _fileStream.Dispose();
         _disposed = true;
     }

     public void WriteData(string data)
     {
         if (_disposed)
         {
             throw new ObjectDisposedException(nameof(FileHandler));
         }
         _writer.WriteLine(data);
     }
 }
```

# Expression Trees

Expression trees are a way of turning functions (like lambdas) into syntax that an underlying database can understand. Think of this like what Entity Framework uses under the hood to transform a query into SQL.

Whenever the C# compiler seems an Expression - `Expression<Func<int, int>> f2 = x => x + 1;` - internally it transforms it into an "expression tree". 

## What is an "expression tree" ?

You can think of it as an object representation of your function. Something like:

```
{
	Body: {
		Right: 'x+1',
		Left: 'x'
	},
	Parameters: ['x']
}
```

This structure is what allows C# to transform this into a SQL query.

# IQueryable

You can think of  ```IQueryable``` as a bridge between the LINQ syntax and some underlying ORM or database engine. The `IQueryable` interface is what any provider needs to implement if they want to integrate LINQ with their data source.  `IQueryable` has the same set of LINQ operators but each provider is free to implement them how they wish. Under the hood, `Iqueryable` uses expression trees because it needs to be able to convert whatever you give into ORM/Database language. Remember that `IQueryable` never returns a value until the query is executed (essentially when you do stuff on a `IQueryable` like for example `Select(), Where(), Aggregate()` you are just "building" your query. You only get the result back once you execute the query).

## `ICollection` vs `IEnumerable`

Both are a way of working with collections. The difference is `IEnumerable` is a read-only interface. You can iterate over the data set but you cannot do things like count, add or remove items, etc. `ICollection` on the other hand is fully featured, meaning you can do more things like counting or adding/removing. `IEnumerable` is lightweight, `ICollection` is heavier because it's has more features. Both support LINQ

# Tasks vs Value Tasks

A `ValueTask` is a lightweight implementation of `Task`. Note that `async await` methods ALWAYS return a `Task` even if you don't add a `return` statement

## Tasks Vs Actions

Tasks represent an async operation. Actions are a pointer to a void delegate. Think of actions as a reference to a "fire and forget" function.

# `decimal vs float vs double`

`decimal` is for higher precision calculation. This is the slowest and more memory intensive type. This is for stuff like money where fractions of cents might be important.

`float` is the other end of the spectrum. This has less precision and therefore is less memory intensive. This is for cases where you need a number with decimals but it's ok if it's not super accurate. Think for example game development.

`double` is something in between the two. It's better precision and faster than `float` but it's not as "good" as a `decimal`

## `class` vs `record` vs `struct`

Think of this as 3 different "levels of service", like a subscription model where you have "free, basic and premium"

### Struct

Structs are the most basic form of an object. They are meant to be used for simple lightweight object. Thinks of for things like LatLng, Point, Color, etc...

Structs are passed by value and the equality comparison is based on the value of the properties (like records). 

One thing to note is that structs don't come by default with the  `==` operator, you need to define it yourself.
### Record

Records are designed for value equality comparisons (meaning go and check all the properties and see if they have the same value). Records are passed by **value**. This is useful for transactional objects like DTOs. 

*note about immutability: records are meant to be immutable but this only achieved by not specifying getters and setters. There's nothing special about the `record`  keyword that makes a record immutable.*

*note about methods and classes: records can have methods, records can have an instance of a class has a property, records support inheritance*
### Class

A class is a class. Supports everything, passed by reference, equality comparison on references not value unless overloaded

## CLR - Common Language Runtime

Is the runtime environment in .NET. Handles things like:
 - memory management
 - garbage collection
 - exceptions
 - code compilation
 - interoperability (invoking unmanaged code)

Another linked concept here is managed code.

Managed code is any code that's executed outside of the .NET CLR context. Think of it this way: what if you wanted to call a windows API from within .NET? .NET doesn't know how to do this itself, it needs to reach out to Windows APIs, probably written in some other language. That code will be unmanaged because it runs outside of the context of .NET. .NET cannot help you there in terms of garbage collection or memory management, you need to handle it yourself. Running unmanaged code requires extra steps importing the dll. 