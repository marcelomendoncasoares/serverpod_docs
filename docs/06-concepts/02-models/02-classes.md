# Class features

## Limiting visibility of a generated class

By default, generated code for your serializable objects is available both on the server and the client. You may want to have the code on the server side only. E.g., if the serializable object is connected to a database table containing private information.

To make a serializable class generated only on the server side, set the serverOnly property to true.

```yaml
class: MyPrivateClass
serverOnly: true
fields:
  hiddenSecretKey: String
```

It is also possible to set a `scope` on a per-field basis. By default all fields are visible to both the server and the client. The available scopes are `all`, `serverOnly`, `none`.

:::info
**none** is not typically used in serverpod apps. It is intended for the serverpod framework, itself.
:::

```yaml
class: SelectivelyHiddenClass
fields:
  hiddenSecretKey: String, scope=serverOnly
  publicKey: String
```

:::info
Serverpod's models can easily be saved to or read from the database. You can read more about this in the [Database](../06-database/02-models) section.
:::

## JSON key aliasing

By default, fields are serialized to JSON using their Dart field name as the key. The `jsonKey` property allows you to specify a different key name for JSON serialization and deserialization, which is useful when integrating with external APIs that use different naming conventions.

```yaml
class: User
fields:
  displayName: String, jsonKey=display_name
  emailAddress: String, jsonKey=email
  createdAt: DateTime, jsonKey=created_at
```

This generates a class where the Dart field names remain camelCase, but the JSON representation uses the specified keys:

```dart
// Dart field names
var user = User(
  displayName: 'John Doe',
  emailAddress: 'john@example.com',
  createdAt: DateTime.parse('2024-01-15T10:30:00.000Z'),
);

// Serializes to JSON with custom keys
// {
//   "display_name": "John Doe",
//   "email": "john@example.com",
//   "created_at": "2024-01-15T10:30:00.000Z"
// }
```

This is particularly helpful when:

- Consuming external APIs that use snake_case or other naming conventions
- Working with legacy systems that have specific JSON field requirements
- Integrating with third-party services like MongoDB (e.g., mapping `id` to `_id`)

:::info
The `jsonKey` property affects JSON serialization and deserialization. It does not affect the database column name. To customize the database column name, use the `column` property instead. This is an experimental feature; see the [Experimental documentation](../21-experimental#column-name-override) under *Column name override* for details.
:::

## Immutable classes

By default, generated classes in Serverpod are mutable, meaning their fields can be changed after creation. However, you can make a class immutable by setting the `immutable` property to `true`. Immutable classes are especially useful when working with state management solutions or when you need value-based equality.

```yaml
class: ImmutableUser
immutable: true
fields:
  name: String
  email: String
```

When you mark a class as immutable:

1. **All fields become final**: Fields cannot be reassigned after the object is created
2. **Generates `operator ==`**: Provides deep equality comparison between instances
3. **Generates `hashCode`**: Ensures instances with the same values have the same hash code
4. **Compatible with `copyWith`**: You can still create modified copies of immutable objects using the `copyWith` method

Example usage:

```dart
var user1 = ImmutableUser(name: 'Alice', email: 'alice@example.com');
var user2 = ImmutableUser(name: 'Alice', email: 'alice@example.com');

// Equality comparison works based on values
print(user1 == user2); // true

// Fields are final and cannot be reassigned
// user1.name = 'Bob'; // This would cause a compile error

// Use copyWith to create modified copies
var user3 = user1.copyWith(name: 'Bob');
print(user3.name); // Bob
print(user3.email); // alice@example.com
```
