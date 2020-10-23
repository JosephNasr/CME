# Algorithmic Data Mappings

- [Algorithmic Data Mappings](#algorithmic-data-mappings)
  - [Description](#description)
  - [S3 Folder Structure](#s3-folder-structure)
  - [ADM File Structure](#adm-file-structure)
    - [Mapping Object](#mapping-object)
      - [Value Operation](#value-operation)
        - [Default Value](#default-value)
        - [Path](#path)
        - [Alias](#alias)
        - [Computation](#computation)
        - [Parse To](#parse-to)
    - [Alias Object](#alias-object)
  - [Computation Definitions](#computation-definitions)
    - [Average](#average)
    - [Choice](#choice)
    - [Complex Choice](#complex-choice)
      - [Conditional Values](#conditional-values)
      - [Conditional](#conditional)
        - [Base Condition](#base-condition)
        - [NOT Condition](#not-condition)
        - [OR Condition](#or-condition)
        - [AND Condition](#and-condition)
    - [Concat](#concat)
    - [Count](#count)
    - [Select](#select)
    - [Transform Date](#transform-date)
    - [Transform String](#transform-string)

## Description

This repository contains the Java Spring Boot implementation of the [Algorithmic Data Mapppings (ADM) library](https://confluence.buehner-fry.com/display/NDEX/NDEX+GXP+Data+Lake#NDEXGXPDataLake-Data-drivenalgorithmicinterfacemappings) in the `adm` module.

## S3 Folder Structure

In order to use the ADM library you will need to upload the mappings to S3 as JSON files, which will then be pulled, applied, and cached for each new item received.

The ADM library will try and find 6[^1] mapping files per item, one for each level, going from the most generic to the most specific[^2] in the following structure:

[^1]: Not all 6 files are required. Any combination of 1 or more files can be created.
[^2]: Each mapping file has the ability to override mappings from its predecessor generic ones.

```text
S3-mappings/                    # Root directory
└── Common/
    ├── Common.json             # Mappings for all items
    └── <EntityType>.json       # Mappings for items of a specific entity
    <PmsInterfaceType>/
    ├── Common/
    │   ├── Common.json         # Mappings for items of a specific interface
    │   └── <EntityType>.json   # Mappings for items of a specific interface and entity
    └── <AccountNumber>/
        ├── Common.json         # Mappings for items of a specific interface and account
        └── <EntityType>.json   # Mappings for items of a specific interface and account and entity
```

## ADM File Structure

An ADM file is a simple JSON file that is validated on runtime against a pre-defined schema.

```json
{
    "mappings": [
        ...
    ],
    "aliases": [
        ...
    ]
}
```

- `mappings`
  - An array of [Mapping objects](#mapping-object)
  - *Required*
- `aliases`
  - An array of [Alias objects](#alias-object)
  - *Optional*

### Mapping Object

```json
{
    "destinationFieldName": "SomeFieldName",
    "value": {
        ...
    }
}
```

- `destinationFieldName`
  - String representing the name of the resulting mapping
  - *Required*
  - Mappings with similar names may override one another [^2]
- `value`
  - [Value Operation](#value-operation) representing the value of the resulting mapping
  - *Required*

#### Value Operation

A value operation is a set of one or more rules that define how a mapping is constructed. It was designed in a recursive manner to allow for more complex mapping definitions, without sacrificing its usage simplicity.

There are 4 basic ways to define a value operation:

##### Default Value

```json
{
    "defaultValue": "This is a static value used as is"
}
```

- Returns the value as is
- Accepts and preserves types
  - String
  - Number
  - Boolean
- If value is *null*, the mapping would not be included in the final result
- Can be added to the other basic operations as a default behavior, if said operation failed

```json
{
    "path": "This.Is.An...Invalid.Path",
    "defaultValue": true
}
```

The above example will return `true`

##### Path

```json
{
    "path": "This.Returns.Value.From.JSON.Path"
}
```

- Computes and returns the value of the provided JSON path string
- Preserves the type of the requested field
- If value is *null*, the mapping would not be included in the final result

##### Alias

```json
{
    "alias": "thisReturnsValueFromAnAlias"
}
```

- Computes and returns the value of the provided [Alias object](#alias-object)
- Preserves the type of the requested field
- If value is *null*, the mapping would not be included in the final result

##### Computation

```json
{
    "computation": {
        ...
    }
}
```

- Computes and returns the value of the provided [Computation object](#computation-definitions)
  - One and only one computation is required
- Preserves the type of the requested field
- If value is *null*, the mapping would not be included in the final result

##### Parse To

In addition to the basic operations, ADM adds an **optional** `parseTo` field that does what the name suggests; parse the result to the specified type.

Available types:

- `string`
- `integer`
- `double`
- `boolean`

```json
{
    "defaultValue": "true",
    "parseTo": "boolean"
}
```

The above example will return `true` and not `"true"`, which would be the same as having the following:

```json
{
    "defaultValue": true
}
```

### Alias Object

An Alias can be perceived as a *function* or a *method*, which is basically a reusable value that is used to perform a single operation.

```json
{
    "name": "anAliasName",
    "computeOnce": true,
    "value": {
        ...
    }
}
```

- `name`
  - A unique name to represent the alias
  - *Required*
  - Is the value to be used with alias value operations
- `computeOnce`
  - A boolean flag to specify whether or not the result is to be cached and not computed with each alias call
  - *Optional* - Default is false
- `value`
  - A [Value Operation](#value-operation) that defines the value to be returned from the alias
  - *Required*

## Computation Definitions

Computations are a set of atomic operations that can be nested to construct a mapping as intended, no matter the complexity.

### Average

An `average` computes the average value of a field recurring in a list.

| Parameter     | Type   | Description                                                                      | Required |
| ------------- | ------ | -------------------------------------------------------------------------------- | -------- |
| `listPath`    | String | JSON path pointing to the root of the list                                       | True     |
| `selectField` | String | The field from within an item of the list on which the average is to be computed | True     |

```json
{
    "average": {
        "listPath": "This.Is.A.Path.Of.A.List",
        "selectField": "thisIsAFieldName"
    }
}
```

### Choice

A `choice` computes if a given field is equal to a certain value, and returns the associoated result, similar to a `switch case`.

| Parameter     | Type                                | Description                                                  | Required                 |
| ------------- | ----------------------------------- | ------------------------------------------------------------ | ------------------------ |
| `field`       | [Value Operation](#value-operation) | The value which the conditions will be tested against        | True                     |
| `cases`       | Map                                 | A map of conditions with their returning values              | True                     |
| `defaultCase` | String, Number, Boolean             | The value to return if no condition was met                  | True                     |
| `ignoreCase`  | Boolean                             | A flag indicating if the equality should be case insensitive | False - Default: *false* |

```json
{
    "choice": {
        "field": {
            "defaultValue": "PI"
        },
        "cases": {
            "e": 2.71828,
            "pi": 3.14159
        },
        "defaultCase": 0,
        "ignoreCase": true
    }
}
```

The above computation will return `3.14159`.

### Complex Choice

A `complexChoice` computes a more complex choice operation. It includes a wider varierty of equality operators, and more complex intricacies between conditions.

| Parameter           | Type                                      | Description                                 | Required |
| ------------------- | ----------------------------------------- | ------------------------------------------- | -------- |
| `conditionalValues` | [Conditional Values](#conditional-values) | The list of conditions to test              | True     |
| `defaultCase`       | [Value Operation](#value-operation)       | The value to return if no condition was met | True     |

```json
{
    "complexChoice": {
        "conditionalValues": [
            ...
        ],
        "defaultCase": {
            "defaultValue": "Return this value if all conditionals failed"
        }
    }
}
```

#### Conditional Values

A list of conditionals, where each conditional would return a value if its condition was met.

| Parameter     | Type                                | Description                                    | Required |
| ------------- | ----------------------------------- | ---------------------------------------------- | -------- |
| `conditional` | [Conditional](#conditional)         | The condition to test                          | True     |
| `value`       | [Value Operation](#value-operation) | The value to return if the conditional was met | True     |

```json
{
    "conditional": {
        ...
    },
    "value": {
        "defaultValue": "Return this value if the conditional was met"
    }
}
```

#### Conditional

A single complex condition that acts as an `if-statement`, with **AND**, **OR**, and **NOT** operations.

##### Base Condition

The base condition where the actual test is done.

| Parameter     | Type                                                                       | Description                  | Required |
| ------------- | -------------------------------------------------------------------------- | ---------------------------- | -------- |
| `firstField`  | [Value Operation](#value-operation)                                        | The first field of the test  | True     |
| `operation`   | *equals*, *greaterThan*, *greaterThanEquals*, *lessThan*, *lessThanEquals* | The test to perform          | True     |
| `secondField` | [Value Operation](#value-operation)                                        | The second field of the test | True     |

```json
{
    "firstField": { "defaultValue": 15 },
    "operation": "greaterThanEquals",
    "secondField": { "defaultValue": 10 }
}
```

The above conditional is `true`.

##### NOT Condition

A conditional operator that negates the result of a child conditional.

| Parameter | Type                        | Description                         | Required |
| --------- | --------------------------- | ----------------------------------- | -------- |
| `not`     | [Conditional](#conditional) | The conditional to negate its value | True     |

```json
{
    "not": {
        "firstField": { "defaultValue": 15 },
        "operation": "greaterThanEquals",
        "secondField": { "defaultValue": 10 }
    }
}
```

The above conditional is `false`.

##### OR Condition

A conditional operator that is met if at least one of its child conditionals are met.

| Parameter | Type                          | Description                      | Required |
| --------- | ----------------------------- | -------------------------------- | -------- |
| `or`      | [Conditional](#conditional)[] | The list of conditionals to test | True     |

```json
{
    "or": [
        {
            "firstField": { "defaultValue": 15 },
            "operation": "greaterThanEquals",
            "secondField": { "defaultValue": 10 }
        },
        {
            "not": {
                "firstField": { "defaultValue": 15 },
                "operation": "greaterThanEquals",
                "secondField": { "defaultValue": 10 }
            }
        }
    ]
}
```

The above conditional is `true`.

##### AND Condition

A conditional operator that is met if exactly all of its child conditionals are met.

| Parameter | Type                          | Description                      | Required |
| --------- | ----------------------------- | -------------------------------- | -------- |
| `and`     | [Conditional](#conditional)[] | The list of conditionals to test | True     |

```json
{
    "and": [
        {
            "or": [
                {
                    "firstField": { "defaultValue": 15 },
                    "operation": "greaterThanEquals",
                    "secondField": { "defaultValue": 10 }
                },
                {
                    "not": {
                        "firstField": { "defaultValue": 15 },
                        "operation": "greaterThanEquals",
                        "secondField": { "defaultValue": 10 }
                    }
                }
            ]
        },
        {
            "firstField": { "defaultValue": 15 },
            "operation": "greaterThanEquals",
            "secondField": { "defaultValue": 10 }
        }
    ]
}
```

The above conditional is `true`.

### Concat

A `concat` operation chains a series of strings together using an optional separator.

| Parameter   | Type                                  | Description                                  | Required |
| ----------- | ------------------------------------- | -------------------------------------------- | -------- |
| `values`    | [Value Operation](#value-operation)[] | The values to concatenate                    | True     |
| `separator` | String                                | The string used to chain together the values | False    |

```json
{
    "concat": {
        "values": [
            { "defaultValue": "John" },
            { "defaultValue": "Smith" },
            { "defaultValue": "Doe" },
        ],
        "separator": " "
    }
}
```

The above computation will return `"John Smith Doe"`.

### Count

A `count` computes the number of elements in a certain list.

| Parameter  | Type   | Description                                | Required |
| ---------- | ------ | ------------------------------------------ | -------- |
| `listPath` | String | JSON path pointing to the root of the list | True     |

```json
{
    "count": {
        "listPath": "This.Is.A.Path.Of.A.List",
    }
}
```

### Select

A `select` computes a single value from a list of items, based on a certain condition.

| Parameter     | Type                                | Description                                                              | Required |
| ------------- | ----------------------------------- | ------------------------------------------------------------------------ | -------- |
| `listPath`    | String                              | JSON path pointing to the root of the list                               | True     |
| `selectField` | String                              | The field name to select from within the list                            | True     |
| `conditional` | [Conditional](#conditional)         | The condition that will be tested against each list item                 | True     |
| `defaultCase` | [Value Operation](#value-operation) | The value to return if the condition did not match against any list item | True     |

```json
{
    "select": {
        "listPath": "This.Is.A.Path.Of.A.List",
        "selectField": "ThisIsAFieldName",
        "conditional": {
            ...
        },
        "defaultCase": "Return this value if the conditional did not match any list item"
    }
}
```

> **NOTE**
> : A `path` operation used within the `select`'s `conditional` should be a relative path, and not absolute:
>
> Sample JSON data with list:
>
> ```json
> {
>     "root": {
>         "list": [
>             {
>                 "name": "John",
>                 "age": 30
>             },
>             {
>                 "name": "Smith",
>                 "age": 25
>             }
>         ]
>     }
> }
> ```
>
> Computation:
>
> ```json
> {
>     "select": {
>         "listPath": "root.list",
>         "selectField": "name",
>         "conditional": {
>             "firstField": { "path": "age" },          # The path is relative to the list itself, and not the root
>             "operation": "lessThan",
>             "secondField": { "defaultValue": "27" }
>         },
>         "defaultCase": "Unknown"
>     }
> }
> ```

### Transform Date

A `transformDate` parses a date string and has the ability the transform it using its sub-operations.

| Parameter      | Type                                | Description                                       | Required |
| -------------- | ----------------------------------- | ------------------------------------------------- | -------- |
| `dateField`    | [Value Operation](#value-operation) | The date to parse                                 | True     |
| `inputFormat`  | String                              | The format of which the date is to be parsed from | True     |
| `outputFormat` | String                              | The format of the resulting date value            | False    |
| `addDays`      | [Value Operation](#value-operation) | A number value that will add days to the date     | False    |
| `addMonths`    | [Value Operation](#value-operation) | A number value that will add months to the date   | False    |
| `addYears`     | [Value Operation](#value-operation) | A number value that will add years to the date    | False    |

```json
{
    "transformDate": {
        "dateField": { "defaultValue": "01-01-2020" },
        "inputFormat": "MM-dd-yyyy",
        "outputFormat": "d MMM yyyy",
        "addDays": { "defaultValue": 1 },
        "addMonths": { "defaultValue": 2 },
        "addYears": { "defaultValue": 3 }
    }
}
```

The above computation will return `"2 Mar 2023"`.

### Transform String

A `transformString` has the ability the transform a string value it using its sub-operations.

| Parameter       | Type                                | Description                                       | Required |
| --------------- | ----------------------------------- | ------------------------------------------------- | -------- |
| `field`         | [Value Operation](#value-operation) | The string to transform                           | True     |
| `toLower`       | Boolean                             | The format of which the date is to be parsed from | False    |
| `toUpper`       | Boolean                             | The format of the resulting date value            | False    |
| `substringFrom` | String, Integer                     | An index value indicating the start of the string | False    |
| `substringTo`   | String, Integer                     | An index value indicating the end of the string   | False    |

```json
{
    "transformString": {
        "field": { "defaultValue": "#january" },
        "toUpper": true,
        "substringFrom": 1,
        "substringTo": 4
    }
}
```

The above computation will return `"JAN"`.

```json
{
    "transformString": {
        "field": { "defaultValue": "2020-01-01T12:40:58+0000" },
        "substringTo": 10
    }
}
```

The above computation will return `"2020-01-01"`.

---
