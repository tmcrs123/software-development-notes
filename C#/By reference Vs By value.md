
# What gets passed by reference or by value

|Type Category|Example Types|Passed By (Default)|Notes|
|---|---|---|---|
|**Value Types**|`int`, `float`, `double`, `bool`, `char`|**By Value**|A copy is passed. Changes inside the method do not affect the original.|
||`struct`, `enum`, `decimal`, `DateTime`|**By Value**|Includes user-defined structs. Can be large—use `in` for efficiency.|
|**Reference Types**|`class`, `string`, `object`, `dynamic`|**By Reference**|The _reference_ is passed by value. Object contents can still be changed.|
||`array[]`, `List<T>`, `Dictionary<K,V>`|**By Reference**|Collections are also reference types.|
|**Nullable Value**|`int?`, `bool?`, etc.|**By Value**|Nullable versions of value types behave like value types.|
|**Pointers**|`int*`, `void*` (unsafe context)|**By Reference**|Must use `unsafe` keyword; rare in managed C# code.|
|**Delegates**|`Action`, `Func`, custom delegates|**By Reference**|They're reference types under the hood.|
|**Interfaces**|`IDisposable`, `IEnumerable`, etc.|**By Reference**|Interfaces are reference types.|
# `in` , `out` and `ref`

`in` means "this is passed by reference and it's read-only"
`ref` means "this is passed by reference but can be changed"
`out` means "this variable needs to be set before the method completes"

All these are passed by reference regardless!
```C#
 public static void DoThings(ref string fruit, in string otherFruit, out int someNumberThatNeedsSetting)
 {
     someNumberThatNeedsSetting = 42;
     fruit = "bananas";
     otherFruit = "tomato" // Error! cannot reassign, it's an in parameter
 }
```