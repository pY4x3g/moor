---
title: "Writing queries"
linkTitle: "Writing queries"
description: Learn how to write database queries in pure Dart with moor
aliases:
 - /queries/
weight: 100
---

{{% pageinfo %}}
__Note__: This assumes that you already completed [the setup]({{< ref "_index.md" >}}).
{{% /pageinfo %}}

For each table you've specified in the `@UseMoor` annotation on your database class,
a corresponding getter for a table will be generated. That getter can be used to
run statements:
```dart
// inside the database class, the `todos` getter has been created by moor.
@UseMoor(tables: [Todos, Categories])
class MyDatabase extends _$MyDatabase {  

  // the schemaVersion getter and the constructor from the previous page
  // have been omitted.
  
  // loads all todo entries
  Future<List<Todo>> get allTodoEntries => select(todos).get();

  // watches all todo entries in a given category. The stream will automatically
  // emit new items whenever the underlying data changes.
  Stream<List<Todo>> watchEntriesInCategory(Category c) {
    return (select(todos)..where((t) => t.category.equals(c.id))).watch();
  }
}
```
## Select statements
You can create `select` statements by starting them with `select(tableName)`, where the 
table name
is a field generated for you by moor. Each table used in a database will have a matching field
to run queries against. Any query can be run once with `get()` or be turned into an auto-updating
stream using `watch()`.
### Where
You can apply filters to a query by calling `where()`. The where method takes a function that
should map the given table to an `Expression` of boolean. A common way to create such expression
is by using `equals` on expressions. Integer columns can also be compared with `isBiggerThan`
and `isSmallerThan`. You can compose expressions using `a & b, a | b` and `a.not()`. For more
details on expressions, see [this guide]({{< relref "expressions.md" >}}).
### Limit
You can limit the amount of results returned by calling `limit` on queries. The method accepts
the amount of rows to return and an optional offset.
### Ordering
You can use the `orderBy` method on the select statement. It expects a list of functions that extract the individual
ordering terms from the table.
```dart
Future<List<Todo>> sortEntriesAlphabetically() {
  return (select(todos)..orderBy([(t) => OrderingTerm(expression: t.title)])).get();
}
```
You can also reverse the order by setting the `mode` property of the `OrderingTerm` to
`OrderingMode.desc`.

### Single values
If you know a query is never going to return more than one row, wrapping the result in a `List`
can be tedious. Moor lets you work around that with `getSingle` and `watchSingle`:
```dart
Stream<Todo> entryById(int id) {
  return (select(todos)..where((t) => t.id.equals(id))).watchSingle();
}
```
If an entry with the provided id exists, it will be sent to the stream. Otherwise,
`null` will be added to stream. If a query used with `watchSingle` ever returns
more than one entry (which is impossible in this case), an error will be added
instead.

### Mapping

Before calling `watch` or `get` (or the single variants), you can use `map` to transform
the result. 
```dart
Stream<List<String>> contentWithLongTitles() {
  final query = select(todos)
    ..where((t) => t.title.length.isBiggerOrEqualValue(16));

  return query
    .map((row) => row.content)
    .watch();
}
```


If you need more complex queries with joins or custom columns, see [this site]({{< relref "../Advanced Features/joins.md" >}}).

## Updates and deletes
You can use the generated classes to update individual fields of any row:
```dart
Future moveImportantTasksIntoCategory(Category target) {
  // for updates, we use the "companion" version of a generated class. This wraps the
  // fields in a "Value" type which can be set to be absent using "Value.absent()". This
  // allows us to separate between "SET category = NULL" (`category: Value(null)`) and not
  // updating the category at all: `category: Value.absent()`.
  return (update(todos)
      ..where((t) => t.title.like('%Important%'))
    ).write(TodosCompanion(
      category: Value(target.id),
    ),
  );
}

Future update(Todo entry) {
  // using replace will update all fields from the entry that are not marked as a primary key.
  // it will also make sure that only the entry with the same primary key will be updated.
  // Here, this means that the row that has the same id as entry will be updated to reflect
  // the entry's title, content and category. As it set's its where clause automatically, it
  // can not be used together with where.
  return update(todos).replace(entry);
}

Future feelingLazy() {
  // delete the oldest nine tasks
  return (delete(todos)..where((t) => t.id.isSmallerThanValue(10))).go();
}
```
__⚠️ Caution:__ If you don't explicitly add a `where` clause on updates or deletes, 
the statement will affect all rows in the table!

{{% alert title="Entries, companions - why do we need all of this?"  %}}
You might have noticed that we used a `TodosCompanion` for the first update instead of
just passing a `Todo`. Moor generates the `Todo` class (also called _data
class_ for the table) to hold a __full__ row with all its data. For _partial_ data,
prefer to use companions. In the example above, we only set the the `category` column,
so we used a companion. 
Why is that necessary? If a field was set to `null`, we wouldn't know whether we need 
to set that column back to null in the database or if we should just leave it unchanged.
Fields in the companions have a special `Value.absent()` state which makes this explicit.

Companions also have a special constructor for inserts - all columns which don't have
a default value and aren't nullable are marked `@required` on that constructor. This makes
companions easier to use for inserts because you know which fields to set.
{{% /alert %}}

## Inserts
You can very easily insert any valid object into tables. As some values can be absent
(like default values that we don't have to set explicitly), we again use the 
companion version.
```dart
// returns the generated id
Future<int> addTodo(TodosCompanion entry) {
  return into(todos).insert(entry);
}
```
All row classes generated will have a constructor that can be used to create objects:
```dart
addTodo(
  TodosCompanion(
    title: Value('Important task'),
    content: Value('Refactor persistence code'),
  ),
);
```
If a column is nullable or has a default value (this includes auto-increments), the field
can be omitted. All other fields must be set and non-null. The `insert` method will throw
otherwise.

Multiple insert statements can be run efficiently by using a batch. To do that, you can
use the `insertAll` method inside a `batch`:
```dart
Future<void> insertMultipleEntries() async{
  await batch((batch) {
    // functions in a batch don't have to be awaited - just
    // await the whole batch afterwards.
    batch.insertAll(todos, [
      TodosCompanion.insert(
        title: 'First entry',
        content: 'My content',
      ),
      TodosCompanion.insert(
        title: 'Another entry',
        content: 'More content',
      ),
      // ...
    ]);
  });
}
```