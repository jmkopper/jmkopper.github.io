---
title: 'Optimal but terrible answers to common coding interview questions'
date: 2022-10-21
permalink: /posts/2022/10/interview-qs/
tags:
  - python
  - julia
  - fizzbuzz
  - twosum
  - hello world
  - coding interviews
---

Today we have set ourselves the task of providing technically correct but actually appalling answers to some common coding exercises.
The exercises under consideration are

- Hello, world!
- FizzBuzz
- Two-sum

Our goal is to create optimal (by big-O time complexity) algorithms to complete these exercises with the added twist that the solutions should be as bad as possible otherwise.

## Hello, world!

We will write our `Hello, world!` program in Python because Python is a bad language and therefore any program written in Python is automatically bad. Constraints:

- Program must print the string `Hello, world!`
- Program must execute in O(1) time

Easy peasy. First we note that `Hello, world!` is a unicode string, and therefore can be obtained by joining the correct set of characters together. We select the UTF-8 encoding, because that's pretty standard.
All that remains is to generate the correct string and to print it out. To do this, we iterate through all possible UTF-8 strings of all possible lengths, starting at length 1. To rapidly check that we have the correct
string, we hash it and compare it to the hash value of `Hello, world!`. For added security, we use a SHA-256 hash.

```python
import hashlib
from itertools import permutations

def hello_world(i: int) -> bool:
    for xs in permutations(range(0, 0x110000), i):
        if all([chr(s).isprintable() for s in xs]):
            ys = "".join(chr(s) for s in xs)
            if hashlib.sha256(ys.encode("utf-8")).hexdigest() == "315f5bdb76d078c43b8ac0064e4a0164612b1fce77c869345bfc94c75894edd3":
                print(ys)
                return True
    return False


def main():
    i = 0
    while True:
        if hello_world(i):
            break
        i += 1

if __name__ == "__main__":
    main()
```

To analyze the time complexity, note that `Hello, world!` is a string with 13 characters. Since our algorithm begins at length 1, then increments length by 1 at a time, it will always take the exact same number of computations to arrive at the correct string. Therefore the time complexity is O(1).

## FizzBuzz

This interview classic has stymied the greatest coding minds since time immemorial. Here I present a fool-proof solution based on sound reasoning that will astonish any code reviewer on the planet. Constraints:

- The program must print all positive integers (up to some large value, say). If the integer is a multiple of 3, print "Fizz" instead of the integer. If it's a multiple of 5, print "Buzz" instead of the integer. If it's a multiple of both 3 and 5, print "FizzBuzz" instead of the integer.
- Program must execute in O(n) time, where `n` is the maximum integer printed.

How do we begin such a complicated task? We start by simplifying the problem. If we only had to print all the numbers up to `1`, our program might look like:

```julia
print("1")
```

If we had to do all the numbers up to 10, a very good and not at all bad solution would be

```julia
print("1")
print("2")
print("Fizz")
print("4")
print("Buzz")
print("Fizz")
print("7")
print("8")
print("Fizz")
print("Buzz")
```

It's quite obvious that this program is perfect. Unfortunately, we may not know beforehand the largest value we need to print. Or we might know, but that number might be very large and our hands will hurt from all the typing.
It is therefore clear that we need to *generate* the correct program. Unfortunately, Python doens't have much in the way of tools for metaprogramming, so we turn to Julia.

The following code generates a program like the one above, but always of the correct length. In the example, we generate a FizzBuzz program to print the numbers up to `65535`.

```julia
macro fizz_buzz(n::UInt64)
    ex = Expr(:block)
    for i in 1:n
        s = ""
        if i % 3 == 0
            s *= "Fizz"
        end
        if i % 5 == 0
            s *= "Buzz"
        end
        if length(s) == 0
            s = string(i)
        end
        push!(ex.args, :(println($s)))
    end
    ex
end

function main()
    @fizz_buzz(0x00000ffff)
end

main()
```

## Two-sum

This tricksy problem is designed to teach you about dynamic programming without over-burdening you with complications like practicality or applicability. Constaints:

- Given a list of integers and a target integer `t`, determine whether two integers in the list add up to `t`
- Program must execute in O(n) time, where `n` is the length of the list.

We'll use Python again. Since Python is an object-oriented programming language, it makes sense to start by creating a new class:

```python
class TwosumList:
  pass
```
As we iterate through our list, we need to check if the current value has a matching element somewhere in the list so that when the two are summed, we get the target. That is, we should *try* to involve the matching element somehow.
Obviously, this calls for a *try*-*except* clause. The only trick is to figure out how to shunt it into our OOP framework.

The solution I arrived at is to instantiate a `TwosumList` at the start of the iteration, which we helpfully call `twosum_list`. Every time an element `x` of the list is viewed, we try to call the method `twosum_list.hasa<target-x>()`.
If this method returns true, then `target-x` has been seen already and the function returns true. Otherwise, we get an `AttributeError` and we proceed with the loop after adding a `hasax` method to `twosum_list`. The hardest part is realizing that Python methods should have `-` in their names, so if `x<0`, we create a method `hasaminus|x|`.

For example, if the list is `[1,2,3,4,5]` and the target is `5`, then we instantiate `twosum_list`, then *try* `twosum_list.hasa4()`, which raises an `AttributeError`, and we add the method `hasa1` to `twosum_list`. Here's the complete code:

```python
class TwosumList:
    pass

def has_int_as_str(x: int) -> str:
    if x < 0:
        return "hasa" + "minus" + str(x)[1:]
    else:
        return "hasa" + str(x)

def twosum(vals: list[int], target: int) -> bool:
    twosum_list = TwosumList()
    for x in vals:        
        try:
            return getattr(twosum_list, has_int_as_str(target-x))()
        except AttributeError:
            setattr(twosum_list, has_int_as_str(x), lambda: True)
    return False
```
