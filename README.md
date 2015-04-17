# Fail early, fail often
## How to Shoot Yourself in the Foot with Python

![Snake Fail](http://cl.jroo.me/z3/e/D/s/e/a.baa-snake-will-eat-itself.jpg)

This document doesn't list _bugs_ in Python, but rather unexpected behaviours. Of course, "unexpected behaviour" depends a lot on what you expect Python to do.

Please feel free to add more common pitfalls by sending a pull request or opening an issue!

### Arithmetics Fail.

Let's do some first grade arithmetics:

```python
>>> a = 2
>>> a * a is 4
True
```

Works as advertised. Let's see if python can handle slightly larger numbers, too:

```python
>>> a = 20
>>> a * a is 400
False
```

__What's happening here?__ Remember that everything in python is an object, even numbers. Also remember that `is` checks for _identity_, not _equality_. So `2 * 2 is 4` is the same as `id(2 * 2) == id(4)`. The reason this works for small numbers is that python creates singletons for integers from -9 to 255 on start-up because they're frequently used -- it's an implementation detail of CPython, not a language feature. However, when we compute `20 * 20`, a new object with value 400 is created, which is a different object than `400`.

> __How do avoid this issue:__ only use `is` to check if things are `True`, `False` or `None`. These are singletons (ie. every `False` in your code is the same object.)

### Class Property Fail

"In the wild, life is a constant battle to find enough to eat..."

```python
class Mammal(object):
    awkwardness = 0

class Platypus(Mammal):
    pass

class Dolphin(Mammal):
    pass
```

We create a mammal class and two sub-classes.

```python
>>> print(Mammal.awkwardness, Platypus.awkwardness, Dolphin.awkwardness)
0 0 0
```

Nothing too unexpected. Let's set the awkwardness of the platypus to a well-deserved 10:

```python
>>> Platypus.awkwardness = 10
>>> print(Mammal.awkwardness, Platypus.awkwardness, Dolphin.awkwardness)
0 10 0
```

All as expected. No remember that all mammals are basically __tubes__, and feel very self-conscious about being a mammal, too. Let's bump the awkwardness of mammals to 3:

```python
>>> Mammal.awkwardness = 3
>>> print(Mammal.awkwardness, Platypus.awkwardness, Dolphin.awkwardness)
3 10 3
```

__Why did the awkwardness of dolphins change? Dolphins are *cute*!__ We're dealing with class properties here. If untouched, they are simply references to the parent's class properties. When we set `Platypus.awkwardness = 10` we create a __new__ class property on the Platypus.

### Scope Fail

Here's one of my favourite Python party tricks (I'm an unpopular party guest). The setup:

```python
answer = 42

def ultimate_question_of_life():
    print(answer)
```

Now for the easy part:

```python
>>> ultimate_question_of_life()
42
```

Right on. But what if we try to one-up Douglas Adams?

```python
answer = 42

def ultimate_question_of_life():
    print(answer)
    answer += 1

ultimate_question_of_life()
```

Ouch:

```python
---------------------------------------------------------------------------
UnboundLocalError                         Traceback (most recent call last)
in <module>()
----> 7 ultimate_question_of_life()

in ultimate_question_of_life()
      3 def ultimate_question_of_life():
----> 4     print(answer)
      5     answer += 1

UnboundLocalError: local variable 'answer' referenced before assignment
```

__Alright, this fails.__ But wait a second, where does it fail? At the print statement that used to succeed in the example above! By adding a line _after_ a perfectly innocuous statement we make this statement suddenly break things! Madness!!

The problem here is that Python is, contrary to common misconception, not interpreted line-by-line. Instead, when we execute code (ie. import a module), Python computes scopes for all blocks, which variables are available inside the scope and where they point to. Since we assign `answer` inside the scope of `ultimate_question_of_life` (note that `+=` doesn't change the value of `answer`, but creates a new object!), we won't be able to refer to the `answer` that's declared outside that scope anymore.

### Oscar Speech Fail / Immutables Part I

As any academy award winning director knows, the most unforgivable of all faux pas is to forget to thank your spouse. Let's write a python script that takes care of our Oscar® speech:

```python
def oscar_speech(people_to_thank=[]):
    people_to_thank.append("my wife")
    for person in people_to_thank:
        print("I want to thank {}".format(person))
```

Alright, ready for the spotlight?

```python
>>> oscar_speech()
I want to thank my wife
>>> oscar_speech(["The Academy", "Lars von Trier"])
I want to thank The Academy
I want to thank Lars von Trier
I want to thank my wife
```

Great. Let's practice some more:

```python
>>> oscar_speech()
I want to thank my wife
I want to thank my wife
>>> oscar_speech()
I want to thank my wife
I want to thank my wife
I want to thank my wife
```

__Huh?__ The problem is that the list we pass on as the default argument only gets created once, at import time - no every time we call the function. So we end up appending our wife to the same list over and over again. This piece of code is identical to the one above and clarifies the issue:

```python
default_list = []
def oscar_speech(people_to_thank=default_list):
    people_to_thank.append("my wife")
```
