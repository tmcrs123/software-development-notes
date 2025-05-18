# What type should you use for money?

Decimal, higher precision, better for financial calculations.

f - Float
m - deciMal
d - Double
# Common Operations
## Converting a string to currency

```C#
public static void CurrencyToDecimal(string money)
{
    decimal result;
    decimal add = 100.69m;

    var success = decimal.TryParse(money, System.Globalization.NumberStyles.Currency, System.Globalization.CultureInfo.GetCultureInfo("en-US"), out result);

    Console.WriteLine(result+add);
}
```

*note: all the globalization stuff is optional. This is just to add context*
## Formatting a decimal as currency

```C#
public static void FormatDecimalAsCurrency(decimal value)
{
    string formatted = value.ToString("C", System.Globalization.CultureInfo.GetCultureInfo("en-GB"));
    Console.WriteLine(formatted);
}
```

A note about the `"C"` . This is how you tell C# that you want the decimal value to be interpreted as currency. There are others options such as `N` for numbers with groupings (1,234.00) or `E` for scientific notation (1.234560E+003).

In this case you do need to specify the culture in order to get the output with currency symbols.

## Rounding

```C#
public static void Round(decimal value, int places)
{
    var res = Math.Round(value, places, MidpointRounding.ToZero);
    Console.WriteLine(res);
}
```

Nothing special here apart from that `MidpointRounding` thing. This is just a modifier to tell C# how to do the rounding (either up or down). Need to investigate this a bit more.
 