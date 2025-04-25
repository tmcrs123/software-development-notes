
## `jsonb_to_recordset`

This is when you have a JSON **ARRAY** and you want to turn it into rows (a table output). The way this works is you need to tell it what element you want to extract for the json object. For example for this json:

```json
[{"name": "batman"}, {"name": "superman"}]
```

I want a table will all the names.

```sql
SELECT 
	* 
FROM jsonb_to_recordset('[{"name": "batman"}, {"name": "superman"}]'::jsonb) AS x(name TEXT);
```

The output is:

|     | Name     |
| --- | -------- |
| 1   | batman   |
| 2   | superman |

## `to_jsonb`

Alternative you can also turn any row of your table into json using the `to_jsonb` function. For example:

```SQL
select to_jsonb(heroes)
from heroes
limit 2
```

Gives you:

```json
"{""id"": 1, ""lastname"": ""Lastname1"", ""firstname"": ""Hero1"", ""description"": {""name"": ""vision"", ""team"": ""Guardians of the Galaxy"", ""origin"": ""Alien"", ""weakness"": ""adamantium"", ""alignment"": ""hero"", ""superpower"": ""intangible body""}}"
"{""id"": 2, ""lastname"": ""Lastname2"", ""firstname"": ""Hero2"", ""description"": {""name"": ""star-lord"", ""team"": ""Guardians of the Galaxy"", ""origin"": ""Asgard"", ""weakness"": ""technology"", ""alignment"": ""villain"", ""superpower"": ""marksmanship""}}"
```

This works even if the rows already have a jsonb column.

If you want to transform into json but you don't want all the rows you can do use an inner query like this:

```SQL
select to_jsonb(filtered)
from (select firstname,lastname from heroes) as filtered
limit 5
```

## Getting data from jsonb fields

There are 2 operators that are relavant here: `->` and `->>`

The difference is that `->` returns values as JSON and `->>` extracts the value from the JSON as text **always**, even if the json is a boolean or an integer.

```sql
SELECT 
	('{"name": "batman", "age": 24}'::jsonb)->>'age' // age will be "24"
```

If you need the value as any other type you need to cast it manually

### jsonb_path_query

This is a way to query jsonb columns without having to use `-> / ->>`

Think of this almost like jQuery for jsonb.

You have property/element accessors as follows:

|JSONPath Element|Meaning|
|---|---|
|`$`|The **root** of the JSON document|
|`.`|Access a **child** key|
|`[*]`|Access **all elements** in an array|
|`[n]`|Access the **n-th element**|
|`?()`|Apply a **filter predicate**|
So you can query like this:

```sql
SELECT jsonb_path_query(description, '$.name') FROM heroes;
```

A note about applying a filter predicate:

The syntax for this is a bit weird.

```SQL
SELECT jsonb_path_query(description, '$.name ? (@ like_regex "V*")') 
FROM heroes;
```

The `@` sign in the filter predicate means "the current item being evaluated".

In this case, as I'm already selecting a `name` property, the `@` will refer to the name as a string. Think of this almost like `var i = element` in a `for...loop`.

This makes more sense if you have an array like this:

```sql
SELECT jsonb_path_query(
    '{"heroes": [{"name": "Batman", "superpower": "fighting skills"}, {"name": "Flash", "superpower": "super speed"}, {"name": "Vigilante", "superpower": "stealth"}]}',
    '$.heroes[*] ? (@.name like_regex "v%")'
);
```

## Updating jsonb columns

You have the naive way:

```sql
update heroes
set description = '{"name": "tiago","team": "whatever","origin": "algueirao- pt"}'
where id = 102
```

And you have the clever way:

```sql
update heroes
set description = description || jsonb_build_object('origin', 'algueirao- pt')
where id = 102
```

`jsonb_build_object` is a function that allows you to build json objects. It takes as many arguments as you want and it's always `(json_key,json_value,json_key,json_value...)`

You can even give it arrays:

```sql
update heroes
set description = description || jsonb_build_object('origin', 'algueirao-pt', 'bananas', 'yellow', 'colors', '["a","b","c"]')
where id = 102
```

Also a note about the `||` operator.

This is the string concatenation operator but in the context of jsonb it's used as a merge operator. Much like spread syntax in javascript.

## JSONB Indexes

There are 2 types of index that apply to JSONB columns: BTree (binary search tree) and GIN (global inverted index).

At a very high level the differences are the way you plan to use the index. BTREEs are good for getting top level json objects, stuff like "find where team = 'guardians%'".

GINs on the other hand are good when you want to look **inside** the json.

When you specify a GIN index you can also specify an option for *how* that GIN index should work. You have 2 options:

- `jsonb_ops` - this is the default value, you don't even need to specify it. This is the most general option and good for both scenarios of getting the top level json as well as looking inside the jsonb
- `jsonb_path_ops` - this is optimized for queries that mostly look inside the JSONB

Example:

```SQL
CREATE INDEX heroes_index
ON heroes
USING GIN (description jsonb_path_ops);
```


Here's a summary of the differences:

| Feature            | **BTREE**                                                        | **GIN**                                                                    |
| ------------------ | ---------------------------------------------------------------- | -------------------------------------------------------------------------- |
| **Best for**       | Exact matches, equality comparisons                              | Searching keys, values, or arrays in JSONB                                 |
| **Efficiency**     | Fast for simple queries, equality checks, and range queries      | Fast for containment queries, nested structures, and large documents       |
| **Indexing JSONB** | Works on **extracted values** (e.g., text values from JSON keys) | Works directly on **entire JSONB objects** (supports deeper searches)      |
| **Space**          | More compact, smaller index sizes                                | Larger index size due to indexing every individual key-value pair          |
| **Query Type**     | Ideal for exact matches like `=`, `IN`, `BETWEEN`                | Ideal for `@>`, `?`, `?&`, and other containment and key existence queries |
## Updating inside a JSONB column

No update directly inside the jsonb
Need to pack/unpack

```sql
select jsonb_agg(
	case
		when details ->> 'imageId' = 'abc123' then jsonb_set(details, '{legend}', to_jsonb('bananas'::text))
		else details
	end
	)
from (
	select jsonb_array_elements(image_details) as details
	from images 
	where atlas_id = 'c8680bbf-5654-4a7d-aa89-dfde26aa07a5'
)
```