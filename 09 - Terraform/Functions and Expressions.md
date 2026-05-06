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

## Ellipsis operator (`...`)

The `...` operator is used for **argument expansion**.

It takes a list and expands it into individual separate arguments for a function that expects multiple arguments instead of a single list.

### The problem it solves

Some Terraform functions like `max()` and `min()` expect individual values, not a list.

Without `...` you would have to write each value manually:

```hcl
max(10, 20, 30)
```

With `...` you can pass a list variable and expand it automatically:

```hcl
variable "numbers" {
  type    = list(number)
  default = [10, 20, 30]
}

max(var.numbers...)   # same as max(10, 20, 30)
```

Result:

```hcl
30
```

---

### Another example with `min`

```hcl
variable "ports" {
  type    = list(number)
  default = [8080, 443, 80]
}

min(var.ports...)   # same as min(8080, 443, 80)
```

Result:

```hcl
80
```

---

### Without vs with `...`

```hcl
# Without ... → passes the list as a single argument → ERROR for max/min
max([10, 20, 30])       # ❌ wrong - max does not accept a list

# With ... → expands the list into individual arguments → CORRECT
max([10, 20, 30]...)    # ✅ correct - same as max(10, 20, 30)
```

---

### Simple rule

- `...` expands a list into individual arguments
- only use it when a function expects multiple separate values, not a list
- most common with `max()`, `min()`, and `concat()`

---

## Splat operator (`[*]`)

The splat operator is used to extract one attribute from **every element** in a list at once.

Instead of looping manually, you use `[*]` to collect a single attribute across all elements into a new list.

### The problem it solves

Imagine you have 3 EC2 instances created with `count` and you want all their IDs.

Without splat you would have to access each one individually:

```hcl
aws_instance.web[0].id
aws_instance.web[1].id
aws_instance.web[2].id
```

With splat you get all of them in one line:

```hcl
aws_instance.web[*].id   # → ["id-1", "id-2", "id-3"]
```

---

### Example with `count`

```hcl
resource "aws_instance" "web" {
  count         = 3
  ami           = "ami-12345678"
  instance_type = "t2.micro"
}

# Get all instance IDs at once
output "all_instance_ids" {
  value = aws_instance.web[*].id
}

# Get all private IPs at once
output "all_private_ips" {
  value = aws_instance.web[*].private_ip
}
```

Result of `all_instance_ids`:

```hcl
["i-0abc123", "i-0def456", "i-0ghi789"]
```

---

### Example with a variable list of objects

```hcl
variable "users" {
  type = list(object({
    name  = string
    email = string
  }))
  default = [
    { name = "Ali",     email = "ali@example.com" },
    { name = "Sara",    email = "sara@example.com" },
    { name = "Mostafa", email = "mostafa@example.com" }
  ]
}

var.users[*].name    # → ["Ali", "Sara", "Mostafa"]
var.users[*].email   # → ["ali@example.com", "sara@example.com", "mostafa@example.com"]
```

---

### Without vs with `[*]`

```hcl
# Without splat → access one element at a time
aws_instance.web[0].id   # only the first instance
aws_instance.web[1].id   # only the second instance

# With splat → get all at once as a list
aws_instance.web[*].id   # ✅ all instance IDs in one list
```

---

### Simple rule

- `[*]` means "for every element, give me this attribute"
- the result is always a list
- most commonly used in outputs to collect attributes from count-based resources
- only works on lists, not maps or sets

---

### Ellipsis vs Splat — do not confuse them

| Operator | Syntax | What it does |
|----------|--------|--------------|
| Ellipsis | `var.list...` | Expands a list into individual function arguments |
| Splat | `resource[*].attr` | Extracts one attribute from every element in a list |

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
- forgetting `...` when passing a list to a function that expects individual arguments
- confusing `[*]` splat with `...` ellipsis — they do different things

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
- The `...` operator expands a list into individual arguments for functions that expect separate values.
- The `[*]` splat operator extracts one attribute from every element in a list.

---

## Simple rule of thumb

- use expressions to build dynamic values
- use functions to transform those values
- use locals to keep complicated logic readable
- keep Terraform logic simple and maintainable
- use `...` when a function needs individual values but you have a list
- use `[*]` when you want one attribute from every element in a list

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
max(var.numbers...)
min(var.ports...)
aws_instance.web[*].id
var.users[*].name
```

---

## Key ideas

- Expressions produce values inside Terraform configuration.
- Functions transform and calculate values.
- Terraform functions are commonly used with strings, lists, maps, and files.
- Conditional expressions help change behavior based on input.
- Complex expressions are often best stored in locals.
- Good Terraform code should stay readable even when it is dynamic.
- The `...` operator expands a list into individual arguments for functions that expect separate values.
- The `[*]` splat operator extracts one attribute from every element in a list and returns a new list.
