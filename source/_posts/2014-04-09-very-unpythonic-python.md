---
layout: post
title: "Very Unpythonic Python"
date: 2014-04-09 03:09:54 -0400
comments: true
categories: 'python'
---

##FizzBuzz

When I applied to [Hacker School](http://www.hackerschool.com), one of the application question was FizzBuzz:

> Write a program that prints out the numbers 1 to 100 (inclusive). If the number is divisible by 3, print *Fizz* instead of the number. If it's divisible by 5, print *Buzz*. If it's divisible by both 3 and 5, print *FizzBuzz*. You can use any language.

(Hacker School has since updated the question slightly, likely to make it more difficult to solve via Google. I've left out the updated version intentionally to minimize my effect on Googlability.)

This problem is fairly straight forward, and a good bite size problem to use as an example for different languages and programming styles; it's similar to "Hello, World!" and "Fibonacci".

## Unpythonic Python

I looked at this problem today while showing a friend the Hacker School application, and started thinking about the many ways to tackle the problem just in Python alone. Python, specifically with [PEP 8](http://legacy.python.org/dev/peps/pep-0008/), lays out the ideal way to write *Pythonic Python*. But Python doesn't enforce *Pythonic Python*, so I started thinking, what other kinds of Python could I write this in.

**Warning: Very Unpythonic Python ahead.** However, all code should work (with Python 2.7.5).

Feel free to [tweet me](https://twitter.com/taubeneck) or comment with suggested fixes, or new additions.

### Pythonic Python

~~~
def fizzbuzz(number):
    if number % 3 == 0 and number % 5 == 0:
        return 'FizzBuzz'
    elif number % 3 == 0:
        return 'Fizz'
    elif number % 5 == 0:
        return 'Buzz'
    else:
        return number

for number in range(1, 101):
    print fizzbuzz(number)

~~~

### Lispy Python

~~~
fizzbuzz = lambda n: 'FizzBuzz' if n % 3 == 0 and n % 5 == 0 else None
fizz = lambda n: 'Fizz' if n % 3 == 0 else None
buzz = lambda n: 'Buzz' if n % 5 == 0 else None

print '\n'.join(map(lambda n: fizzbuzz(n) or fizz(n) or buzz(n) or str(n), range(1, 101)))
~~~

### Clojurly Python

~~~
def fizzbuzz(n):
    return 'FizzBuzz' if n % 3 == 0 and n % 5 == 0 else None

def fizz(n):
    return 'Fizz' if n % 3 == 0 else None

def buzz(n):
    return 'Buzz' if n % 5 == 0 else None

def fizz_andor_maybenot_buzz(n):
    print fizzbuzz(n) or fizz(n) or buzz(n) or str(n)

map( fizz_andor_maybenot_buzz, xrange(1, 101))
~~~

### Javacious Python

~~~
import sys


class FizzBuzz(object):
    def __init__(self, n):
        if n % 15 == 0:
            self.value = 'FizzBuzz';
        elif n % 3 == 0:
            self.value = 'Fizz';
        elif n % 5 == 0:
            self.value = 'Buzz';
        else:
            self.value = str(n);

    def __repr__(self):
        return self.value;

if __name__ == '__main__':
    n = 101;
    for i in range(n):
        sys.stdout.write(str(FizzBuzz(i))+'\n');
~~~

###C-ly Python

~~~
def main():
    n, i, N, value = 0, 0, [], '';
    n = 101;
    N = range(1, n);

    for i in N:
        if i % 15 == 0:
            value = 'FizzBuzz';
        elif i % 3 == 0:
            value = 'Fizz';
        elif i % 5 == 0:
            value = 'Buzz';
        else:
            value = str(i);
        print value;

    return 0;

main();
~~~

### Javascripty Python

~~~
import sys

n = 101;
N = range(1, n);

for i in N:
    if (i % 15 == 0):
        sys.stdout.write('FizzBuzz\n');
    elif (i % 3 == 0):
        sys.stdout.write('Fizz\n');
    elif (i % 5 == 0):
        sys.stdout.write('Buzz\n');
    else:
        sys.stdout.write(str(n)+'\n');
~~~