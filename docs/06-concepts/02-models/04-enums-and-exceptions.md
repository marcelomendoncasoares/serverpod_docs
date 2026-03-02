# Enums and exceptions

## Exception

The Serverpod models supports creating exceptions that can be thrown in endpoints by using the `exception` keyword. For more in-depth description on how to work with exceptions see [Error handling and exceptions](../04-exceptions).

```yaml
exception: MyException
fields:
  message: String
  errorType: MyEnum
```

## Enum

It is easy to add custom enums with serialization support by using the `enum` keyword.

```yaml
enum: Animal
values:
  - dog
  - cat
  - bird
```

By default the serialization will convert the enum to an int representing the index of the value. Changing the order may therefore have unforeseen consequences when reusing old data (such as from a database). Changing the serialization to be based on the name instead of index is easy.

```yaml
enum: Animal
serialized: byName
values:
  - dog
  - cat
  - bird
```

`serialized` has two valid values `byName` and `byIndex`. When using `byName` the string literal of the enum is used, when using `byIndex` the index value (0, 1, 2, etc) is used.

:::info

It's recommended to always set `serialized` to `byName` in any new Enum models, as this is less fragile and will be changed to the default setting in version 3 of Serverpod.

:::

### Default value

A default value is used when an unknown value is deserialized. This can happen, for example, if a new enum option is added and older clients receive it from the server, or if an enum option is removed but the database still contains the old value.

To configure a default value, use the `default` keyword.

```yaml
enum: Animal
serialized: byName
default: unknown
values:
  - unknown
  - dog
  - cat
  - bird
```

In the example above, if the Enum `Animal` receives an unknown option such as `"fish"` it will be deserialized to `Animal.unknown`. This is useful for maintaining backward compatibility when changing the enum values.

:::warning
If no default value is specified, deserialization of unknown values will throw an exception. Adding a default value prevents these exceptions, but may also hide real issues in your data. Use this feature with caution.
:::

### Enhanced enums with properties

Serverpod supports enhanced enums that can have custom properties attached to each value. This is useful when you need to associate additional data with enum values, such as display names, codes, or configuration values.

To define an enhanced enum, add a `properties` section that declares the property names and types, then specify property values for each enum value:

```yaml
enum: HttpStatus
serialized: byName
properties:
  code: int
  message: String
values:
  - ok:
      code: 200
      message: 'OK'
  - notFound:
      code: 404
      message: 'Not Found'
  - internalError:
      code: 500
      message: 'Internal Server Error'
```

This generates an enhanced Dart enum with the specified properties:

```dart
enum HttpStatus implements SerializableModel {
  ok(200, 'OK'),
  notFound(404, 'Not Found'),
  internalError(500, 'Internal Server Error');

  const HttpStatus(this.code, this.message);

  final int code;
  final String message;

  // Serialization methods...
}
```

You can then access the properties on any enum value:

```dart
var status = HttpStatus.ok;
print(status.code);    // 200
print(status.message); // OK
```

#### Supported property types

Enhanced enum properties support the following types:

- `int` / `int?`
- `double` / `double?`
- `bool` / `bool?`
- `String` / `String?`

#### Default property values

Properties can have default values, making them optional when defining enum values. The syntax follows the same pattern as [class field defaults](05-advanced#default-values):

```yaml
enum: Priority
properties:
  level: int
  description: String, default='No description'
values:
  - low:
      level: 1
  - medium:
      level: 2
      description: 'Medium priority'
  - high:
      level: 3
      description: 'High priority - handle first'
```

In this example, `low` will use the default description `'No description'`, while `medium` and `high` have explicit descriptions.
