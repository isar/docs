---
id: crud
title: Create, Read, Update, Delete
---

When you have your Collections defined, learn how to manipulate them!

## Opening Isar

Before you can do anything, you have to open an Isar instance. Each instance needs a directory with write permission.

```dart
final isar = await openIsar();
```

You can use the default config or provide some of the following parameters.

| Config          | Description                                                                                                                                                                                                |
| --------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| `name`          | You can open multiple instances with distinct names. By default, `"isar"` is used.                                                                                                                         |
| `directory`     | The storage location for this instance. You can pass a relative or absolute path. By default, `NSDocumentDirectory` is used for iOS and `getDataDirectory` for Android. The final location is `path/name`. |
| `maxSize`       | Sets the maximum instance size. The default is 1GB and you probably don't need to change it.                                                                                                               |
| `encryptionKey` | Optional 32 byte (256bit) key to encrypt the database                                                                                                                                                      |

You can either store the Isar instance in a global variable or use your favorite dependency injection package to manage it.

If an instance is already open, calling `openIsar()` will yield the existing instance regardless of the specified parameters. That's useful for using isar in an isolate.

## Collections

The Collection object is how you find, query, and create new records of a given type.

### Get a collection

All your collections live in the Isar instance. Remember the `Contact` class we annotated before with `@Collection()`. You can get the contacts collection with:

```dart
final contacts = isar.contacts;
```

That was easy!

### Get a record (by id)

```dart
final contact = await contacts.get(someId);
```

`get()` returns a `Future`. All Isar operations are asynchronous by default. Most operations have a synchronous counterpart:

```dart
final contact = contacts.getSync(someId);
```

### Query records

Find a list of records matching given conditions using `.where()`:

```dart
final allContacts = await contacts.where().findAll();
final starredContacts = await contacts.where().filter().isStarred().findAll();
```

➡️ Learn more: [Queries](queries)

## Modifying the database

To create, update, or delete records, use the respective operations wrapped in a write transaction:

```dart
await isar.writeTxn(() async {
  final contact = await contacts.get(someId)

  contact.isStarred = false;
  await contacts.put(contact); // perform update operations

  await contact.delete(contact.id); // or delete operations
});
```

➡️ Learn more: [Transactions](transactions)

### Create a new record

When an object is not yet managed by Isar, you need to `.put()` it into a collection. If the id field is `null`, Isar will assign an auto-increment id.

```dart
final newContact = Contact()
  ..firstName = "Albert"
  ..lastName = "Einstein"
  ..isStarred = true;
await isar.writeTxn(() async {
  await contacts.put(newContact);
})
```

### Update a record

Both creating and updating works with `yourCollection.put(yourObject)`. If the id is null (or does not exist), the object is inserted, otherwise it is updated.

### Delete records

```dart
await isar.writeTxn(() async {
  contacts.delete(contact.id);
});
```

or:

```dart
await isar.writeTxn(() async {
  final idsOfUnstarredContacts = await contacts.where().filter().isNotStarred().idProperty().findAll();
  contacts.deleteAll(idsOfUnstarredContacts);
});
```
