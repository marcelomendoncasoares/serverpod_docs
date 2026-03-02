# Advanced topics

## Adding documentation

Serverpod allows you to add documentation to your serializable objects in a similar way that you would add documentation to your Dart code. Use three hashes (###) to indicate that a comment should be considered documentation.

```yaml
### Information about a company.
class: Company
fields:
  ### The name of the company.
  name: String

  ### The date the company was founded, if known.
  foundedDate: DateTime?

  ### A list of people currently employed at the company.
  employees: List<Employee>
```

## Vector fields

Vector types are used for storing high-dimensional vectors, which are especially useful for similarity search operations.

When specifying vector types, the dimension is required between parentheses (e.g., `Vector(1536)`). Common dimensions include:

- 1536 (OpenAI embeddings)
- 768 (many sentence transformers)
- 384 (smaller models)

All vector types support specialized distance operations for similarity search and filtering. See the [Vector distance operators](../06-database/06-filter#vector-distance-operators) section for details.

To ensure optimal performance with vector similarity searches, consider creating specialized vector indexes on your vector fields. See the [Vector indexes](../06-database/04-indexing#vector-indexes) section for more details.

:::info
The usage of Vector fields requires the pgvector PostgreSQL extension to be installed, which comes by default on new Serverpod projects. To upgrade an existing project, see the [Upgrading to pgvector support](../../08-upgrading/03-upgrade-to-pgvector) guide.
:::

### Vector

The `Vector` type stores full-precision floating-point vectors for general-purpose embeddings.

```yaml
class: Document
table: document
fields:
  ### The category of the document (e.g., article, tutorial).
  category: String

  ### The contents of the document.
  content: String

  ### A vector field for storing document embeddings
  embedding: Vector(1536)
```

### HalfVector

The `HalfVector` type uses half-precision (16-bit) floating-point numbers, providing memory savings with acceptable precision loss for several applications.

```yaml
class: Document
table: document
fields:
  content: String
  ### Half-precision embedding for memory efficiency
  embedding: HalfVector(1536)
```

### SparseVector

The `SparseVector` type efficiently stores sparse vectors where most values are zero, which is ideal for high-dimensional data with few non-zero elements.

```yaml
class: Document
table: document
fields:
  content: String
  ### Sparse vector for keyword-based embeddings
  keywords: SparseVector(10000)
```

### Bit

The `Bit` type stores binary vectors where each element is 0 or 1, offering maximum memory efficiency for binary embeddings.

```yaml
class: Document
table: document
fields:
  content: String
  ### Binary vector for semantic hashing
  hash: Bit(256)
```

## Generated code

Serverpod generates some convenience methods on the Dart classes.

### copyWith

The `copyWith` method allows for efficient object copying with selective field updates and is available on all generated classes. Here's how it operates:

```dart
var john = User(name: 'John Doe', age: 25);
var jane = john.copyWith(name: 'Jane Doe');
```

The `copyWith` method generates a deep copy of an object, preserving all original fields unless explicitly modified. It can distinguish between a field set to `null` and a field left unspecified (undefined). When using `copyWith`, any field you don't update remains unchanged in the new object.

### toJson / fromJson

The `toJson` and `fromJson` methods are generated on all models to help with serialization. Serverpod manages all serialization for you out of the box and you will rarely have to use these methods by your self. See the [Serialization](06-serialization) section for more info.

### Custom methods

Sometimes you will want to add custom methods to the generated classes. The easiest way to do this is with [Dart's extension feature](https://dart.dev/language/extension-methods).

```dart
extension MyExtension on MyClass {
  bool isCustomMethod() {
    return true;
  }
}
```

## Default Values

Serverpod supports defining default values for fields in your models. These default values can be specified using three different keywords that determine how and where the defaults are applied:

### Keywords

- **default**: This keyword sets a default value for both the model (code) and the database (persisted data). It acts as a general fallback if more specific defaults aren't provided.
- **defaultModel**: This keyword sets a default value specifically for the model (the code side). If `defaultModel` is not provided, the model will use the value specified by `default` if it's available.
- **defaultPersist**: This keyword sets a default value specifically for the database. If `defaultPersist` is not provided, the database will use the value specified by `default` if it's available.

### How priorities work

- **For the model (code side):** If both `defaultModel` and `default` are provided, the model will use the `defaultModel` value. If `defaultModel` is not provided, it will fall back to using the `default` value.
- **For the database (persisted data):** If both `defaultPersist` and `default` are provided, the database will use the `defaultPersist` value. If `defaultPersist` is not provided, it will fall back to using the `default` value.

You can use these default values individually or in combination as needed. It is not required to use all default types for a field.

:::info

When using `default` or `defaultModel` in combination with `defaultPersist`, it's important to understand how the interaction between these keywords affects the final value in the database.

If you set a `default` or `defaultModel` value, the model's field or variable will have a value when it's passed to the database—it will not be `null`. Because of this, the SQL query will not use the `defaultPersist` value since the field already has a value assigned by the model. In essence, assigning a `default` or `defaultModel` is like directly providing a value to the field, and the database will use this provided value instead of its own default.

This means that `defaultPersist` only comes into play when the model does not provide a value, allowing the database to apply its own default setting.

:::

### Supported default values

#### Boolean

| Type        | Keyword           | Description                                                  |
| ----------- | ----------------- | ------------------------------------------------------------ |
| **Boolean** | `true` or `false` | Sets the field to a boolean value, either `true` or `false`. |

**Example:**

```yaml
boolDefault: bool, default=true
```

#### DateTime

| Type                      | Keyword                                                          | Description                                  |
| ------------------------- | ---------------------------------------------------------------- | -------------------------------------------- |
| **Current Date and Time** | `now`                                                            | Sets the field to the current date and time. |
| **Specific UTC DateTime** | UTC DateTime string in the format `yyyy-MM-dd'T'HH:mm:ss.SSS'Z'` | Sets the field to a specific date and time.  |

**Example:**

```yaml
dateTimeDefaultNow: DateTime, default=now
dateTimeDefaultUtc: DateTime, default=2024-05-01T22:00:00.000Z
```

#### Double

| Type       | Keyword          | Description                                |
| ---------- | ---------------- | ------------------------------------------ |
| **Double** | Any double value | Sets the field to a specific double value. |

**Example:**

```yaml
doubleDefault: double, default=10.5
```

#### Duration

| Type                  | Keyword                                            | Description                                                                                                                                                |
| --------------------- | -------------------------------------------------- | ---------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Specific Duration** | A valid duration in the format `Xd Xh Xmin Xs Xms` | Sets the field to a specific duration value. For example, `1d 2h 10min 30s 100ms` represents 1 day, 2 hours, 10 minutes, 30 seconds, and 100 milliseconds. |

**Example:**

```yaml
durationDefault: Duration, default=1d 2h 10min 30s 100ms
```

#### Enum

| Type     | Keyword              | Description                              |
| -------- | -------------------- | ---------------------------------------- |
| **Enum** | Any valid enum value | Sets the field to a specific enum value. |

**Example:**

```yaml
enum: ByNameEnum
serialized: byName
values:
  - byName1
  - byName2
```

```yaml
enum: ByIndexEnum
serialized: byIndex
values:
  - byIndex1
  - byIndex2
```

```yaml
class: EnumDefault
table: enum_default
fields:
  byNameEnumDefault: ByNameEnum, default=byName1
  byIndexEnumDefault: ByIndexEnum, default=byIndex1
```

In this example:

- The `byNameEnumDefault` field will default to `'byName1'` in the database.
- The `byIndexEnumDefault` field will default to `0` (the index of `byIndex1`).

#### Integer

| Type        | Keyword           | Description                                 |
| ----------- | ----------------- | ------------------------------------------- |
| **Integer** | Any integer value | Sets the field to a specific integer value. |

**Example:**

```yaml
intDefault: int, default=10
```

#### String

| Type       | Keyword          | Description                                |
| ---------- | ---------------- | ------------------------------------------ |
| **String** | Any string value | Sets the field to a specific string value. |

**Example:**

```yaml
stringDefault: String, default='This is a string'
```

#### UuidValue

| Type              | Keyword                                                                                                                                                                                                   | Description                                                                                                                                       |
| ----------------- | --------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------------------------------------------- |
| **Random UUID**   | `random`                                                                                                                                                                                                  | Generates a random UUID. On the Dart side, `Uuid().v4obj()` is used. On the database side, `gen_random_uuid()` is used.                           |
| **Random UUIDv7** | `random_v7`                                                                                                                                                                                               | Generates a random UUIDv7. On the Dart side, `Uuid().v7obj()` is used. On the database side, a generated `gen_random_uuid_v7()` function is used. |
| **UUID String**   | Valid UUID in the format 'xxxxxxxx-xxxx-Mxxx-Nxxx-xxxxxxxxxxxx' where M is the UUID version field. The upper two or three bits of digit N encode the variant. E.g. '550e8400-e29b-41d4-a716-446655440000' | Assigns a specific UUID to the field.                                                                                                             |

**Example:**

```yaml
uuidDefaultRandom: UuidValue, default=random
uuidDefaultUuid: UuidValue, default='550e8400-e29b-41d4-a716-446655440000'
uuidDefaultRandomUuidV7: UuidValue, default=random_v7
```

### Example

```yaml
class: DefaultValue
table: default_value
fields:
  ### Sets the current date and time as the default value.
  dateTimeDefault: DateTime, default=now

  ### Sets the default value for a boolean field.
  boolDefault: bool, defaultModel=false, defaultPersist=true

  ### Sets the default value for an integer field.
  intDefault: int, defaultPersist=20

  ### Sets the default value for a double field.
  doubleDefault: double, default=10.5, defaultPersist=20.5

  ### Sets the default value for a string field.
  stringDefault: String, default="This is a string", defaultModel="This is a string"
```

## Keywords

| **Keyword**                                                         | Note                                                                                                          | [class](01-setup#class) | [exception](04-enums-and-exceptions#exception) | [enum](04-enums-and-exceptions#enum) |
| ------------------------------------------------------------------- | ------------------------------------------------------------------------------------------------------------- | :-------------: | :---------------------: | :-----------: |
| [**values**](04-enums-and-exceptions#enum)                           | A special key for enums with a list of all enum values.                                                       |                 |                         |      ✅       |
| [**serialized**](04-enums-and-exceptions#enum)                       | Sets the mode enums are serialized in                                                                         |                 |                         |      ✅       |
| [**properties**](04-enums-and-exceptions#enhanced-enums-with-properties) | Defines custom properties for enhanced enums.                                                                 |                 |                         |      ✅       |
| [**immutable**](02-classes#immutable-classes)                        | Boolean flag to generate an immutable class with final fields, equals operator, and hashCode.                 |       ✅        |           ✅            |               |
| [**serverOnly**](02-classes#limiting-visibility-of-a-generated-class) | Boolean flag if code generator only should create the code for the server.                                    |       ✅        |           ✅            |      ✅       |
| [**table**](../06-database/02-models)                                | A name for the database table, enables generation of database code.                                           |       ✅        |                         |               |
| [**managedMigration**](../06-database/11-migrations#opt-out-of-migrations)   | A boolean flag to opt out of the database migration system.                                                   |       ✅        |                         |               |
| [**fields**](01-setup#class)                                        | All fields in the generated class should be listed here.                                                      |       ✅        |           ✅            |               |
| [**type (fields)**](01-setup#class)                                 | Denotes the data type for a field.                                                                            |       ✅        |           ✅            |               |
| [**required**](01-setup#required-fields)                             | Makes the field as required. This keyword can only be used for **nullable** fields.                           |       ✅        |           ✅            |               |
| [**scope**](02-classes#limiting-visibility-of-a-generated-class)     | Denotes the scope for a field.                                                                                |       ✅        |                         |               |
| [**jsonKey**](02-classes#json-key-aliasing)                           | Sets a custom key name for JSON serialization and deserialization.                                            |       ✅        |                         |               |
| [**persist**](../06-database/02-models)                              | A boolean flag if the data should be stored in the database or not can be negated with `!persist`             |       ✅        |                         |               |
| [**relation**](../06-database/03-relations/01-one-to-one)             | Sets a relation between model files, requires a table name to be set.                                         |       ✅        |                         |               |
| [**name**](../06-database/03-relations/01-one-to-one#bidirectional-relations)   | Give a name to a relation to pair them.                                                                       |       ✅        |                         |               |
| [**parent**](../06-database/03-relations/01-one-to-one#with-an-id-field) | Sets the parent table on a relation.                                                                          |       ✅        |                         |               |
| [**field**](../06-database/03-relations/01-one-to-one#custom-foreign-key-field) | A manual specified foreign key field.                                                                         |       ✅        |                         |               |
| [**onUpdate**](../06-database/03-relations/05-referential-actions)   | Set the referential actions when updating data in the database.                                               |       ✅        |                         |               |
| [**onDelete**](../06-database/03-relations/05-referential-actions)   | Set the referential actions when deleting data in the database.                                               |       ✅        |                         |               |
| [**optional**](../06-database/03-relations/01-one-to-one#optional-relation)     | A boolean flag to make a relation optional.                                                                   |       ✅        |                         |               |
| [**indexes**](../06-database/04-indexing)                             | Create indexes on your fields / columns.                                                                      |       ✅        |                         |               |
| [**fields (index)**](../06-database/04-indexing)                     | List the fields to create the indexes on.                                                                     |       ✅        |                         |               |
| [**type (index)**](../06-database/04-indexing)                        | The type of index to create.                                                                                  |       ✅        |                         |               |
| [**parameters (index)**](../06-database/04-indexing#vector-indexes)  | Parameters for specialized index types like HNSW and IVFFLAT vector indexes.                                  |       ✅        |                         |               |
| [**distanceFunction (index)**](../06-database/04-indexing#vector-indexes)    | Distance function for vector indexes (l2, innerProduct, cosine, l1).                                          |       ✅        |                         |               |
| [**unique**](../06-database/04-indexing)                             | Boolean flag to make the entries unique in the database.                                                      |       ✅        |                         |               |
| [**default**](#default-values)                                       | Sets the default value for both the model and the database. This keyword cannot be used with **relation**.    |       ✅        |                         |               |
| [**defaultModel**](#default-values)                                  | Sets the default value for the model side. This keyword cannot be used with **relation**.                     |       ✅        |                         |               |
| [**defaultPersist**](#default-values)                                | Sets the default value for the database side. This keyword cannot be used with **relation** and **!persist**. |       ✅        |                         |               |
| [**extends**](03-inheritance#extending-a-class)                      | Specifies a parent class to inherit from.                                                                     |       ✅        |           ✅            |               |
| [**sealed**](03-inheritance#sealed-classes)                          | Boolean flag to create a sealed class hierarchy, enabling exhaustive type checking.                           |       ✅        |                         |               |
