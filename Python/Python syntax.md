---
tags:
  - python
---
**PEP 8**: is the style guide for Python code, providing conventions for writing clean and readable code

**Variables:** Represent data stored as strings, tuples, dictionaries, lists, and objects

**Keywords:** Special words that are reserved for specific purposes and that can only be used for those purposes (in not or for while return)

**Operators:** Symbols that perform operations on objects and values (+ - * / % // < > =)  

---
**Functions:** A group of related statements to perform a task and return a value

```python
def to_celsius(x):
    """Convert Fahrenheit to Celsius."""
    return (x - 32) * 5 / 9
```


*Note:* `'''Convert Fahrenheit to Celsius'''`  is **docstring** , which is a **string written at the start of a module, function, class, or method that explains what it does**

| Code Element | Docstring Location |
|-------------|--------------------|
| Module | First line of the file |
| Function | First line inside the function |
| Class | First line inside the class |
| Method | First line inside the method |

to show docstring
```python
print(to_celsius.__doc__)
```
or
```python
help(to_celsius)
```
Result:
Convert Fahrenheit to Celsius.

---

**Conditional statements:** Sections of code that direct program execution based on specified conditions
```python
number = -4
if number > 0:
   print('Number is positive.')
elif number == 0:
   print('Number is zero.')
else:
   print('Number is negative.')
```


## Naming rules and conventions
- Names cannot contain spaces.
- Names may be a mixture of upper and lower case characters.
- Names can’t start with a number but may contain numbers after the first character.
- Variable names and function names should be written in snake_case, which means that all letters are lowercase and words are separated using an underscore. 
- Descriptive names are better than cryptic abbreviations because they help other programmers (and you) read and interpret your code. For example, student_name is better than sn. It may feel excessive when you write it, but when you return to your code you’ll find it much easier to understand