# Setup

Models are YAML files used to define serializable classes in Serverpod. They are used to generate Dart code for the server and client, and, if a database table is defined, to generate database code for the server.

Using regular `.yaml` files within `lib/src/models` is supported, but it is recommended to use `.spy.yaml` (.spy stands for "Serverpod YAML"). Using this file type allows placing the model files anywhere in your servers `lib` directory and enables syntax highlighting provided by the [Serverpod Extension](https://marketplace.visualstudio.com/items?itemName=serverpod.serverpod) for VS Code.

The files are analyzed by the Serverpod CLI when generating the project and creating migrations.

Run `serverpod generate` to generate dart classes from the model files.

## Class

```yaml
class: Company
fields:
  name: String
  foundedDate: DateTime?
  employees: List<Employee>
```

Supported types are [bool](https://api.dart.dev/dart-core/bool-class.html), [int](https://api.dart.dev/dart-core/int-class.html), [double](https://api.dart.dev/dart-core/double-class.html), [String](https://api.dart.dev/dart-core/String-class.html), [Duration](https://api.dart.dev/dart-core/Duration-class.html), [DateTime](https://api.dart.dev/dart-core/DateTime-class.html), [ByteData](https://api.dart.dev/dart-typed_data/ByteData-class.html), [UuidValue](https://pub.dev/documentation/uuid/latest/uuid_value/UuidValue-class.html), [Uri](https://api.dart.dev/dart-core/Uri-class.html), [BigInt](https://api.dart.dev/dart-core/BigInt-class.html), [Vector](05-advanced#vector), [HalfVector](05-advanced#halfvector), [SparseVector](05-advanced#sparsevector), [Bit](05-advanced#bit) and other serializable [classes](02-classes), [exceptions](04-enums-and-exceptions#exception) and [enums](04-enums-and-exceptions#enum). You can also use [List](https://api.dart.dev/dart-core/List-class.html)s, [Map](https://api.dart.dev/dart-core/Map-class.html)s and [Set](https://api.dart.dev/dart-core/Set-class.html)s of the supported types, just make sure to specify the types. All supported types can also be used inside [Record](https://api.dart.dev/dart-core/Record-class.html)s. Null safety is supported. Once your classes are generated, you can use them as parameters or return types to endpoint methods.

### Required fields

Nullable types are supported by adding a `?` after the type. E.g., `String?` or `List<Employee>?`. The optional `required` keyword makes the generated field a required constructor parameter.

```yaml
class: Person
fields:
  name: String
  nickname: String?, required
  age: int?
```

In the example above, `nickname` will be a required constructor parameter.

:::info
Serverpod's models can easily be saved to or read from the database. You can read more about this in the [Database](../06-database/02-models) section.
:::
