---
title: Schema Definition
lang: en
related_path: zh-cn/schema-definition.html
---

# Schema Definition

To use SQL DSL, we need to let Ktorm know our schemas. Assuming we have two tables, `t_department` and `t_employee`, their schemas are given in the SQL below, how should we describe these two tables with Ktorm?

```sql
create table t_department(
  id int not null primary key auto_increment,
  name varchar(128) not null,
  location varchar(128) not null
);

create table t_employee(
  id int not null primary key auto_increment,
  name varchar(128) not null,
  job varchar(128) not null,
  manager_id int null,
  hire_date date not null,
  salary bigint not null,
  department_id int not null
);
```

## Table Objects

Generally, we can define Kotlin objects extending `Table` to describe our table schemas in Ktorm. The following code defines the two tables with Ktorm: 

```kotlin
object Departments : Table<Nothing>("t_department") {
    val id = int("id").primaryKey()
    val name = varchar("name")
    val location = varchar("location")
}

object Employees : Table<Nothing>("t_employee") {
    val id = int("id").primaryKey()
    val name = varchar("name")
    val job = varchar("job")
    val managerId = int("manager_id")
    val hireDate = date("hire_date")
    val salary = long("salary")
    val departmentId = int("department_id")
}
```

We can see that both `Departments` and `Employees` are extending from `Table` whose constructor accepts a table name as the parameter. There is also a generic type parameter for `Table` class, that is the entity class's type that current table is binding to. Here we don't bind to any entity classes, so `Nothing` is OK. 

Columns are defined as properties in table objects by Kotlin's *val* keyword, their types are defined by type definition functions, such as int, long, varchar, date, etc. Commonly, these type definition functions follow the rules below:  

- They are all `Table` class's extension functions that are only allowed to be used in table object definitions. 
- Their names are corresponding to the underlying SQL types' names. 
- They all accept a parameter of string type, that is the column's name.
- Their return types are `Column<C>`, in which C is the type of current column. We can chaining call the extension function `primaryKey` to declare the current column as a primary key. 

In general, we define tables as Kotlin singleton objects, but we don't really have to stop there. For instance, assuming that we have two tables that are totally the same, they have the same columns, but their names are different. In this special case, do we have to copy the same column definitions to each table? No, we don't. We can reuse our codes by subclassing: 

```kotlin
sealed class Employees(tableName: String) : Table<Nothing>(tableName) {
    val id = int("id").primaryKey()
    val name = varchar("name")
    val job = varchar("job")
    val managerId = int("manager_id")
    val hireDate = date("hire_date")
    val salary = long("salary")
    val departmentId = int("department_id")
}

object RegularEmployees : Employees("t_regular_employee")

object FormerEmployees : Employees("t_former_employee")
```

For another example, sometimes our table is one-off, we don't need to use it twice, so it's not necessary to define it as a global object, for fear that the naming space is polluted. This time, we can even define the table as an anonymous object inside a function: 

```kotlin
val t = object : Table<Nothing>("t_config") {
    val key = varchar("key").primaryKey()
    val value = varchar("value")
}

// Get all configs as a Map<String, String>
val configs = database.from(t).select().associate { row -> row[t.key] to row[t.value] }
```

Flexible usage of Kotlin's language features is helpful for us to reduce duplicated code and improve the maintainability of our projects. 

## SqlType

`SqlType` is an abstract class which provides a unified abstraction for data types in SQL. Based on JDBC, it encapsulates the common operations of obtaining data from a `ResultSet` and setting parameters to a `PreparedStatement`. In the section above, we defined columns by column definition functions, eg. int, varchar, etc. All these functions have a `SqlType` implementation behind them. For example, here is the implementation of `int` function: 

```kotlin
fun BaseTable<*>.int(name: String): Column<Int> {
    return registerColumn(name, IntSqlType)
}

object IntSqlType : SqlType<Int>(Types.INTEGER, typeName = "int") {
    override fun doSetParameter(ps: PreparedStatement, index: Int, parameter: Int) {
        ps.setInt(index, parameter)
    }

    override fun doGetResult(rs: ResultSet, index: Int): Int? {
        return rs.getInt(index)
    }
}
```

`IntSqlType` is simple, it just obtaining int query results via `ResultSet.getInt` and setting parameters via `PreparedStatement.setInt`. 

Here is a list of SQL types supported in Ktorm by default: 

| Function Name | Kotlin Type             | Underlying SQL Type | JDBC Type Code (java.sql.Types) |
| ------------- | ----------------------- | ------------- | ---------------------------- |
| boolean       | kotlin.Boolean          | boolean       | Types.BOOLEAN                |
| int           | kotlin.Int              | int           | Types.INTEGER                |
| short         | kotlin.Short            | smallint      | Types.SMALLINT               |
| long          | kotlin.Long             | bigint        | Types.BIGINT                 |
| float         | kotlin.Float            | float         | Types.FLOAT                  |
| double        | kotlin.Double           | double        | Types.DOUBLE                 |
| decimal       | java.math.BigDecimal    | decimal       | Types.DECIMAL                |
| varchar       | kotlin.String           | varchar       | Types.VARCHAR                |
| text          | kotlin.String           | text          | Types.LONGVARCHAR            |
| blob          | kotlin.ByteArray        | blob          | Types.BLOB                   |
| bytes         | kotlin.ByteArray        | bytes         | Types.BINARY                 |
| jdbcTimestamp | java.sql.Timestamp      | timestamp     | Types.TIMESTAMP              |
| jdbcDate      | java.sql.Date           | date          | Types.DATE                   |
| jdbcTime      | java.sql.Time           | time          | Types.TIME                   |
| timestamp     | java.time.Instant       | timestamp     | Types.TIMESTAMP              |
| datetime      | java.time.LocalDateTime | datetime      | Types.TIMESTAMP              |
| date          | java.time.LocalDate     | date          | Types.DATE                   |
| time          | java.time.Time          | time          | Types.TIME                   |
| monthDay      | java.time.MonthDay      | varchar       | Types.VARCHAR                |
| yearMonth     | java.time.YearMonth     | varchar       | Types.VARCHAR                |
| year          | java.time.Year          | int           | Types.INTEGER                |
| enum | kotlin.Enum | enum | Types.VARCHAR |
| uuid | java.util.UUID | uuid | Types.OTHER |

## Extend More Data Types

Sometimes, Ktorm's built-in data types may not satisfy your requirements. For example, you may want to save a JSON column to a table, many relational databases have supported JSON data type, but raw JDBC haven't yet, nor Ktorm doesn't support it by default. However, you can do it by yourself: 

```kotlin
class JsonSqlType<T : Any>(
    val objectMapper: ObjectMapper,
    val javaType: JavaType
) : SqlType<T>(Types.VARCHAR, "json") {

    override fun doSetParameter(ps: PreparedStatement, index: Int, parameter: T) {
        ps.setString(index, objectMapper.writeValueAsString(parameter))
    }

    override fun doGetResult(rs: ResultSet, index: Int): T? {
        val json = rs.getString(index)
        if (json.isNullOrBlank()) {
            return null
        } else {
            return objectMapper.readValue(json, javaType)
        }
    }
}
```

The class above is a subclass of `SqlType`, it provides JSON data type support via the Jackson framework. Now we have `JsonSqlType`, how can we use it to define a column? Looking back the `int` function's implementation above, we notice that there is a `registerColumn` function called. This function is exactly the entry point provided by Ktorm to support datatype extensions. We can also write an extension function like this: 

```kotlin
inline fun <reified C : Any> BaseTable<*>.json(
    name: String,
    mapper: ObjectMapper = sharedObjectMapper
): Column<C> {
    return registerColumn(name, JsonSqlType(mapper, mapper.constructType(typeOf<C>())))
}
```

The usage is as follows: 

```kotlin
object Foo : Table<Nothing>("foo") {
    val bar = json<List<Int>>("bar")
}
```

In this way, Ktorm is able to read and write JSON columns now. Actually, this is one of the features of the ktorm-jackson module, if you really need to use JSON columns, you don't have to repeat the code above, please add its dependency to your project: 

Maven： 

```xml
<dependency>
    <groupId>org.ktorm</groupId>
    <artifactId>ktorm-jackson</artifactId>
    <version>${ktorm.version}</version>
</dependency>
```

Or Gradle： 

```groovy
compile "org.ktorm:ktorm-jackson:${ktorm.version}"
```

Additionally, Ktorm 2.7 provides a `transform` function. With this function, we can extend data types based on existing ones by adding some specific transformations to them. In this way, we get new data types without writing `SqlType` implementations manually. 

For example, in the following code, we define a column of type `Column<UserRole>`, but the underlying SQL type in the database is still `int`. We just perform some transformations while obtaining column values or setting parameters into prepared statements. 

```kotlin
val role = int("role").transform({ UserRole.fromCode(it) }, { it.code })
```

Please note such transformations will perform on every access to a column, that means you should avoid heavy transformations here.
