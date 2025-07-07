LINQ is SINGLE THREAD!!! (Unless you are doing Parallel Linq)

## Mapping -> SelectMany

This is a sort of flattening operator. The "Many" part of it it's because it goes `n` levels deep and flattens the output. For example:

```C#
var sequences = new[] { "red,green,blue", "yellow", "gray, pink" };
var flat = sequences.SelectMany(x => x.Split(',')); //red,green,blue,yellow,gray,pink
```

There's also another weird feature which is the ability to flatten 2 collections in one go. This is like finding all possible pairs from 2 collections. Like this:

```C#
string[] obj = { "house", "car", "bycicle" };
string[] colors = { "red", "green", "blue" };

Console.WriteLine(JsonSerializer.Serialize(colors.SelectMany(_ => obj, (c,o) => $"{c} {o}")));}
```

## Filtering

Apart from `Where` there's also a `ofType<T>` operator:

```C#
var integers = dump.OfType<int>(); //IEnumerable<int>
```

# Grouping

Know that `GroupBy` has an overload that allows you to chose only some properties of an object to keep. For example if you have a group of people and want to keep only the first name you would do this

```C#
var grouped = persons.GroupBy(p => p.age, y => y.first);
```

This will give you something that is an `IGrouping` . An `IGrouping` is kind off like a dictionary. It has a key property but it does not have a value property.  You can do this:

```C#
foreach (var ageKey in grouped)
{
    Console.WriteLine(ageKey.Key); // the age, an int

    foreach (var name in ageKey)
    {
        Console.WriteLine(name); // the name, a string
    }
}
```

Here's a better example of what `GroupBy` does step by step:

```C#
var list = new List<(string sapNumber, string filePath)>
{
    ("A1", "file1"),
    ("A2", "file2"),
    ("A1", "file3"),
    ("A3", "file4"),
    ("A2", "file5"),
    ("A4", "file6")
};

var groups = list.GroupBy(t => t.sapNumber); 

// the line above turns the tupples into something like this:
[
  Group "A1": [("A1", "file1"), ("A1", "file3")],
  Group "A2": [("A2", "file2"), ("A2", "file5")],
  Group "A3": [("A3", "file4")],
  Group "A4": [("A4", "file6")]
]
```

# Sets

3 things here:
- Intersect - what's common to both
- Except - what exists in one but not the other
- Union - everything that exists in both. Note that this removes duplicates! For example string "helloooooo" and "help" -> "helop"

# Quantifiers

Nothing special here:

3 operators:
- any
- contains
- all

# Partitioning 

This is skipping and taking. There's `skip, skipWhile, take, takeWhile`

```c#
var seq1 = new int[] { 3, 3, 2, 2, 1, 1, 2,2,3,3 };
var b = seq1.Skip(2).Take(3); // 2,2,1
var c = seq1.SkipWhile(i => i == 3); // first number 2 onwards (2,2...)
var d = seq1.TakeWhile(i => i == 3); // only first 2 elements (3,3)
```

# Joining

## Join

2 types of joining here: normal `Join` and `GroupJoin`

`Join` can be interpreted as an `inner join` between 2 collections.  Take the following example:

```C#
var people = new Person2[]
{
    new Person2("tiago", "tiago@gmail"),
    new Person2("maria", "maria@gmail"),
    new Person2("ana", "ana@gmail"),
    new Person2("ricardo", "ricardo@gmail"),
};

var records = new Record[]
{
    new Record("tiago@gmail", "tiago@gmailSID"),
    new Record("maria@gmail", "maria@gmailSID"),
    new Record("ana@gmail", "ana@gmailSID"),
    new Record("ricardo@gmail", "ricardo@gmailSID")
};

var query = people.Join(records, p => p.email, r => r.Mail, (person,record) =>
{
    return new { Name = person.name, SkypeId = record.SkypeId };
});
```

The query RHS of the query variable can be read as `PEOPLE INNER JOIN RECORDS ON PEOPLE.EMAIL = RECORDS.EMAIL` . The lambda function as the 4th argument is what you want to return from the joining operation. Also note than it returns all matching elements even if there is more than one. For example this:

```C#
var people = new Person2[]
{
    new Person2("tiago", "tiago@gmail"),
    new Person2("maria", "maria@gmail"),
    new Person2("ana", "ana@gmail"),
    new Person2("ricardo", "ricardo@gmail"),
};

var records = new Record[]
{
    new Record("tiago@gmail", "tiago@gmailSID"),
    new Record("tiago@gmail", "tiago@gmailSID2"),
    new Record("tiago@gmail", "tiago@gmailSID3"),
    new Record("maria@gmail", "maria@gmailSID"),
    new Record("ana@gmail", "ana@gmailSID"),
    new Record("ricardo@gmail", "ricardo@gmailSID")
};

var query = people.Join(records, p => p.email, r => r.Mail, (person,record) =>
{
    return new { Name = person.name, SkypeId = record.SkypeId };
});
```

 ...returns `[{"Name":"tiago","SkypeId":"tiago@gmailSID"},{"Name":"tiago","SkypeId":"tiago@gmailSID2"},{"Name":"tiago","SkypeId":"tiago@gmailSID3"},{"Name":"maria","SkypeId":"maria@gmailSID"},{"Name":"ana","SkypeId":"ana@gmailSID"},{"Name":"ricardo","SkypeId":"ricardo@gmailSID"}]`

## Group Join

Group join works like join in terms of syntax but logically is a bit different because you group elements first. This only makes sense if you have  a one to many relationship from one collection to the other. 

In terms of syntax it's the same but the lambda part is different. In this case you get an `IEnumerable` of all the matching records (whereas if you did `join` if and had matching records you would get them separately)

```C#
var query2 = people.GroupJoin(records, p => p.email, r => r.Mail, (person, records) => //records is an IEnumerable of all matching records
{
    return new { Name = person.name, SkypeIds = records.Select(r => r.SkypeId).ToArray() };
});
```

The output of the query above is as follows: `[{"Name":"tiago","SkypeIds":["tiago@gmailSID","tiago@gmailSID2","tiago@gmailSID3"]},{"Name":"maria","SkypeIds":["maria@gmailSID"]},{"Name":"ana","SkypeIds":["ana@gmailSID"]},{"Name":"ricardo","SkypeIds":["ricardo@gmailSID"]}]`

# Equality Comparison

This is the `SequenceEquals` operator. This works to check if 2 IEnumerables have the same sequence of values. However this is only good for primitives. If you need to compare 2 collections of a specific type you need to override `Equals` and `GetHashCode` at class level. For example, if you want this to be `true`...

```C#
var peopleOne = new Person2[]
{
    new Person2("tiago", "tiago@gmail"),
    new Person2("maria", "maria@gmail"),
    new Person2("ana", "ana@gmail"),
    new Person2("ricardo", "ricardo@gmail"),
};

var peopleTwo = new Person2[]
{
    new Person2("tiago", "tiago@gmail"),
    new Person2("maria", "maria@gmail"),
    new Person2("ana", "ana@gmail"),
    new Person2("ricardo", "ricardo@gmail"),
};

Console.WriteLine(peopleOne.SequenceEqual(peopleTwo));
```

... you need to override the `Person` class as follows: 
```c#
public class Person2
{
    public string name;
    public string email;

    public Person2(string first, string skypeAddress)
    {
        this.name = first;
        this.email = skypeAddress;
    }

    public override bool Equals(object? obj)
    {
        if (obj == null || GetType() != obj.GetType())
        {
            return false;
        }

        Person2 other = (Person2)obj;

        return other.email == this.email && other.name == this.name;
    }

    public override int GetHashCode()
    {
        return HashCode.Combine(this.email, this.name);
    }
}
```

 Note about `GetHashCode` : `SequenceEquals` works fine if just the `Equals` comparison but it's good practice to implement both because some structures like `Dictionary` rely on the hash to compare elements. If you don't implement it you could get inconsistent results

# Element operations

Operations that return a single value OR throw an exception. Operators are:

- Single
- SingleOrDefault
- First
- FirstOrDefault
- Last
- LastOrDefault
- ElementAt
- ElementAtOrDefault

Note that you can pass a predicate to `First` and `Last` to return the first (or last) element that matches a condition.

# Concat

Concat 2 collections together. No magic here. But here's a cool way to extend `IEnumerable`

```C#
static class IEnumerableExtensions
{
    public static IEnumerable<T> Prepend<T>(this IEnumerable<T> values, T value)
    {
        yield return value;
        foreach (var v in values)
        {
            yield return value;
        }
    }
}
```

# Aggregation

2 operators where: `Count` and `Aggregate`. `Count` is what it says on the box, `Aggregate` is the same as `Reduce` in javascript. 

``` c#
int[] a = new int[] { 1, 2, 3, 4, 5, 6, 7, 8, 9 };

a.Aggregate((acc, next) => acc + next); // 45
```

You can also choose to specify a seed value like this:

``` c#
int[] a = new int[] { 1, 2, 3, 4, 5, 6, 7, 8, 9 };

a.Aggregate(5, (acc, next) => acc + next); // 50
```

## Parallel LINQ

By default LINQ executes in one thread. If you do invoke `AsParallel` you are moving to a multithread paradigm.

```C#
int count = 50;
IEnumerable<int> numbers = Enumerable.Range(0, count);

int[] cubes = new int[count];

numbers.AsParallel().ForAll(x =>
{
    cubes[x] = x;
});
```

Note: `ForAll` is like `Select` here. Is something you apply to each element of the collection but for parallel stuff. 
Note 2: `Ordered` vs `Unordered` . This means "maintain the order for the output of the LINQ statement". It does not mean "process items in order". The advantage of doing `Ordered/Unordered` is that it either keeps the order if you need it OR, if you don't need it, makes it more performant (if you specify `Unordered`)

# Cancellation tokens and merge options

Take the following example:

```C#
var cts = new CancellationTokenSource();
var items = ParallelEnumerable.Range(1, 20);

var results = items
    .WithCancellation(cts.Token)
    .WithMergeOptions(ParallelMergeOptions.FullyBuffered)
    .Select(i =>
{
    double result = Math.Log10(i);
    Console.WriteLine(result);
    Console.WriteLine(Thread.CurrentThread.ManagedThreadId);
    if (i > 1)
    {
        cts.Cancel();
        throw new InvalidOperationException();
    }
    return result;
});

try
{
    foreach (var r in results)
    {
        Console.WriteLine(r);
    }
}
catch (AggregateException ag)
{
    ag.Handle(e =>
    {
        Console.WriteLine(e.Message);
        return true;
    });
}
```

Focus on Cancellation token and merge options.

Cancellation tokens is your way of saying "cancel whatever threads are doing work". What this means is that you don't get immediate cancellation because some threads may be processing when the cancellation token is invoked. That's also why the exception thrown is an `AggregateException`. Think of this like one exception for thread

`WithMergeOptions`, specifies how you want the items to be delivered. If you think about in when you are in a parallel LINQ world you essentially have a producer/consumer pattern. MergeOptions lets you specify how you want the process items to be delivered (buffered, all at once, etc...)

