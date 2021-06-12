---
id: indexes
title: Indexes
---

Understanding how indexes work is essential to optimize query performance. Isar lets you choose which index you want to use and how you want to use it. We'll start with a quick introduction what indexes are.

## What are indexes?

When a collection is unindexed, the order of the rows will likely not be discernible by the query as optimized in any way, and your query will therefore have to search through the objects linearly. In other words, the queries will have to search through every object to find the objects matching the conditions. As you can imagine, this can take a long time. Looking through every single object is not very efficient.

For example, the fictional collection is completely unordered.

```dart
@Collection()
class Product {
  int? id;

  late String name;

  late int price;
}
```

#### Data:

| id | name | price |
| --- | --- | --- |
| 1 | Book | 15 |
| 2 | Table | 55 |
| 3 | Chair | 25 |
| 4 | Pencil | 3 |
| 5 | Lightbulb | 12 |
| 6 | Carpet | 60 |
| 7 | Pillow | 30 |
| 8 | Computer | 650 |
| 9 | Soap | 2 |

A query that tries to find all products that cost more than €30 has to search through all nine rows. That's not an issue for nine rows but it will become a problem for 100k rows.

```dart
final expensiveProducts = await isar.products.where()
  .filter()
  .priceGreaterThan(30)
  .findAll();
```

To improve the performance of this query we index the `price` propery. An index is like a sorted lookup table:

```dart
@Collection()
class Product {
  int? id;

  late String name;

  @Index()
  late int price;
}
```

#### Generated index:

| price | id |
| --- | --- |
| 2 | 9 |
| 3 | 4 |
| 12 | 5 |
| 15 | 1 |
| 25 | 3 |
| 30 | 7 |
| **55** | 2 |
| **60** | 6 |
| **650** | 8 |

Now the query can be executed a lot faster. The executer can directly jump to the last three index rows and find the corresponding objects by their id.

### Sorting

Another cool thing indexes can do is super fast sorting. Sorted queries are very expensive because the database has to load all results in memory before sorting them. Even if you specify an offset or limit because they are applied after sorting.

Let's imagine we want to find the four cheapest products. We could use the following query:

```dart
final cheapest = await isar.products.where()
  .sortByPrice()
  .limit(4)
  .findAll();
```

In this example the database would have to load all (!) objects, sort them by price and return the four products with the lowest price.

As you can probably imagine, this can be done a lot more efficient with the index from before. The database takes the first four rows of the index and returns the corresponding objects.

To use the index for sorting we would write the query like this:

```dart
final cheapestFast = await isar.products.where()
  .anyPrice()
  .limit(4)
  .findAll();
```

The `.anyX()` where clause tells Isar to use an index just for sorting. You can also use a where clause like `.priceGreaterThan()` and still get sorted results.


## Composite indexes

A composite index is an index on multiple properties. Isar allows you to create composite indexes that consist of up to three properties.

A composite index is also known as a multiple-column index.

It's probably best to start with an example. We create a person collection and define a composite index on the age and name properties:

```dart
@Collection()
class Person {
  int? id;

  late String name;

  @Index(composite: [CompositeIndex('name')])
  late int age;

  late String hometown;
}
```

#### Data:

| id | name | age | hometown |
| --- | --- | --- | --- |
| 1 | Daniel | 20 | Berlin |
| 2 | Anne | 20 | Paris |
| 3 | Carl | 24 | San Diego |
| 4 | Simon | 24 | Munich |
| 5 | David | 20 | New York |
| 6 | Carl | 24 | London |
| 7 | Audrey | 30 | Prague |
| 8 | Anne | 24 | Paris |

#### Generated index

| age | name | id |
| --- | --- | --- |
| 20 | Anne | 2 |
| 20 | Daniel | 1 |
| 20 | David | 5 |
| 24 | Anne | 8 |
| 24 | Carl | 3 |
| 24 | Carl | 6 |
| 24 | Simon | 4 |
| 30 | Audrey | 7 |

The generated composite index contains all persons sorted by their age and then by their name.

Obviously composite indexes are great if you want to create efficient queries that are sorted by multiple properties.

But composite indexes also allow advanced where clauses with multiple properties:

```dart
final result = await isar.where()
  .ageNameEqualTo(24, 'Carl')
  .hometownProperty()
  .findAll() // -> ['San Diego', 'London']
```

The last property of a composite index also supports conditions like `startsWith()` or `lessThan()`:

```dart
final result = await isar.where()
  .ageEqualToNameStartsWith(20, 'Da')
  .findAll() // -> [Daniel, David]
```

## Unique indexes

A unique index ensures the index does not contain any duplicate values.

A unique index may consist of one or multiple properties. If a unique index has one property, the values in this property will be unique. In case the unique index has multiple property, the combination of values in these property is unique.

Any attempt to insert or update data into the unique index that causes a duplicate will result in an error.

### Replace indexes

A replace index is always unique. The only difference to a regular unique index is what happens if you try to insert a duplicate value. Rather than throwing an exception the replace index will replace existing objects with the same value.


## Index type

There are multiple different type for indexes:

### Value index

This is the default type and also the only allowed type for all properties that don't hold Strings. Property values are used to build the index.

### Hash index

Strings are hashed to reduce the storage required by the index. The disadvantage of hash indexes is that they can't be used for prefix scans (`startsWith` where clauses.

### Word index

Strings are splitted on Grapheme Clusters or word boundaries, according to the [Unicode Standard Annex #29](https://www.unicode.org/reports/tr29/) rules and stored individually. Can be used for full-text search.