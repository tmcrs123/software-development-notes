There are 2 main classes to work with dates: `DateTime` and `TimeSpan`

`DateTime` refers to a date and time like `08:59 23/04/2025` and `TimeSpan` refers to the time part of the date, `08:59`

The default `TimeSpan` is `00:00:00`, the default `DateTime` is `01/01/0001 00:00:00`

## Common operations
### Parse a date from a string

Use `tryParseExact` and not `tryParse` . Difference is the `exact` version needs to have everything specified, the other version does a best effort and can lead to errors.

```C#
public static void ParseDateFromString(string date)
{
    DateTime result = default;
    
    bool conversionOk = DateTime.TryParseExact(date, "MM/dd/yyyy HH:mm:ss", System.Globalization.CultureInfo.InvariantCulture, System.Globalization.DateTimeStyles.None, out result);

    Console.WriteLine($"Day is {result.Day}, month is {result.Month}, year is {result.Year}");

    if(result.TimeOfDay != TimeSpan.Zero)
    {
        Console.WriteLine($"Hour is {result.Hour}, Minute is {result.Minute}, Second is {result.Second}");
    }
}
```

### Doing math with dates

Just need to remember the `TimeSpan` class. This is how you do math. However the `DateTime` class also has convenience methods like `AddYear` or `AddMonth`. For example `TimeSpan` does not have a way to add years, you need to add 365 days.

```C#
public static void SubtractTimeSpan(string date)
{
    DateTime parsedDate = default;

    DateTime.TryParseExact(date, "MM/dd/yyyy HH:mm:ss", System.Globalization.CultureInfo.InvariantCulture, System.Globalization.DateTimeStyles.None, out parsedDate);

    parsedDate = parsedDate.Add(new TimeSpan(365,0,0,0));
    parsedDate = parsedDate.Subtract(new TimeSpan(12,0,0));

    Console.WriteLine(parsedDate.ToLongTimeString());
}
```
 