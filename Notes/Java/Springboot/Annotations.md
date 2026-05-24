## Lombok

### `@Getter`
Generates getter methods automatically.

### `@Setter`
Generates setter methods automatically.

### @Data
Shortcut annotation.
Equivalent to combining:
```
@Getter
@Setter
@ToString
@EqualsAndHashCode
@RequiredArgsConstructor
```

### ==Important Practical Point

Many developers blindly use `@Data` on JPA entities. That can cause problems because:
- `equals/hashCode` may trigger lazy loading
- `toString()` may recursively print relationships
- Hibernate proxies can behave unexpectedly

For JPA entities, many teams prefer instead of `@Data`.
```@Getter
@Setter
@NoArgsConstructor
@AllArgsConstructor
```


## JPA / Hibernate Annotations
From [Hibernate ORM](https://hibernate.org/orm/?utm_source=chatgpt.com) and [Jakarta Persistence Specification](https://jakarta.ee/specifications/persistence/?utm_source=chatgpt.com).

### `@Entity`

Marks a class as a database entity.
Meaning:
> "This Java class should map to a database table."
```
@Entity
class User {
    private Long id;
    private String name;
}
```
Hibernate/JPA now treats `User` as a table-mapped object.
Without `@Entity`, JPA ignores the class completely.

### `@Table`

Specifies the actual database table name.
```
@Entity
@Table(name = "users")
class User {
}
```
Maps class `User` → table `users`.
Without `@Table`, default behavior is usually:

### ==Very Important Difference

| Thing       | Used In          | Example                |
| ----------- | ---------------- | ---------------------- |
| Entity name | JPA/JPQL queries | `SELECT u FROM User u` |
| Table name  | Actual database  | `SELECT * FROM users`  |

### @MappedSuperclass
`@MappedSuperclass` is a JPA annotation used for **inheritance of common fields**, but the superclass itself is **NOT a database table**.

A `@MappedSuperclass`:
- cannot have repositories
- cannot be directly persisted
- cannot be target of relationships
#### Summary

| Annotation          | Has Table | Queryable | Used For                     |
| ------------------- | --------- | --------- | ---------------------------- |
| `@Entity`           | Yes       | Yes       | Actual DB entity             |
| `@MappedSuperclass` | No        | No        | Sharing common mapped fields |

### @Id
Marks the primary key.
```
@Id
private UUID id;
```


### @GeneratedValue
Generate the primary key automatically.

#### Generation Strategies

| Strategy   | Example           |
| ---------- | ----------------- |
| `IDENTITY` | Auto increment    |
| `SEQUENCE` | DB sequence       |
| `AUTO`     | Hibernate decides |
| `UUID`     | UUID generation   |


### @Column

|Property|Purpose|Example|Effect|
|---|---|---|---|
|`name`|Custom DB column name|`@Column(name="user_name")`|Maps field to specific column|
|`nullable`|Allow null or not|`nullable = false`|Creates `NOT NULL`|
|`unique`|Prevent duplicate values|`unique = true`|Creates `UNIQUE` constraint|
|`length`|Max string length|`length = 50`|Creates `VARCHAR(50)`|
|`precision`|Total digits for decimal|`precision = 10`|Used for `DECIMAL`|
|`scale`|Digits after decimal|`scale = 2`|`DECIMAL(10,2)`|
|`updatable`|Allow updates or not|`updatable = false`|Field immutable after insert|
|`insertable`|Include in INSERT query|`insertable = false`|Hibernate skips during insert|
|`columnDefinition`|Custom SQL definition|`columnDefinition="TEXT"`|Direct DB-specific SQL|
|`table`|Map field to another table|`table="audit_table"`|Used in secondary tables|
#### `columnDefinition`
Used to define the **exact SQL column type or SQL definition manually** instead of letting Hibernate choose it.
#### Example
```
@Column(columnDefinition = "TEXT")private String description;
```
#### Common Uses
- `TEXT`
- `JSONB`
- default values
- DB-specific types
### Key Point
Gives direct control over generated SQL column definition.

#### `table`
Used to store a field in a **different table** than the main entity table.
Requires `@SecondaryTable`.
### Example
```
@Column(table = "user_details")private String address;
```
### Key Point
Maps a field to another database table while keeping the same Java entity.

### @SecondaryTable
Used to map **one entity class to multiple database tables**.
Normally:
- one entity → one table
With `@SecondaryTable`:
- one entity → multiple tables

```
@Entity
@Table(name = "users")
@SecondaryTable(
    name = "user_details",
    pkJoinColumns = @PrimaryKeyJoinColumn(name = "user_id")
)
public class User {

    @Id
    private Long id;

    private String name;

    @Column(table = "user_details")
    private String address;
}
```
Used For 
- splitting large tables
- optional data
- legacy DB schemas
- performance optimization
- normalization
#### pkJoinColumns
Defines how both tables are connected.

```
@SecondaryTable:
Maps one entity to multiple database tables.
Used with @Column(table="...").
Tables are joined using primary key mapping.
```


### `@CreationTimestamp`
Automatically sets the timestamp when the entity is first created.
### `@UpdateTimestamp`
Automatically updates timestamp whenever entity changes.



## Jackson Annotation

### @JsonInclude(JsonInclude.Include.NON_EMPTY)

`@JsonInclude` is a Jackson serialization annotation.
it controls which fields should appear in JSON response

Example
```
@JsonInclude(JsonInclude.Include.NON_EMPTY)
private List<FieldValidationError> errors;
```
> “Only include `errors` in JSON if it is NOT empty.”

# What Counts As “Empty”
`NON_EMPTY` excludes:

| Type   | Excluded When |
| ------ | ------------- |
| List   | empty         |
| String | `""`          |
| Map    | empty         |
| Array  | empty         |
| null   | null          |

#### Different methods
`Include.ALWAYS` --> Default behavior. -> Everything serialized.
`Include.NON_NULL` --> Excludes only null
`Include.NON_DEFAULT` -- > Excludes fields having default values.


## Security Annotations[[Spring Security & JWT]]

## Validation Annotations [[Validation Annotations]]


