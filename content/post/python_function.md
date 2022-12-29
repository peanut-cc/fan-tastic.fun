---
title: "How to define Function"
date: 2022-12-10T19:51:40+08:00
draft: false
tags: ["cs61a", "python","function"]
categories: ["cs61a"]
---

## A Guide to Designing Function

- Give each function exactly one job, but make it apply to many related situations.
- Don't repeat yourself(DRY): Implement a process juest once, but execute it many times.
- Define functions generally.

## Generalizing Patterns with Arguments

```python
"""Generalization."""

from math import pi, sqrt

def area_square(r):
    return r *r

def area_circle(r):
    return r * r * pi

def area_hexago(r):
    return r * r * 3 * sqrt(3) / 2

```

If we need to make some judgements on r now, maybe we need to add in each function.But at this point you should remember the rule: `Don't Repeat youself`.

Improve the above code:

```python
"""Generalization."""

from math import pi, sqrt

def area(r, shape_constant):
    assert r > 0, 'A length must be positive'
    return r * r * shape_constant

def area_square(r):
    return area(r, 1)

def area_circle(r):
    return area(r, pi)

def area_hexago(r):
    return area(r, 3 * sqrt(3) / 2) 
```

## Locally Defined functions

Functions defined within tohner function bodies are bound to names in a local frame.

```python
"""Generalization."""

def make_adder(n):
    """Return a function that take one k and return k+n
    >> add_three = make_adder(3)
    >> add_three(4)
    7
    """
    def adder(k):
        return k + n
    return adder
```

## The purpose of Higer-Order Functions

`Functions are first-class`: Functions can be manipulated as values in our programming language.

`Heigher-order function`: A function that takes a function as an argument value or returns a function as a return value.

- Higher-order functions:
- Express general methods of computation.
- Remove repetition from programs.
- Separate concerns among functions.

## Enviroments for Heigher-Order Functions

`Heigher-order function`: A function that takes a function as an argument value or returns a function as a return value.

Our environment diagrams already handle the case of heigher-order functions!

```python
def make_adder(n):
    def adder(k):
        return k+n
    return adder
```

- Every user-defined function has a parent frame(often global)
- The parent of a function is the frame in which it was defined.
- The parent of a frame is the parent of the function called.

### How to Draw an Enviroment Diagram

When a function is defined:

Create a function value: `func<name>(<formal parameters>)[parent=<parent>]`
Bind `<name>` to the function value in the current frame.

When a function is called:

1. Add a local fram, titled with the `<name>` of the function being called.
2. Copy the parent of the function to the local frame: `[parent=<label>]`
3. Bind the `formal parameters` to the arguments in the local frame.
4. Execute the body of the function in the environment that starts with the local frame.

### Labmda Function Environments

A lambda function's parent is the current frame in which the lambda expression is evaluated.

```python
a = 1
def f(g):
    a = 2
    return lambda y: a * g(y)
f(lambda y: a + y)(a)
```

![Environment Diagrams with Lambda](/images/python_function/lambda.png)

personal understanding:

- The current program is running with a global global frame When the program is loaded.
- `a = 1` is in the global frame.
- `lambda y: a + y` is in the global frame. so `a = 1`
- `lambda y: a * g(y)` frame is in f. so `a = 2`
- when executing `f(lambda y: a + y)(a)` finally execute `lambda y: a * g(y)`, a = 2,y=2. and in g(y),a = 1, y = 1
- So the final caculation result is 4
