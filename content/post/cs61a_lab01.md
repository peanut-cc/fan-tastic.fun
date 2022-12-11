---
title: "CS61a_lab01"
date: 2022-12-11T14:29:47+08:00
draft: false
tags: ["cs61a", "python"]
categories: ["cs61a"]
---

Although this lab is not difficult, after writing the answer and comparing it with the answer, I found that there is still something to learn.

## lab01.1

```python
def falling(n, k):
    """Compute the falling factorial of n to depth k.

    >>> falling(6, 3)  # 6 * 5 * 4
    120
    >>> falling(4, 3)  # 4 * 3 * 2
    24
    >>> falling(4, 1)  # 4
    4
    >>> falling(4, 0)
    1
    """
```

The code written by this lab itself is indeed more concise than the answer.I feel that I have some problems in my thinking. My own thinking is more troublesome, although my own can also solve the problem.Perhaps more practice is needed to exercise the thinking ability.

my answer:

```python
def falling(n, k):
    """Compute the falling factorial of n to depth k.

    >>> falling(6, 3)  # 6 * 5 * 4
    120
    >>> falling(4, 3)  # 4 * 3 * 2
    24
    >>> falling(4, 1)  # 4
    4
    >>> falling(4, 0)
    1
    """
    if k == 0:
        return 1
    result = 1
    number = n - k
    while n > number:
        result = result * n
        n -= 1
    return result
```

better answer:

```python
def falling(n, k):
    """Compute the falling factorial of n to depth k.

    >>> falling(6, 3)  # 6 * 5 * 4
    120
    >>> falling(4, 3)  # 4 * 3 * 2
    24
    >>> falling(4, 1)  # 4
    4
    >>> falling(4, 0)
    1
    """
    total, stop = 1, n - k
    while n > stop:
        total, n = total * n, n - 1
    return total
```

For this lab, my idea is correct, but the code is not concise engouth.
After solving the problem, I should try to optimize the code to see if there is a more elegant way tof coding.

## lab01.2

```python
def sum_digits(y):
    """Sum all the digits of y.

    >>> sum_digits(10) # 1 + 0 = 1
    1
    >>> sum_digits(4224) # 4 + 2 + 2 + 4 = 12
    12
    >>> sum_digits(1234567890)
    45
    >>> a = sum_digits(123) # make sure that you are using return rather than print
    >>> a
    6
    """
```

my answer:

```python
def sum_digits(y):
    """Sum all the digits of y.

    >>> sum_digits(10) # 1 + 0 = 1
    1
    >>> sum_digits(4224) # 4 + 2 + 2 + 4 = 12
    12
    >>> sum_digits(1234567890)
    45
    >>> a = sum_digits(123) # make sure that you are using return rather than print
    >>> a
    6
    """
    result = 0
    while y != 0:
        result += y % 10
        y = y // 10
    return result
```

better code:

```python
def sum_digits(y):
    """Sum all the digits of y.

    >>> sum_digits(10) # 1 + 0 = 1
    1
    >>> sum_digits(4224) # 4 + 2 + 2 + 4 = 12
    12
    >>> sum_digits(1234567890)
    45
    >>> a = sum_digits(123) # make sure that you are using return rather than print
    >>> a
    6
    """
    total = 0
    while y > 0:
        total, y = total + y % 10, y // 10
    return total
```

For this lab, my idea is correct,the code name is neat and the difference is not big, but the code of the answer is more pythonic.

## lab01.3

```python
def double_eights(n):
    """Return true if n has two eights in a row.
    >>> double_eights(8)
    False
    >>> double_eights(88)
    True
    >>> double_eights(2882)
    True
    >>> double_eights(880088)
    True
    >>> double_eights(12345)
    False
    >>> double_eights(80808080)
    False
    """
```

my answer:

```python
def double_eights(n):
    """Return true if n has two eights in a row.
    >>> double_eights(8)
    False
    >>> double_eights(88)
    True
    >>> double_eights(2882)
    True
    >>> double_eights(880088)
    True
    >>> double_eights(12345)
    False
    >>> double_eights(80808080)
    False
    """
    is_appeared_eight = False
    while n != 0:
        result = n % 10
        if is_appeared_eight and result == 8:
            return True
        elif is_appeared_eight and result != 8:
            is_appeared_eight = False
        elif not is_appeared_eight and result == 8:
            is_appeared_eight = True
        n = n // 10
    return False
```

better answer:

```python
def double_eights(n):
    """Return true if n has two eights in a row.
    >>> double_eights(8)
    False
    >>> double_eights(88)
    True
    >>> double_eights(2882)
    True
    >>> double_eights(880088)
    True
    >>> double_eights(12345)
    False
    >>> double_eights(80808080)
    False
    """
    prev_eight = False
    while n > 0:
        last_digit = n % 10
        if last_digit == 8 and prev_eight:
            return True
        elif last_digit == 8:
            prev_eight = True
        else:
            prev_eight = False
        n = n // 10
    return False
```

In this lab, I think the if/else of my answer looks clearer.
