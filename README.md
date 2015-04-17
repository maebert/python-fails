# How to Shoot Yourself in the Foot with Python
## Common pitfalls and misunderstandings

![Snake Fail](http://cl.jroo.me/z3/e/D/s/e/a.baa-snake-will-eat-itself.jpg)

This document doesn't list _bugs_ in Python, but rather unexpected behaviours. Of course, "unexpected behaviour" depends a lot on what you expect Python to do.

Please feel free to add corrections, clarifications and more common pitfalls by sending a pull request or opening an issue!

### Contents

- [Arithmetics Fail.](#arithmetics-fail)
- [Class Property Fail](#class-property-fail)
- [Scope Fail](#scope-fail)
- [Oscar Speech Fail / Immutables Part I](#oscar-speech-fail--immutables-part-i)
- [Immutable Fail, part II](#immutable-fail-part-ii)
- [Cooking the Books Fail](#cooking-the-books-fail)
- [Integer Division Fail](#integer-division-fail)
- [Closure Fail](#closure-fail)


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

### Immutable Fail, part II

The pledge:

```python
flying_circus = ["Eric Idle", "Terry Gilliam"]

def casting_a():
    flying_circus.append("John Cleese")
    return flying_circus

def casting_b():
    flying_circus += ["Terry Jones"]
    return flying_circus
```

The turn:

```python
>>> casting_a()
['Eric Idle', 'Terry Gilliam', 'John Cleese']
```


The prestige:

```python
>>> casting_b()
---------------------------------------------------------------------------
UnboundLocalError                         Traceback (most recent call last)
in <module>()
----> 1 casting_b()

in casting_b()
      7 def casting_b():
----> 8     flying_circus += ["Terry Jones"]
      9     return flying_circus

UnboundLocalError: local variable 'flying_circus' referenced before assignment
```

Why does `list.append` succeed, but `list += [...]` fail? Because `list.append` alters the object, whereas `+=` tries to create a new object. Remember our scope fail above. `flying_circus += ["Terry Jones"]` is the same as `flying_circus = flying_circus + ["Terry Jones"]`. Because we will assign the variable `flying_circus` it won't be available in our scope until after the assignment. However before we try to assign it, we try to compute `flying_circus + ["Terry Jones"]`. For comparison,

```python
def casting_c():
    flying_circus_new = flying_circus + ["Terry Jones"]
    return flying_circus_new
```

will work perfectly fine.


### Cooking the Books Fail

Let's turn our attention to the use of Python in the scientific community. A frequent problem many scientists encounter is that their data doesn't _quite_ match the hypothesis. Instead of going through the arduous step of refining our hypothesis, we can just, you know, _tweak_ the data a little bit until it looks like what it was _supposed_ to look like to start with. But just to be safe, let's work on a copy of our data and not touch the original:

```python
data = {
    'x': [0,1,2,3],
    'y': [1,3,9,16]
}
actual_data = data.copy()
```

So, obviously the effect here is quadratic, right? And the `3` on the y-axis is just a tiny perturbance in our measurements. Let's fix that!

```python
>>> actual_data['y'][1] = 4
>>> print(actual_data)
{'y': [1, 4, 9, 16], 'x': [0, 1, 2, 3]}
```

Much better! Let's just make sure our original data is still the same.

```python
>>> print(data)
{'y': [1, 4, 9, 16], 'x': [0, 1, 2, 3]}
```

__Damn.__ When we created a copy of our data, we actually created a so-called __shallow copy__. This means that we create a new `dict` object, but we only copy the references of the keys and values. So the list we're altering in `baked_data` is actually the same list as the one in the original `data`.

Similarly, copying a list with `[:]`, as in `my_list = [1,2,3]; new_list = my_list[:]` only creates a shallow copy and can lead to similar unexpected effects.

> __How to avoid this issue:__ Use the `deepcopy` module.

### Integer Division Fail

Here's something that works, but is inadvisable.

```python
ducks = ["Donald", "Huey", "Dewey", "Louie"]
middle = len(ducks) / 2
print(ducks[middle])
```

As any adventurous and brave Pythonista does these days, you upgrade your code to Python 3, and suddenly:


```python
# In python3
ducks = ["Donald", "Huey", "Dewey", "Louie"]
middle = len(ducks) / 2
print(ducks[middle])
---------------------------------------------------------------------------
UnboundLocalError                         Traceback (most recent call last)
in <module>
----> 1 print(ducks[middle])

TypeError: list indices must be integers, not float
```

___Why?___ Because in python2, `/` has different meanings depending on wheather you feed in floats or integers. If both left and right side are integers, the result will also be an integer. In python3, `/` will __always__ produce a float, and of course you can't index a list with floats.

> __How to avoid this issue:__ Use `//` for integer devision.

### Closure Fail

This is a real-life example from production code I once wrote and now feel very ashamed for.

```python
def valid_password(pwd):
    return False  # In production, we'd do actual password validation here

def wrong_password_prompts():
    return [lambda pwd: "Password {} incorrect - {} attempts left".format(pwd, 3-i) for i in range(3)]

def get_password():
    for bad_attempt in wrong_password_prompts():
        pwd = input()
        if not valid_password(pwd):
            print(bad_attempt(pwd))
        else:
            return True
    return False
```

Let's let this sink in for a second. The crucial and most shameful part is `wrong_password_prompts`, where we return a list of three anonumous functions. The first function should return `"Password xyz incorrect - 3 attempts left"` when called with password `"xyz"`. The second function should return `"Password xyz incorrect - 2 attempts left"` and so on. Let's see what happens:

```python
>>> get_password()
xyz
Password xyz incorrect - 1 attempts left
swordfish
Password swordfish incorrect - 1 attempts left
shibboleth
Password shibboleth incorrect - 1 attempts left
```

__Why is there always only one attempt left?__ because the string we return only gets formatted when we call the anonymous functions. And the `i` we use to format it is actually just the `i` that gets  "left over" after the loop over `range(3)` is done - which has value `2`. Specifically, it leaked outside the scope.
