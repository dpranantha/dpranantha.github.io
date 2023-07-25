---
title: "Transforming data class objects to Google BigQuery TableRow via reflection in Kotlin"
permalink: /transforming-data-class-objects-to-gbq-tablerow-via-reflection-in-kotlin
date: 2021-01-06 21:00:00+01:00
categories:
- Coding
- Data engineering
tags:
- Kotlin
- Reflection
- Kotlin Object
- Member Properties
- Google BigQuery API
---

I had a use case to transform multiple data classes into the [Google BigQuery TableRow](https://developers.google.com/resources/api-libraries/documentation/bigquery/v2/java/latest/index.html?com/google/api/services/bigquery/model/TableRow.html) model.
It is relatively straightforward if we have data classes with only a couple of mandatory and optional properties.

```kotlin
data class ExampleClass(
   val mandatoryString: String,
   val mandatoryDouble: Double,
   val nullableString: String?,
   val nullableBoolean: Boolean?,
   val nullableInt: Int?,
   val nullableDouble: Double?,
   val mandatoryFloat: Float,
   val nullableDateTime: DateTime?)
```

Then, we can simply transform the ExampleClass above into the Google BigQuery TableRow model as below.

```kotlin
val example = ExampleClass("s", 1.0, null, true, 1, 0.1, 2.0f, null)
val transformedExample = example.let {
   val tableRow = TableRow()
   tableRow.set("mandatoryString", it.mandatoryString)
   tableRow.set("mandatoryDouble", it.mandatoryDouble)
   tableRow.set("nullableString", it.nullableString)
   tableRow.set("nullableBoolean", it.nullableBoolean)
   tableRow.set("nullableInt", it.nullableInt)
   tableRow.set("nullableDouble", it.nullableDouble)
   tableRow.set("mandatoryFloat", it.mandatoryFloat)
   tableRow.set("nullableDateTime", it.nullableDateTime)
   tableRow
}
```

We can further extract the codes inside `let` into an extension function of ExampleClass. However, the problem is that 
my data classes have many fields, i.e., between 50 and 100. It makes transformation logic rather tedious, not to mention extra effort for code maintenance.

To avoid writing repetitive and long lines of code, I had to once again rely on my old friend, reflection (using [kotlin-reflect](https://kotlinlang.org/api/latest/jvm/stdlib/kotlin.reflect/)).
Thus, instead of writing transformation for each data class and accessing each of its property members, I can use reflection to fill out all the TableRow fields. 
Also, I can set a table schema for a Google BigQuery table via reflection. Kotlin reflection has all information on the nullability and type of each property within a class.

The codes are concise and save me some trouble from maintaining the code, such as adding/removing a property from a data class (assuming there is a design process to maintain `backward-compatibility`) 
or introducing new data classes into the codes. Here is what it looks like

```kotlin
inline fun <reified T : Any> T.toTableRow(): TableRow {
    val tableRow = TableRow()
    for (prop in T::class.declaredMemberProperties) {
        prop.isAccessible = true
        tableRow.set(prop.name, prop.get(this))
    }
    return tableRow
}
```

It is an extension function for any object. We need an `inline function` with `reified` type parameters to access
type `T` and object within property `prop.get()` function. Now, I can write the transformation for `ExampleClass` below.

```kotlin
val example = ExampleClass("s", 1.0, null, true, 1, 0.1, 2.0f, null)
val transformedExample = example.toTableRow()
```

How about transforming a data class into Google BigQuery [TableSchema](https://cloud.google.com/bigquery/docs/schemas)?
That is a bit involved but also straightforward. In this case, we want to transform the `ExampleClass` data class into the following schema.

```
ExampleClass TableSchema
   - field: mandatoryString, type: STRING, mode: REQUIRED
   - field: mandatoryDouble: FLOAT64, mode: REQUIRED
   - field: nullableString: STRING, mode: NULLABLE
   - field: nullableBoolean: BOOL, mode: NULLABLE
   - field: nullableInt: INT64, mode: NULLABLE
   - field: nullableDouble: FLOAT64, mode: NULLABLE
   - field: mandatoryFloat: FLOAT64, mode: REQUIRED
   - field: nullableDateTime: TIMESTAMP, mode: NULLABLE
```

The transformation helper codes are shown below.

```kotlin
fun <T : Any> KClass<T>.toTableSchema(): TableSchema {
    val mutableFields = mutableListOf<TableFieldSchema>()
    for (prop in this.declaredMemberProperties) {
        mutableFields.add(
            TableFieldSchema().setName(prop.name)
            .setType(prop.returnType.classifier?.determineType())
            .setMode(prop.returnType.determineMode())
        )
    }
    return TableSchema().setFields(mutableFields.toList())
}

private fun KType.determineMode(): String = if (this.isMarkedNullable) "NULLABLE" else "REQUIRED"

private fun KClassifier?.determineType(): String {
    return this?.let {
        when (it) {
            Float::class -> "FLOAT64"
            Double::class -> "FLOAT64"
            Int::class -> "INT64"
            Boolean::class -> "BOOL"
            DateTime::class -> "TIMESTAMP"  //DateTime class from com.google.api.client.util.DateTime
            else -> "STRING"  //for now date and other types are treated as string
        }
    } ?: "STRING"
}
```

The first extension function is the exposed function, which returns Google BigQuery `TableSchema` class.
The other two private functions determine the mode (based on the nullability of each property) and the type, respectively.
Hence, I can create a schema out of the `ExampleClass` data class by simply writing.

```kotlin
val schema = ExampleClass::class.toTableSchema()
```

This can be used for any data class. It is pretty handy, isn't it?!

Have fun coding!
