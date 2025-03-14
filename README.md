A growing list of things I dislike about Python.

There are workarounds for some of them (often half-broken and usually unintuitive)
and others may even be considered to be virtues by some people.

# Generic

## Zero static checking

The most critical problem of Python is complete lack of static checking
(it does not even detect missing variable definitions)
which increases debugging time and makes refactoring more time-consuming than needed.
This becomes particularly obvious when you run your app on a huge amount of data overnight,
just to detect missing initialization in some rarely called function in the morning.

There is [Pylint](https://www.pylint.org) but it is a linter (i.e. style checker)
rather than a real static analyzer so it is unable, by design, to detect
many serious errors which require dataflow analysis.
For example if fails on basic stuff like
* invalid string formatting (fixed [here](https://github.com/PyCQA/pylint/pull/2465))
* iterating over unsorted dicts (reported [here](https://github.com/PyCQA/pylint/issues/2467) with draft patch, rejected because maintainers consider it unimportant (no particular reasons provided))
* dead list computations (e.g. using `sorted(lst)` instead of `lst.sort()`)
* modifying list while iterating over it (reported [here](https://github.com/PyCQA/pylint/issues/2471) with draft patch, rejected because maintainers consider it unimportant (no particular reasons provided))
* etc.

## Global interpreter lock

[GIL](https://wiki.python.org/moin/GlobalInterpreterLock) precludes high-performant multithreading
which is suprising at the age of multicores.

## No type annotations

_Type annotations have finally been introduced in Python 3.5. Google has even developed a [pytype](https://github.com/google/pytype) type inferencer/checker but it seems to have [serious](https://github.com/google/pytype/issues/581) [limitations](https://github.com/google/pytype/issues/580) so it's unclear whether it's production-ready._

Lack of type annotations forces people to use hungarian notation
in complex programs (hello 90-s!).

# Language

## Negation operator syntax issue

This is not a valid syntax:
```
x == not y
```

## Unable to overload logic operators

It's not possible to overload `and`, `or` or `not` (which might have been handy to represent e.g. operations on set-like or geometric objects).
There's even a [PEP](https://www.python.org/dev/peps/pep-0335/) which was rejected
because Guido disliked particular implementation.

## Hiding type errors via (un)helpful conversions

It's very easy to make a mistake of writing `len(lst1) == lst2`
instead of intended `len(lst1) == len(lst2)`.
Python will (un)helpfully make it harder to find this error
by silently converting first variant to `[len(lst1)] * len(lst2) == lst2`
(instead of aborting with a type fail).

## Limited lambdas

For unclear reason lambda functions only support expressions
so anything that has control flow requires a local named function.

[PEP 3113](https://www.python.org/dev/peps/pep-3113/)
tries to persuade you that this was a good design decision:
```
While an informal poll of the handful of Python programmers I know personally ...
indicates a huge majority of people do not know of this feature ...
```

## Problematic operator precedence

The `is` and `is not` operators have the same precedence as comparisons
so this code
```
op.post_modification is None != full_op.post_modification is None
```
would rather unexpectedly evalute as
```
((op.post_modification is None) != full_op.post_modification) is None
```

## The useless self

Explicitly writing out `self` in all method declarations and calls
should not be needed. Apart from being an unnecessary boilerplate,
this enables another class of bugs:
```
class A:
  def first(x, *y):
    return x

a = A
print(a.first(1,2,3))  # Will print a, not 1
```

## Optional parent constructors

Python does not require a call to parent class constructor:
```
class A(B):
  def __init__(self, x):
    super().__init__()
    self.x = x
```
so when it's missing you'll have hard time understanding
whether it's been omitted deliberately or accidentally.

## Inconsistent syntax for tuples

Python allows omission of parenthesis around tuples in most cases:
```
for i, x in enumerate(xs):
  pass

x, y = y, x

return x, y
```
but not all cases:
```
foo = [x, y for x in range(5) for y in range(5)]
SyntaxError: invalid syntax
```

## No tuple unpacking in lambdas

It's not possible to do tuple unpacking in lambdas so instead
of concise and readable
```
lst = [(1, 'Helen', None), (3, 'John', '121')]
lst.sort(key=lambda n, name, phone: (name, phone))  # TypeError: <lambda>() missing 2 required positional arguments
```
you should use
```
lst.sort(key=lambda n_name_phone: (n_name_phone[1], n_name_phone[2]))
```

This seems to be intentional decision as tuple unpacking does work in Python 2.

## Inconsistent syntax of set literals

Sets can be initialized via syntactic sugar:
```
>>> x = {1, 2, 3}
>>> type(x)
<class 'set'>
```
but it breaks for empty sets:
```
>>> x = {}
>>> type(x)
<class 'dict'>
```

## Inadvertent sharing

It's too easy to inadvertently share references:
```
a = b = []
```
or
```
def foo(x=[]):  # foo() will return [1], [1, 1], [1, 1, 1], etc.
  x.append(1)
  return x
```
or even
```
def foo(obj, lst=[]):
  obj.lst = lst
foo(obj)
obj.lst.append(1)  # Hoorah, this modifies default value of foo
```

## Functions always return

Default return value from function (when `return` is omitted) is `None`.
This makes it impossible to declare subroutines which are not supposed
to return anything (and verify this at runtime).

## Automatic field insertion

Assigning a non-existent object field adds it instead of throwing an exception:
```
class A:
  def __init__(self):
    self.x = 0
...
a = A()
a.y = 1  # OK
```

This complicates refactoring because forgetting to update an outdated field name
deep inside your (or your colleague's) program will silently work,
breaking your program much later.

This can be overcome with `__slots__` but when have you seen them
used last time?

## Hiding class variables in instances

When accessing object attribute via `obj.attr` syntax Python will first search
for `attr` in `obj`'s instance variables. If it's not present,
it will search `attr` in class variables of `obj`'s class.

This behavior is reasonable and matches other languages.

Problem is that status of `attr` will change if we write it:
```
class A:
  x = 1

a = A()

# Here self.v means A.v ...
print(A.x) # 1
print(a.x) # 1

# ... and here it does not
a.x = 2
print(A.x) # 1
print(a.x) # 2
```

This leads to this particularly strange and unexpected semantics:
```
class A:
  x = 1

  def __init__(self):
    self.x += 1

print(A.x)  # 1

a = A()
print(A.x)  # 1
print(a.x)  # 2
```
To understand what's going on, note that `__init__` is interpreted as
```
def __init__(self):
  self.x = A.x + 1
```

## "Is" operator does not work for primitive types

No comments:
```
> 2+2 is 4
True
> 999+1 is 1000
False
```

This happens because [only sufficiently small integer objects are reused](https://stackoverflow.com/a/306353/2170527):
```
# Two different instances of number "1000"
>>> id(999+1)
140068481622512
>>> id(1000)
140068481624112

# Single instance of number "4"
>>> id(2+2)
10968896
>>> id(4)
10968896
```

## Inconsistent index checks

Invalid indexing throws an exception
but invalid slicing does not:
```
>>> a=list(range(4))
>>> a[4]
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
IndexError: list index out of range
>>> a[4:5]
[]
```

## Spurious `:`s

Python got rid of spurious bracing but introduced a spurious `:` lexeme instead.
The lexeme is not needed for parsing and its only purpose
[was](http://effbot.org/pyfaq/why-are-colons-required-for-the-if-while-def-class-statements.htm)
to somehow "enhance readability".

## Unnatural operator priority

Normally all unary operators have higher priority than binary ones but of course not in Python:
```
>>> not 'x' in ['x', False]
False
>>> (not 'x') in ['x', False]
True
>>> not ('x' in ['x', False])
False
```

A funny consequence of this is that `x not in lst` and `not x in lst` notations are equivalent.

## Weird semantics of `super()`

When you call `super().__init__` in your class constructor:
```
class B(A):
  def __init__(self):
    super(B, self).__init__()
```
it will NOT necessarily call constructor of superclass (`A` in this case).

Instead it will call a constructor of _some other_ class from class hierarchy
of `self`s class (if this sounds a bit complicated that's because it actually is).

Let's look at a simple example:
```
#   object
#   /    \
#  A      B
#  |      |
#  C      D
#   \    /
#      E

class A(object):
  def __init__(self):
    print("A")
    super().__init__()

class B(object):
  def __init__(self):
    print("B")
    super().__init__()

class C(A):
  def __init__(self, arg):
    print(f"C {arg}")
    super().__init__()

class D(B):
  def __init__(self, arg):
    print(f"D {arg}")
    super().__init__()

class E(C, D):
  def __init__(self, arg):
    print(f"E {arg}")
    super().__init__(arg)
```
If we try to construct an instance of `E` we'll get a puzzling error:
```
>>> E(10)
E 1
C 1
A
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
  File "<stdin>", line 4, in __init__
  File "<stdin>", line 4, in __init__
  File "<stdin>", line 4, in __init__
TypeError: __init__() missing 1 required positional argument: 'arg'
```
What happens here is that for diamond class hierarchies Python will
execute constructors in strange unintuitive order
(called MRO, explained in details [here](https://docs.python.org/3/howto/mro.html)).
In our case the order happens to be E, C, A, D, B.

As you can see poor `A` suddenly has to call `D` instead of expected `object`.
`D` requires `arg` of which `A` is of course not aware of, hence the crash.
So when you call `super()` you have no idea which class you are calling into,
nor what the expected `__init__` signature is.

When using `super()` ALL your `__init__` methods have to use keyword arguments only
and pass all of them to the caller:
```
class A(object):
  def __init__(self, **kwargs):
    print("A")
    if type(self).__mro__[-2] is A:
      # Avoid "TypeError: object.__init__() takes no parameters" error
      super().__init__()
      return
    super().__init__(**kwargs)

class B(object):
  def __init__(self, **kwargs):
    print("B")
    if type(self).__mro__[-2] is B:
      # Avoid "TypeError: object.__init__() takes no parameters" error
      super().__init__()
      return
    super().__init__(**kwargs)

class C(A):
  def __init__(self, **kwargs):
    arg = kwargs["arg"]
    print(f"C {arg}")
    super().__init__(**kwargs)

class D(B):
  def __init__(self, **kwargs):
    arg = kwargs["arg"]
    print(f"D {arg}")
    super().__init__(**kwargs)

class E(C, D):
  def __init__(self, **kwargs):
    arg = kwargs["arg"]
    print(f"E {arg}")
    super().__init__(**kwargs)

E(arg=1)
```
Notice the especially beautiful `__mro__` checks which I needed to avoid error in `object.__init_`
which just so happens to NOT support the kwargs convention
(see [super() and changing the signature of cooperative methods](https://stackoverflow.com/questions/56714419/super-and-changing-the-signature-of-cooperative-methods)
for details).

I'll let the reader decide how more readable and efficient this makes your code.

See [Python's Super is nifty, but you can't use it](https://fuhm.net/super-harmful/) for an in-depth discussion.

# Standard Libraries

## List generators fail check for emptiness

Thanks to active use of generators in Python 3 it became easier
to misuse standard APIs:
```
if filter(lambda x: x == 0, [1,2]):
  print("Yes")  # Prints "Yes"!
```
Surpisingly enough this does not apply to `range` (i.e. `bool(range(0))` returns `False` as expected).

## Argparse does not support negated flags

`argparse` does not provide automatic support for `--no-XXX` flags.

## Argparse has useless default formatter

By default formatter used by argparse
* won't display default option values
* will make code examples provided via `epilog` unreadable
  by stripping leading whitespaces

Enabling both features requires defining a custom formatter:
```
class Formatter(argparse.ArgumentDefaultsHelpFormatter, argparse.RawDescriptionHelpFormatter):
  pass
```

## Getopt does not allow parameters after positional arguments

```
>>> opts, args = getopt.getopt(['A', '-o', 'B'], 'o:')
>>> opts
[]
>>> args
['A', '-o', 'B']
>>> opts, args = getopt.getopt(['-o', 'B', 'A'], 'o:')
>>> opts
[('-o', 'B')]
>>> args
['A']
```

## Split and join disagree on argument order

`split` and `join` accept list and separator in different order:
```
sep.join(lst)
lst.split(sep)
```

## Builtins do not behave as normal functions

Builtin functions [do not support named arguments](https://stackoverflow.com/a/24463222/2170527)
e.g.
```
>>> x = {1: 2}
>>> x.get(2, 0)
0
>>> x.get(2, default=0)
Traceback (most recent call last):
  File "<stdin>", line 1, in <module>
TypeError: get() takes no keyword arguments
```

## Inconsistent naming

`string.strip()` and `list.sort()`, although named similarly (a verb in imperative mood),
have very different behavior: string's method returns a stripped copy whereas
list's one sorts object inplace (and returns `None`).

## Path concatenation may silently ignore inputs

`Os.path.join` will [silently drop preceding inputs on argument with leading slash](https://stackoverflow.com/questions/1945920/why-doesnt-os-path-join-work-in-this-case):
```
>>> print(os.path.join('/home/yugr', '/../libexec'))
/../libexec
```

Python docs mentions this behavior:
> If a component is an absolute path, all previous components are thrown away and joining continues from the absolute path component.

but unfortunately do not provide any reasons for such irrational choice. Throwing exception would be much less error-prone.

Google search for [why os.path.join throws away](https://www.google.com/search?q=why+os.path.join+throws+away+site%3Astackoverflow.com) returns 22K results at the time of this writing...

## Different rounding algorithm

Python uses a non-standard banker's rounding algorithm:
```
# Python3 (bankers rounding)
>>> round(0.5)
0
>>> round(1.5)
2
>>> round(2.5)
2
```
Apart from being counterintuitive, this is also different from most other languages (C, Java, Ruby, etc.).

# Name Resolution

## No way to localize a name

Python lacks lexical scoping i.e. there is no way to localize variable
in a scope smaller than a function.
This often hurts when renaming variables during code refactoring.
Forgetting to rename variable name in a single place, causes interpreter
to pick up an unrelated name from unrelated block 50 lines above or
from previous loop iteration.

This is especially inconvenient for one-off variables (e.g. loop counters):
```
for i in range(100):
  ...

# 100 lines later

for j in range(200):
  a[i] = 0  # Yikes, forgot to rename!
```

One could remove a name from scope via `del i` but this is considered too
verbose so noone uses it.

## Everything is exported

Similarly to above, there is no control over visibility (can't hide class methods,
can't hide module functions). You are left with a _convention_ to precede
private functions with `_` and hope for the best.

## Multiple syntaxes for aliasing imported module

Python allows different syntaxes for the aliasing functionality:
```
from mod import submod as X
import mod.submod as X
```

## Assignment makes variable local

Python scoping rules require that assigning a variable automatically declares it local.
This causes inconsistencies and weird limitations in practice. E.g. variable would
be considered local even if assignment follows first use:
```
global xxx
xxx = 0

def foo():
  a = xxx  # Throws UnboundLocalError
  xxx = 2
```
and even if assignment is never ever executed:
```
def foo():
  a = xxx  # Still aborts...
  if False:
    xxx = 2
```
This is particularly puzzling in long functions when someone accidentally
adds a local variable which matches name of a global variable used in other part
of the same function:
```
def value():
  ...

def foo():
  ...  # A lot of code
  value(1)  # Surprise! UnboundLocalError
  ...  # Yet more code
  for value in data:
    ...
```

Once you've lost some time debugging the issue, you can be overcome
for global variables by declaring their names as `global`
before first use:
```
def foo():
  global xxx
  a = xxx
  xxx = 2
```
But there are no magic keywords forvariables from non-global outer scopes so they
are essentially unwritable from nested scopes i.e. closures:
```
def foo():
  xxx = 1
  def bar():
    xxx = 2  # No way to modify xxx...
```

The only available "solution" is to wrap the variable into a fake 1-element array
(whaat?!):
```
def foo():
  xxx = [1]
  def bar():
    xxx[0] = 2
```

## Unreachable code that gets executed

Normally statements that belongs to false branches are not executed.
E.g. this code works:
```
if True:
  import re
re.search(r'1', '1')
```
and this one raises `NameError`:
```
if False:
  import re
re.search(r'1', '1')
```
This does not apply to `global` declarations though:
```
xxx = 1
def foo():
  if False:
    global xxx
  xxx = 42
foo()
print(xxx)  # Prints 42
```

## Relative imports are unusable

Relative imports (`from .xxx.yyy import mymod`) have many weird limitations
e.g. they will not allow you to import module from parent folder and
they will seize work in main script
```
ModuleNotFoundError: No module named '__main__.xxx'; '__main__' is not a package
```

A workaround is to use extremely ugly `sys.path` hackery:
```
import sys
import os.path
sys.path.append(os.path.join(os.path.dirname(__file__), 'xxx', 'yyy'))
```

Search for "python relative imports" on stackoverflow to see some really clumsy Python code
(e.g. [here](https://stackoverflow.com/questions/279237/import-a-module-from-a-relative-path)
or [here](https://stackoverflow.com/questions/1918539/can-anyone-explain-pythons-relative-imports)).
Also see [When are circular imports fatal?](https://datagrok.org/python/circularimports/)
for more weird limitations of relative imports with respect to circular dependencies.

# Performance

## Automatic optimization is hard

It's very hard to automatically optimize Python code
because there are far too many ways
in which program may change execution environment e.g.
```
  for re in regexes:
    ...
```
(see e.g. [this quote](https://youtu.be/2wDvzy6Hgxg?t=690) from Guido).
Existing optimizers (e.g. pypy) have to rely on idioms and heuristics.

# Infrastructure

## Syntax checking

Syntax error reporting in Python is extremely primitive.
In most cases you simply get `SyntaxError: invalid syntax`.

## Different conventions on OSes

Windows and Linux use different naming convention for Python executables
(`python` on Windows, `python2`/`python3` on Linux).

## Debugger is slow

Python debugging is super-slow (few orders of magnitude slower than
interpretation).

## No static analyzer

Already mentioned in [Zero static checking](#zero-static-checking).

## Unable to break on pass statement

Python debugger will [ignore breakpoints set on pass statements](https://stackoverflow.com/a/47626134/2170527).
Thus poor-man's conditional breakpoints like
```
if x > 0:
  pass
```
will silently fail to work, leaving a false impression that condition is always false.

## Semantic changes in Python 3

Most people blame Python 3 for syntax changes which break existing code
(e.g. making `print` a regular function) but the real problem is
_semantic_ changes as they are much harder to detect and debug.
Some examples
* integer division:
```
print(1/2)  # Prints "0" in 2, "0.5" in 3
```
* checking `filter`ed list for emptiness:
```
if filter(lambda x: x, [0]):
  print("X")  # Executes in 3 but not in 2
```
* order of keys in dictionary is random until Python 3.6
* different rounding algorithm:
```
# Python2.7
>>> round(0.5)
1.0
>>> round(1.5)
2.0
>>> round(2.5)
3.0

# Python3 (bankers rounding)
>>> round(0.5)
0
>>> round(1.5)
2
>>> round(2.5)
2
```

## Dependency hell

Python community does not seem to have a strong culture of preserving API backwards compatibility
or following [SemVer convention](https://semver.org/)
(which is hinted by the fact that there are no widespread tools for checking Python package API
compatibility). This is not surprising given that even minor versions of Python 3 itself
break old and popular APIs (e.g. [time.clock](https://bugs.python.org/issue31803)).
Another likely reason is lack of good mechanisms to control what's exported from module
(prefixing methods and objects with underscore is _not_ a good mechanism).

In practice this means that it's too risky to allow differences in minor (and even patch) versions
of dependencies.
Instead the most robust (and thus most common) solution is to fix _all_ app dependencies
(including the transitive ones) down to patch versions (via blind `pip freeze > requirements.txt`)
and run each app in a dedicated virtualenv or Docker container.

Apart from complicating deployment, fixing versions also
complicates importing module in other applications
(due to increased chance of conflicing dependencies)
and upgrading dependencies later on to get bugfixes and security patches.

For more details see the excellent ["Dependency hell: a library author's guide" talk](https://www.youtube.com/watch?v=OaBhcueqNqw)
and an alternative view in [Should You Use Upper Bound Version Constraints?](https://iscinumpy.dev/post/bound-version-constraints/)
