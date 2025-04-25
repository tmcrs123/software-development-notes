# JSON

First thing is to understand the difference between a field and a property in C#. In JSON there's no difference, it's all key value pairs

In C#, a field is a property that does not have a getter or setter. It's a class variable that may be private or read-only.

A property is a class property that has a getter and setter. An example:

```C#
public class User
{
    public User(string name, int age)
    {
        Name = name;
        Age = age;
    }

    [JsonPropertyName("name")]
    [JsonPropertyOrder(1)]
    public string Name { get; } // property
    
    [JsonPropertyName("age")]
    [JsonPropertyOrder(0)]
    public int Age { get; set; } // property

    public string someField; // field
}
```

Note that you need the mapping in the property name for the deserialization to work. The order is optional and it just means the order in which the key value pairs in JSON will be serialized

## Serialization options

You can specify how you want things to be serialized by passing an `options` object to the serializer. Like this:

```C#
var options = new JsonSerializerOptions() { 
    WriteIndented = true,
    IncludeFields = false
};

var usersjson = JsonSerializer.Deserialize<List<User>>(file, options);
```

There are a lot of options and I won't cover all. But know some caveats:

1 - If the field is set, it gets printed regardless, even if you set `IncludeFields = false`. Think of this in the case of constants. If you set the value it comes out in JSON.

2 - `IncludeReadOnlyFields` - this one is weird. It seems to impact what gets serialized rather than what gets deserialized. The deserialization is impacted by how you define your model:

```c#
//this will never be read from JSON, it's readonly
public readonly string someField = "bananas"; 


//this will be read from JSON, it's not readonly
public string someOtherField = "more apples"; // field
```

Note that `get;set` always map to what you have in the JSON.

When it comes to deserializing the options actually make sense. It does what it says on the box.

## Converting from XML to JSON

```C#
//3.Convert XML to JSON

//var doc = XDocument.Load("users.xml"); //this is LINQ stuff!
var xml = new XmlDocument();
xml.Load("users.xml");

var nodes = xml.SelectNodes("users/user");

var users = new List<Dictionary<string, string>>();


foreach (XmlNode node in nodes)
{
    var currentUsr = new Dictionary<string, string>();
    
    foreach (XmlNode childNode in node.ChildNodes)
    {
        currentUsr[childNode.Name] = childNode.InnerText;

    }
    users.Add(currentUsr);
    Console.WriteLine(JsonSerializer.Serialize(currentUsr));
};

var serialized = JsonSerializer.Serialize(users);

Console.WriteLine(serialized);
```

2 notes here:
- `XDocument.Load("users.xml")` this is LINQ stuff for when you want to do LINQ on XML
- `JsonSerializer.Serialize(users)` is one of the very useful overloads you can pass to the serializer. Passing a `List<Dictionary<string,string>>` is cool because you can essentially saves key value pairs and just throw it at the serializer