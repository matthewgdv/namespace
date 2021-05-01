# preface

In the spirit of the final line in the zen of python:

Namespaces are one honking great idea -- let's do more of those!

I've decided to write up a proposal I've had churning around in my head for a while now of adding a new soft keyword to the language:

`namespace`

For anyone who isn't aware, with the release of python 3.9 we now have a new and shiny parser that can backtrack, and therefore is capable of using soft-keywords. That means that the cost (in terms of backwards-compatibility) of implementing new (soft) keywords is much lower than before. The cost in terms of increasing language complexity is, of course, unchanged, and always works against new changes.

In spite of that, I think this proposal could have pretty significant value to the language, so here goes!

# the toy example

First off, for anyone who is familiar with similar namespace keywords in other languages like C#, rest easy. This is not that. At all. This new keyword would not be in any way connected to the import machinery, but rather would allow us to create new namespaces within:

1) modules
2) classes
3) locals

The proposal is probably best initially explained using some code:

In its simplest form:

```python
# some_module.py

namespace constants:
  RETRY_DELAY_SECONDS = 10
  RETRY_ATTEMPTS = 6
  ...  # more constants here
  
  
constants.RETRY_ATTEMPTS == 6  # True
```

At this point you're probably asking yourself something along the lines of 'So how is this any different from:

```python
class Constants:
  RETRY_DELAY_SECONDS = 10
  ...
```


The answer in this toy example is...

Not much, except that:

```python
import sys

vars(sys.modules[__name__])["constants.RETRY_ATTEMPTS"] == 6  # True
```

Essentially, in the same way that:

```python
MODULE_LEVEL_CONSTANT = 3
```

Is syntactic sugar for:

```python
vars(sys.modules[__name__])["MODULE_LEVEL_CONSTANT"] = 3
```

A namespace directive would like this:

```python
namespace constants:
  NAMESPACED_CONSTANT = True
  ANOTHER_CONSTANT = "hi"
```

Would be syntactic sugar for:

```python
vars(sys.modules[__name__])["constants.NAMESPACED_CONSTANT"] = True
vars(sys.modules[__name__])["constants.ANOTHER_CONSTANT"] = "hi"
```

However, this would come with the added bonus that the non-dynamic method of access using dot-notation `constants.MODULE_LEVEL_CONSTANT` would then be actually valid, rather than raising a `NameError: name 'constants' is not defined`. Furthermore, The value actually exists in the module `globals`, rather than belonging to a class that just lives within the module `globals`, which would be the case if we had used a `class` statement instead.

Basically, using class blocks for namespacing (by leveraging the syntactic sugar for declaring class attributes within a class block) is ubiquitous in python, but I would argue that this is only the case _precisely_ because we lack a dedicated namespacing mechanism.

'But why...?' I can already hear many readers asking, '...do we need a new (soft) keyword for this when the way it is currently done works perfectly fine?'.

The point I will try my hardest to convince you of here is that there are potentially some pretty awesome improvements to both **usability** and **clarity** to be had in python namespacing (particularly for classes) that you _didn't even know you wanted_. In addition, I suppose there might also be some performance benefits from not having to create new classes for namespacing purposes, and a more formal declaration of intent by using `namespace` rather than `class`.

# a real example

A fairly common design pattern for a libary class with several logical groupings of methods is to separate them out into accessor delegates, which actually implement the methods using a reference to the parent. This makes the class much more usable within an IDE, because rather than getting a huge amount of autocompletion suggestions all at once you instead just get a few small groups, and you can then descend into whichever group of methods you actually care about.

In the real world, this might look something like this:

```python
from __future__ import annotations
from collections import UserString
from some_string_manipulation_library import snake_case, camel_case, ...


class BaseAccessor:
    def __init__(self, parent: MyStr) -> None:
        self.parent = parent


class CasingAccessor(BaseAccessor):
    def snake(self) -> MyStr:
        return type(self.parent)(snake_case(self.parent))

    def camel(self) -> MyStr:
        return type(self.parent)(camel_case(self.parent))

    ...  # more casing-related methods go here


class RegexAccessor(BaseAccessor):
    ...  # regex-related methods go here


class MyStr(UserString):
    @property
    def case(self) -> CasingAccessor:
        return CasingAccessor(self)

    @property
    def re(self) -> RegexAccessor:
        return RegexAccessor(self)
```

For a hypothetical `str` subclass with additional utility methods.

They can now be used like:

```python
MyStr("hElLo wOrLd").case.snake()  # "hello_world"
```

This sort of situation would be precisely where `namespace` would shine, because the needlessly complicated (and less performant) block above becomes:

```python
from __future__ import annotations
from collections import UserString
from some_string_manipulation_library import snake_case, camel_case, ...

class MyStr(UserString):
    namespace case:
        def snake(self) -> MyStr:
            return type(self)(snake_case(self))

        def camel(self) -> MyStr:
            return type(self)(camel_case(self))

        ...

    namespace re:
        ...
```

So much cleaner!

Similarly to the toy example that dealt with modules, these methods can be found within the class `__dict__` with keys such as, for example `case.snake`. Keys that are perfectly valid in current python, and you've always been able to use them yourself dynamically by accessing the underlying module/class `dict`s, but without any syntactic sugar for setting and accessing them.

That is what this proposal ultimately boils down to if you were to summarize it in a single sentence:

**The addition of syntactic sugar for module/class/local variables that contain dots in their names.**

The critical point is that though a `namespace` block is indented and does have its own scope, it is not really a true language scope in the same way that a class definition is. Python only has 3 'real' scopes (module, class, function). It can be thought of more like the sort of scope of a `with` or `try` block.

This means that you can namespace out your methods while still having them actually be methods within that class (not methods for a completely different nested class). Basically, they retain acccess to their implicit first argument (`self` unless you are a heretic).

# more example use-cases

Let's imagine a class that can be constructed from many different sources and can be exported to many different destinations. Regardless of the abstracted implementation details of that class (probably using the factory design pattern or similar) The interface of the class might look something like this:

```python
from __future__ import annotations
import os
from credentials import AwsCredentials, AzureCredentials


class GenericData:
    ...  # class implementation
    
    def export_to_local_filesystem(self) -> None:
        ...
    
    def export_to_aws_s3(self) -> None:
        ...
    
    def export_to_azure_blob(self) -> None:
        ...    

    ...  # more export options
    
    @classmethod
    def from_local_filesystem(cls, path: os.PathLike) -> GenericData:
        ...
        
    @classmethod
    def from_aws_s3(cls, path: os.PathLike, bucket_name: str, credentials: AwsCredentials) -> GenericData:
        ...

    @classmethod    
    def from_azure_blob(cls, path: os.PathLike, container_name: str, credentials: AzureCredentials) -> GenericData:
        ...

    ...  # more construction options
```

For use like:

```python
GenericData.from_azure_blob(...).export_to_aws_s3(...)
```

However, using `namespace` it could end up as something like:

```python
from __future__ import annotations
import os
from credentials import AwsCredentials, AzureCredentials


class GenericData:
    ...  # class implementation


    namespace export_to:
        def local_filesystem(self) -> None:
            ...
    
        def aws_s3(self) -> None:
            ...
    
        def azure_blob(self) -> None:
            ...
    
        ...  # more export options

    namespace from_:
        @classmethod
        def local_filesystem(cls, path: os.PathLike) -> GenericData:
            ...
    
        @classmethod
        def aws_s3(cls, path: os.PathLike, bucket_name: str, credentials: AwsCredentials) -> GenericData:
            ...
    
        @classmethod
        def azure_blob(cls, path: os.PathLike, container_name: str, credentials: AzureCredentials) -> GenericData:
            ...
    
        ...  # more construction options

```

It would now instead be used like:

```python
GenericData.from_.azure_blob(...).export_to.aws_s3(...)
```

I think the gain in code clarity here is pretty huge. By grouping together related methods into namespaces like this the code becomes easier for others to read and maintain, and it becomes more convenient for users to consume with their fancy IDEs, since you can just type `GenericData.from_.` and will then get all the options for constructing new `GenericData` instances.

# a small point on python scopes

Though this is a relatively niche situation that relies on the user making some poor choices to begin with (i.e. most people will never see this), there are some rather... nasty quirks inherent to python scoping rules when a variable with the same name is reused across multiple different types of scopes (particularly if a class scope is involved). I will simply drop a link here and leave it at that. The main point I'm making is that using `namespace` avoids these issues altogether since it is not a true scope in the way that class definitions are.

http://lackingrhoticity.blogspot.com/2008/08/4-python-variable-binding-oddities.html

As an example of something that cannot currently be done with nested class scopes but could easily be done with the `namespace` keyword:

```python
class Outer:
  class Inner:
    host
    user = "ubuntu"
    
    # we can access some_attr from here if we want to
    
  class AnotherInner:
    Inner.some_attr
```

# conclusion

I wrote this proposal in a somewhat informal tone on a github repo as something that exists in a weird in-between place that isn't quite a normal-length post on python-ideas, but also isn't actually a full-blown PEP. I would be happy to write a PEP for this if this gained traction. Though for the actual implementation I would need a core dev with knowledge of the python internals to work with me, since I have no experience coding in C, and don't really know enough about the python internals.

If there is somewhere better I could cross-post this to please just let me know and I will do so.

If you've read so far, thanks a bunch! I pretty much wrote this up all in one go in a fit of equal parts stubbornness and madness. Thanks for making the python community as awesome as it is :)
