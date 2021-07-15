---
id: schema
title: Schema
---

When using Isar, you're dealing with Collections. A collection can only contain a single type of Dart object. To let Isar know which objects you want to store, you need to annotate your classes with `@Collection()`. The Isar code generator will take care of the rest. All the collections combined are called the "database schema".

## Annotating classes

The Isar generator will find all classes annotated with `@Collection()`. All model classes need to define an id by annotating a property with `@Id()`. Only `int` properties may be used as id.

```dart
@Collection()
class Contact {
  @Id()
  int? id;

  String firstName;

  String lastName;

  bool isStarred;
}
```

### Id

Each class needs and `int` field annotated with `@Id()` that uniquely identifies an object. If a class has a field called `id`, you can omit the `@Id()` annotation.

```dart
@Collection()
class Pet {
  int? id; // field is called id so an @Id() annotation is not required

  String name;
}
```

Isar automatically indexes id fields, which allows you to efficiently read and modify objects based on their id.

You can either set ids yourself or by setting them to `null` request Isar to assign an auto-increment id.

### Supported types

Isar supports the following data types:
- `bool`
- `int`
- `double`
- `DateTime`
- `String`
- `Uint8List`
- `List<bool>`
- `List<int>`
- `List<double>`
- `List<DateTime>`
- `List<String>`

It is important to understand how nullability works in Isar:
Number types do **NOT** have a dedicated `null`-representation. Instead a specific value will be used. `int` uses the `int.MIN` value and `double` uses `double.NaN`. `bool`, `String` and `List` have a separate `null` representation.

While this may sound strange at first, it allows you to change the nullability of your fields freely without requiring migration or special code to handle `null`s.

➡️ Use `TypeConverter`s to store unsupported types like enums: [Type Converters](type-converters)

### 8-byte and 4-byte numbers

`int` and `double` have an 8-byte representation in Dart. By default, this is also true for Isar. You can however change numbers to a 4-byte representation to save disk space by annotating number fields with `@Size32()`. It is your responsibility to make sure that you do not store a number that requires eight byte in a `@Size32()` field.

### Ignoring fields

By default, all public fields of a class will be persisted. By annotating a field with `@Ignore()`, you can exclude it from persistence. Keep in mind that it is not good practice to store information in your Isar objects that is not persisted.

### Renaming classes and fields

You have to be careful when you want to rename a class or field. Most of the time the old class or field will just be droped and recreated. With the `@Name()` annotation, you can name classes and fields in the database independantly from Dart. The following code will yield the exact same schema as the code above.

```dart
@Collection()
@Name("Contact")
class MyContactClass1 {
  @Id()
  @Name("id")
  int? myObjectId;

  @Name("firstName")
  String theFirstName;

  @Name("lastName")
  String familyNameOrWhatever;

  bool isStarred;
}
```


## Schema migration

It is possible to change the schema between releases of your app (for example by adding collections) but it is very important to follow the rules of schema migration.

You are allowed to do the following modifications:
 - Add new collections
 - Remove existing collections (the data will be deleted)
 - Change the nullability of a field (e.g. `int` -> `int?` or `List<String?>` -> `List<String>`)
 - Add fields
 - Remove fields
 - Rename collections and fields annotated with `@Name()`
 - Add indexes
 - Remove indexes

Be careful: If you rename a field or collection that is not annotated with `@Name()`, the field or collection will be dropped and recreated.

**ILLEGAL MODIFICATIONS**
- Changing the type of fields in existing collections
- Changing the id of a collection
