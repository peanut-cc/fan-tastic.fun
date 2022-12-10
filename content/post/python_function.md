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
