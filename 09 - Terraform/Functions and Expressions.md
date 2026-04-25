---
tags:
  - Terraform
---
Terraform functions and expressions are used to calculate values, transform data, and write more flexible configuration.

They help you:
- build dynamic values
- combine strings
- work with lists and maps
- apply conditions
- avoid repeating hardcoded logic

These are very important in real Terraform projects because not every value should be fixed or written manually.

---

## Main idea

Terraform expressions are the pieces of code that produce values.

Terraform functions help you manipulate those values.

Simple idea:
- expression = any Terraform code that returns a value
- function = built-in helper that transforms or calculates values

Examples:
- `var.environment`
- `local.common_tags`
- `"app-${var.environment}"`
- `length(var.subnet_ids)`
- `upper(var.project_name)`

---

## What is an expression

An expression is anything in Terraform that produces a value.

Examples:

```hcl
var.region
```

```hcl
local.project_name
```

```hcl
aws_instance.web.id
```

```hcl
"app-${var.environment}"
```

Expressions are used everywhere in Terraform, such as:
- resource arguments
- locals
- outputs
- conditionals
- loops
- module inputs

---

## What is a function

A function is a built-in Terraform helper that takes values as input and returns a result.

Example:

```hcl
upper("dev")
```

This returns:

```hcl
"DEV"
```

Functions help you process:
- strings
- numbers
- lists
- maps
- file content
- encoded data
- IP-related values

---

## String interpolation

Terraform often uses string interpolation to build dynamic strings.

Example:

```hcl
"${var.project_name}-${var.environment}"
```

This combines values into one string.

Example result:

```hcl
"myapp-dev"
```

This is commonly used for:
- names
- tags
- identifiers
- environment-based resource naming

---

## Common expression examples

### Variable reference

```hcl
var.region
```

---

### Local reference

```hcl
local.bucket_name
```

---

### Resource attribute reference

```hcl
aws_instance.web.id
```

---

### Module output reference

```hcl
module.network.vpc_id
```

---

## Conditional expressions

Terraform supports conditional logic.

Syntax:

```hcl
condition ? true_value : false_value
```

Example:

```hcl
var.environment == "prod" ? "t3.medium" : "t2.micro"
```

This means:
- if environment is `prod`, use `t3.medium`
- otherwise use `t2.micro`

This is very common for environment-based behavior.

---

## Common string functions

### `upper`

Converts a string to uppercase.

```hcl
upper("dev")
```

Result:

```hcl
"DEV"
```

---

### `lower`

Converts a string to lowercase.

```hcl
lower("DEV")
```

Result:

```hcl
"dev"
```

---

### `title`

Converts text to title case.

```hcl
title("my app")
```

Result:

```hcl
"My App"
```

---

### `replace`

Replaces part of a string.

```hcl
replace("app-dev", "dev", "prod")
```

Result:

```hcl
"app-prod"
```

---

### `format`

Builds a formatted string.

```hcl
format("%s-%s", var.project_name, var.environment)
```

This is another way to build dynamic names.

---

## Common numeric and collection functions

### `length`

Returns the length of a string, list, or map.

```hcl
length(["a", "b", "c"])
```

Result:

```hcl
3
```

---

### `max`

Returns the largest value.

```hcl
max(1, 5, 3)
```

Result:

```hcl
5
```

---

### `min`

Returns the smallest value.

```hcl
min(1, 5, 3)
```

Result:

```hcl
1
```

---

## Common list functions

### `join`

Joins list elements into one string.

```hcl
join("-", ["app", "dev"])
```

Result:

```hcl
"app-dev"
```

---

### `split`

Splits a string into a list.

```hcl
split("-", "app-dev")
```

Result:

```hcl
["app", "dev"]
```

---

### `concat`

Combines multiple lists.

```hcl
concat(["a", "b"], ["c", "d"])
```

Result:

```hcl
["a", "b", "c", "d"]
```

---

### `element`

Returns an item by index.

```hcl
element(["a", "b", "c"], 1)
```

Result:

```hcl
"b"
```

---

## Common map functions

### `lookup`

Gets a value from a map by key.

```hcl
lookup(var.instance_types, "dev", "t2.micro")
```

This means:
- try to get the `dev` value
- if not found, return `t2.micro`

---

### `keys`

Returns the keys of a map.

```hcl
keys({dev = "small", prod = "large"})
```

---

### `values`

Returns the values of a map.

```hcl
values({dev = "small", prod = "large"})
```

### `merge`

Combines multiple maps or objects into one.

```hcl
merge(
  { env = "dev" },
  { owner = "mostafa" },
  { env = "prod" }
)
```

Result:

```hcl
{
  env   = "prod"
  owner = "mostafa"
}
```

If the same key appears in more than one map, the later value wins.

This is very commonly used for combining tag maps.

Example:

```hcl
locals {
  common_tags = merge(
    var.default_tags,
    { Environment = var.environment }
  )
}
```
---

## Useful conversion functions

### `toset`

Converts a value to a set.

```hcl
toset(["dev", "stage", "prod"])
```

This is often used with `for_each`.

---

### `tolist`

Converts a value to a list.

```hcl
tolist(["a", "b"])
```

---

### `tomap`

Converts a value to a map.

```hcl
tomap({
  env = "dev"
})
```

---

## Common encoding and file functions

### `file`

Reads a file from disk.

```hcl
file("user-data.sh")
```

This is useful for:
- startup scripts
- templates
- config files

---

### `base64encode`

Encodes a string in base64.

```hcl
base64encode("hello")
```

---

### `jsonencode`

Converts a Terraform value into a JSON string.

```hcl
jsonencode({
  env = "dev"
})
```

This is very useful when a provider expects JSON input.

---

## Expressions in locals

Functions and expressions are often used inside locals.

Example:

```hcl
locals {
  bucket_name   = lower("${var.project_name}-${var.environment}")
  instance_type = var.environment == "prod" ? "t3.medium" : "t2.micro"
}
```

This is a very common pattern in Terraform.

---

## Expressions in `for_each`

You often use expressions with loops and repetition.

Example:

```hcl
resource "aws_s3_bucket" "app" {
  for_each = toset(["dev", "stage", "prod"])
  bucket   = "myapp-${each.key}"
}
```

This combines:
- `for_each`
- `toset`
- interpolation

---

## Expressions in outputs

Outputs can also use expressions and functions.

Example:

```hcl
output "bucket_name_upper" {
  value = upper(local.bucket_name)
}
```

This returns a transformed value instead of a raw one.

---

## Common mistakes

- making expressions too complex
- nesting too many functions in one line
- using hardcoded values when expressions would be cleaner
- confusing `each.key`, `count.index`, `var.`, and `local.`
- writing logic that is difficult to read or debug

---

## Good practices

- keep expressions readable
- use locals to simplify complex expressions
- use functions when they improve clarity
- avoid unnecessary complexity
- prefer clear naming over clever one-liners
- use conditionals for simple branching, not overly complicated logic

---

## Important notes

- Expressions are values computed inside Terraform configuration.
- Functions are built-in helpers for transforming values.
- Conditional expressions use the form `condition ? true_value : false_value`.
- Functions are commonly used with strings, lists, maps, and files.
- Locals are often the best place to store complex expressions.
- Readability matters more than writing everything in one line.

---

## Simple rule of thumb

- use expressions to build dynamic values
- use functions to transform those values
- use locals to keep complicated logic readable
- keep Terraform logic simple and maintainable

---

## Must memorize

```hcl
var.environment
local.project_name
aws_instance.web.id
module.network.vpc_id
```

```hcl
"${var.project_name}-${var.environment}"
```

```hcl
var.environment == "prod" ? "t3.medium" : "t2.micro"
```

```hcl
upper("dev")
lower("DEV")
replace("app-dev", "dev", "prod")
format("%s-%s", var.project_name, var.environment)
```

```hcl
length(["a", "b", "c"])
join("-", ["app", "dev"])
split("-", "app-dev")
lookup(var.instance_types, "dev", "t2.micro")
toset(["dev", "stage", "prod"])
file("user-data.sh")
jsonencode({ env = "dev" })
```

---

## Key ideas

- Expressions produce values inside Terraform configuration.
- Functions transform and calculate values.
- Terraform functions are commonly used with strings, lists, maps, and files.
- Conditional expressions help change behavior based on input.
- Complex expressions are often best stored in locals.
- Good Terraform code should stay readable even when it is dynamic.