---
id: queries
title: Queries
---

Querying is how you find records that match certain conditions, for example:

- Find all starred contacts
- Find distinct first names in contacts
- Delete all contacts that don't have a last name defined

Because queries are executed on the database, and not in Dart, they're really fast. When you cleverly use indexes, you can improve the query performance even further. In the following, you'll learn how to write queries and how you can make them as fast as possible.

There are two different methods of filtering your records: Filters and where clauses. We'll start by taking a look at how filters work.

## Filters
Filters are easy to use and understand. Depending on the type of your properties, there are different filter operators available most of which have self-explanatory names.

Filters work by evaluating an expression for every object in the collection being filtered. If the expression resolves to true, Isar includes the object in the results. Filters have no effect on the ordering of the results.

We'll use the following model for the examples below:

```dart
@Collection()
class Shoe {
  int id;

  int? size;

  String model;

  bool isUnisex;
}
```

### Query conditions

Depending on the type of a field, there are different conditions available.

Condition | Description
--- | ---
`.equalTo(value)` | Matches values that are equal to the specified `value`. 
`.between(lower, upper)` | Matches values that are between `lower` (included) and `upper` (included).
`.greaterThan(bound)` | Matches values that are greater than `bound`.
`.lessThan(bound)` | Matches values that are less than `bound`. `null` values will be included by default because `null` is considered smaller than any other value.
`.isNull()` | Matches values that are `null`. 

Let's assume the database contains four shoes with sizes 39, 40, 46 and one with unset (`null`) size. Unless you perform sorting, the values will be returned sorted by id.

```dart 

isar.shoes.where().filter().sizeLessThan(40).findAll() // -> [39, null]

isar.shoes.where().filter().sizeLessThan(40, include: true).findAll() // -> [39, null, 40]

isar.shoes.where().filter().sizeBetween(39, 46, includeLower: false).findAll() // -> [40, 46]

```

### Logical operators

You can composit predicates using logical operators.

Operator | Description
--- | ---
`.and()` | Evaluates to true if both left-hand and right-hand expressions are true.
`.or()` | Evaluates to true if either expression returns true.
`.not()` | Negates the result of the following expression.
`.group()` | Group conditions and allow to specify order of evaluation.

If you want to find all shoes with size 46, you can use the following query:

```dart
final result = await isar.shoes.where().filter().sizeEqualTo(46).findAll();
```

If you want to use more than one condition, you can combine multiple filters using logical **and** `.and()` and logical **or** `.or()`:

```dart
final result = await isar.shoes.where().filter()
  .sizeEqualTo(46)
  .and() // Optional. Filters are implicitly combined with logical and.
  .isUnisexEqualTo(true)
  .findAll();
```

You can also group conditions using `.group()`:
```dart
final result = await isar.shoes.where().filter()
  .sizeBetween(43, 46)
  .and()
  .group((q) => q
    .modelNameContains('Crocs')
    .or()
    .isUnisexEqualTo(false)
  )
  .findAll()
```

To negate a condition or group, use logical **not** `.not()`:

```dart
final result = await isar.shoes.where().filter()
  .not().sizeEqualTo(46)
  .and()
  .not().isUnisexEqualTo(true)
  .findAll();
```

### String conditions

You can compare string values using these string operators. Regex-like wildcards allow more flexibility in search.

Condition | Description
--- | ---
`.startsWith(value)` | Matches string values that begins with provided `value`.
`.contains(value)` | Matches string values that contain the provided `value`.
`.endsWith(value)` | Matches string values that end with the provided `value`.
`.matches(wildcard)` | Matches string values that match the provided `wildcard` pattern. 

**Case sensitivity**  
All string operations have an optional `caseSensitive` parameter that defaults to `true`.

**Wildcards:**  
A [wildcard string expression](https://en.wikipedia.org/wiki/Wildcard_character) is a string that uses normal characters with two special wildcard characters:
- The `*` wildcard matches zero or more of any character
- The `?` wildcard matches any character.
For example, the wildcard string `"d?g"` matches `"dog"`, `"dig"`, and `"dug"`, but not `"ding"`, `"dg"`, or `"a dog"`.

### Query modifiers

Sometimes it is necessary to build a query based on some conditions or for different values. Isar has a very powerful tool to build conditional queries:

Modifier | Description
--- | ---
`.optional(condition, queryBuilder)` | Extends the query only if the `condition` is `true`.
`.repeat(values, queryBuilder)` | Extends the query for each value in `values`.

Modifiers can be combined however you like. Every part of a query can be optional or repeated but it does not always make sense. Repeatedly applying a limit for example will effectively only use the limit applied last.

In this example we build a method that can find shoes with an optional filter:
```dart
Future<List<Shoe>> findShoes(int? sizeFilter) {
  return isar.shoes.where()
    .filter()
    .optional(
      sizeFilter != null,
      (q) => q.sizeEqualTo(sizeFilter!), // only apply filter if sizeFilter != null
    ).findAll();
}
```

If you want to find all shoes that have one of multiple shoe sizes you can either write a conventional query or use the `repeat()` modifier:

```dart

final shoes1 = await isar.shoes.where()
    .filter()
    .sizeEqualTo(38)
    .or()
    .sizeEqualTo(40)
    .or()
    .sizeEqualTo(42)
    .findAll();

final shoes2 = await isar.shoes.where()
    .filter()
    .repeat(
      [38, 40, 42],
      (q, int size) => q.sizeEqualTo(size).or()
    ).findAll();

// shoes1 == shoes2
```

### Links

If your model contains [links or backlinks](links) you filter your query based on the linked objects.

```dart
@IsarCollection()
class Teacher {
    int id;

    String subject;
}

@IsarCollection()
class Student {
    int id;

    String name;

    final teachers = IsarLinks<Teacher>();
}
```

We can for example find all students that have a math or English teacher:

```dart
final result = await isar.students.where()
  .filter()
  .teachers((q) => q.subjectEqualTo('Math')
    .or()
    .subjectEqualTo('English')
  ).findAll()
```

Link filters evaluate to `true` if at least one linked object matches the conditions.


## Where clauses
Where clauses are a very powerful tool but it can be a little difficult to get them right.

In contrast to filters where clauses use the indexes you defined in the schema. Querying an index is a lot faster than filtering each record individually. As a basic rule, you should always try to reduce the records as much as possible using where clauses and do the remaining filtering using filters.

You can combine where clauses using logical **or**.

Let's add indexes to the shoe collection:

```dart
@Collection()
class Shoe with IsarObject {
  int id;

  @Index()
  int? size;

  String model;

  @Index(composite: [CompositeIndex('size')])
  bool isUnisex;
}
```

There are two indexes. The index on `size` allows us to use where clauses like `.sizeEqualTo()`. The composite index on `isUnisex` allows where clauses like `isUnisexSizeEqualTo()`. But also `isUnisexEqualTo()` because you can also use any prefix of an index.

We can now rewrite the query from before that finds unisex shoes in size 46 using the composite index. This query will be a lot faster than the previous one:

```dart
final result = isar.shoes.where().isUnisexSizeEqualTo(true, 46).findAll();
```

Where clauses have two more superpowers: They give you "free" sorting and a super fast distinct operation.

## Sorting

You can define how the results should be sorted when executing the query using the `.sortBy()`, `.sortByDesc()`, `.thenBy()` and `.thenByDesc()` methods.

To find all shoes sorted by model name in acending order and size in descending order:

```dart
final sortedShoes = isar.shoes.where().sortByModel().thenBySizeDesc().findAll();
```

Sorting a lot of results can be an expensive operation. Luckily, we can again use indexes an make our query lighning fast even if we need to sort a million objects.

### Where clause sorting

If you use a single index in your query, the results are already sorted by the index. That's a big deal!

Let's assume we have shoes in sizes `[43, 39, 48, 40, 42, 45]` and we want to find all shoes with a size greater than or equal to `42` and also have them sorted by size:

```dart
final bigShoes = isar.shoes.where().sizeGreaterThan(42).findAll();
// -> [43, 45, 48]
````

As you can see, the result is sorted by the `size` index. If you want to reverse the sort order, you can set `ascending` to `false`:

```dart
final bigShoesDesc = await isar.shoes.where(ascending: false).sizeGreaterThan(42).findAll();
// -> [48, 45, 43]
```

Sometimes you don't want to use a where clause but still benefit from the implicit sorting. You can use the `any` where clause:

```dart
final shoes = await isar.shoes.where().anySize().findAll();
// -> [39, 40, 42, 43, 45, 48]
```

If you use a composite index, the results are sorted by all fields in the index.


:::tip General rule of thumb:

If you need the results to be sorted, consider using an index for that purpose. Especially if you work with `offset()` and `limit()`.

:::

Sometimes it's not possible or useful to use an index for sorting. For such cases, you should use indexes to reduce the number of resulting entries as much as possible.

## Unique values

To return only entries with unique values, use the distinct predicate. For example, to find out how many different shoe models you have in your Isar database:

```dart
final shoes = await isar.shoes.where().distinctByModel().findAll();
```

You can also chain multiple distinct conditions for example to find all shoes with distinct model-size combinations:

```dart
final shoes = await isar.shoes.where()
  .distinctByModel()
  .distinctBySize()
  .findAll();
```

### Where clause distinct

If you have a non unique index, you may want to get all of its distinct values. You could use the `distinctBy` operation but it's performed after sorting and filters so there is some overhead to it.  
If you only use a single where clause you can instead rely on the index to perform the distinct operation.

Another great advantages of indexes is that you get "free" sorting. When you query results using a **single** where clause, the results are sorted by the index. For composite indexes, the result are sorted by all fields in the index.


## Offset & Limit

It's often a good idea to limit the number of results from a query. You can do so by setting a `limit()`:

```dart
final firstTenShoes = await isar.shoes.where().limit(10).findAll();
```

By setting an `offset()` you can also paginate the results of your query.

```dart
final firstTenShoes = await isar.shoes.where()
  .offset(20)
  .limit(10)
  .findAll();
```

## Execution order

Isar queries are always executed in the same order:
1. Traverse primary or secondary index to find objects
2. Filter objects
3. Sort results
4. Apply distinct operation
5. Offset & limit results
6. Return results

## Query operations

In the previous examples we used `.findAll()` to retrieve all matching objects. There are more operations available however:


Operation | Description
--- | ---
`.findFirst()` | Retreive only the first matching object or `null` if none matches.
`.findAll()` | Retreive all matching objects.
`.count()` | Count how many objects match the query.
`.deleteFirst()` | Delete the first matching object from the collection.
`.deleteAll()` | Delete all matching objects from the collection.
`.build()` | Compile the query to reuse it later.

## Property queries

If you are only interested in the values of a single property, you can use a property query. Just build a normal query and select a property:

```dart
List<String> models = await isar.shoes.where().modelProperty().findAll();

List<int> sizes = await isar.shoes.where().sizeProperty().findAll();
```

Using only a single property saves time during deserialization.

## Aggregation

You can also aggregate the values of a property query. The following aggregation operations are available:

Operation | Description
--- | ---
`.min()` | Finds the minimum value or `null` if none matches.
`.max()` | Finds the maximum value or `null` if none matches.
`.sum()` | Sums all values.
`.average()` | Calculates the average of all values or `NaN` if none matches.

Using aggregations is vastly faster than finding all matching objects and performing the aggregation manually.


## Dynamic queries

All of the examples above used the QueryBuilder and the generated static extension methods. Maybe you want to create very dynamic queries or even a custom query language (like the Isar Inspector. In that case you can use the `buildQuery()` method: