---
id: quickstart
title: Quickstart
slug: /
---

## 1. Add dependencies

First, add Isar to your project. Add the following packages to your `pubspec.yaml`. Always use the latest version.

```yaml
dependencies:
  isar: any
  isar_flutter: any

dev_dependencies:
  isar_generator: any
  build_runner: any
```

For non-Flutter projects, you need to manually include the Isar Core binaries.


## 2. Annotate classes

Annotate your classes with `@Collection` and choose an id field.

```dart
@Collection()
class Contact {
  @Id()
  int? id;
  
  String name;
}
```

## 3. Run code generator

Execute the following command to start the `build_runner`:

```
dart run build_runner build
```


## 4. Open Isar instance

This opens an Isar instance at the default location.

```dart
final isar = await openIsar();
```


## 5. Write and read from database

Once your instance is open, you can start using the database.

```dart
final contact = Contact()
  ..name = "My first contact";

await isar.writeTxn((isar) async {
  isar.contacts.put(contact);
});

final allContacts = await isar.contacts.where().findAll();
```
