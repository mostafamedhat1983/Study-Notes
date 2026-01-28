---
tags:
  - Linux
  - Bash_Script
---
## Bash Arrays

Bash arrays are variables that store multiple values, allowing you to organize and manipulate collections of data efficiently. Bash supports two types: indexed arrays (numeric indices) and associative arrays (string keys).

## Declaring Arrays

There are several ways to create arrays in bash:

**Direct initialization:**

``` bash
myArray=("cat" "dog" "mouse" "frog")
```

**Using declare command:**

``` bash
declare -a indexedArray    # Indexed array
declare -A assocArray      # Associative array (requires -A)
```

**Individual element assignment:**

``` bash
programming_languages[0]="Bash"
programming_languages[1]="Python"
programming_languages[2]="JavaScript"
```

Note that there are no commas between elements, and the equal sign should have no spaces around it.​

## Accessing Array Elements

Use curly braces with the array name and index to access elements:[](https://docs.vultr.com/how-to-use-arrays-in-bash)

``` bash
array=("Bash" "Python" "JavaScript")
echo ${array[0]}    # Output: Bash
echo ${array[1]}    # Output: Python
```

Bash arrays are zero-indexed, meaning the first element is at index 0.[](https://ioflood.com/blog/bash-declare-array/)

## Useful Array Operations

**Print all elements:**

``` bash
echo ${myArray[@]}    # All elements
echo ${myArray[*]}    # All elements (alternative)
```

**Get array length:**

``` bash
echo ${#myArray[@]}   # Number of elements
```

**Get all indices:**

``` bash
echo ${!myArray[@]}   # Prints all indices
```

**Loop through elements:**

``` bash
for item in "${myArray[@]}"; do
    echo $item
done

```

**Append elements:**

``` bash
myArray+=("new element")
```

## Associative Arrays

Associative arrays use string keys instead of numeric indices and must be declared with `-A`:[](https://docs.vultr.com/how-to-use-arrays-in-bash)

``` bash
declare -A fruits=(
  [apple]="red"
  [banana]="yellow"
  [cherry]="red"
)


echo ${fruits[apple]}     # Output: red
```

## Print All Keys

``` bash
echo "${!fruits[@]}"
```



Outputs: `apple banana cherry` (words separated by spaces).[](https://linuxhandbook.com/bash-associative-arrays/)

## Iterate Over Keys

``` bash
for key in "${!fruits[@]}"; do
  echo "Key: $key, Value: ${fruits[$key]}"
done
```

Quotes preserve keys with spaces; access values via `${array[$key]}`

**To check if a key has a value (exists and is set) in a Bash associative array, use `[[ -v array[key] ]]` (Bash 4.2+).**  
This tests if the key is defined, regardless of empty value.[](https://stackoverflow.com/questions/59395145/bash-check-if-key-exists-in-associative-array)  
For distinguishing set keys from unset ones (even if empty), use parameter expansion like `[[ ${array[key]+set} ]]`.[](https://stackoverflow.com/questions/59395145/bash-check-if-key-exists-in-associative-array)​

## Preferred Method

``` bash
declare -A fruits=( [apple]="red" [banana]="" )

if [[ -v fruits[apple] ]]; then
  echo "apple exists"
fi
```

Outputs "apple exists"; works for empty values too.[](https://nickjanetakis.com/blog/associative-arrays-in-bash-aka-key-value-dictionaries)

## Alternative for Non-Empty

``` bash
if [[ -n "${fruits[banana]}" ]]; then
  echo "banana has non-empty value"
else
  echo "banana empty or unset"
fi
```

`-n` checks length >0; fails for empty strings.[](https://phoenixnap.com/kb/bash-associative-array)​

## Key Existence Only

``` bash
if [[ ${fruits[cherry]+exists} ]]; then
  echo "cherry key is set"
fi
```
`+exists` expands only if key set; reliable for empty values.
## Important Notes

Always use quotes around array expansions like `"${array[@]}"` to preserve elements with spaces. The `@` symbol treats each element as a separate word, while `*` treats all elements as a single string.