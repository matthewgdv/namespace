# preface

In the spirit of the final line in the zen of python:Namespaces are one honking great idea -- let's do more of those!I've decided to write up a proposal I've had churning around in my head for a while now of adding a new soft keyword to the language:

`namespace`

For anyone who isn't aware, with the release of python 3.9 we now have a new and shiny parser that can backtrack, and therefore is capable of using soft-keywords. That means that the cost (in terms of backwards-compatibility) of implementing new (soft) keywords is much lower than before. The cost in terms of increasing language complexity is, of course, unchanged, and always works against new changes.
In spite of that, I think this proposal could have pretty significant value to the language, so here goes:
As an aside, it's here that I should warn you that this is pretty lengthy, and contains quite a few code examples. I've written this post as markup at () in case it doesn't render properly for anyone in your email client.

Okay, aside over. Onwards!

# the toy example (please don't judge the entire proposal by this!)

First off, for anyone who is familiar with similar namespace keywords in other languages like C#, rest easy. This is not that. At all. This new keyword would not be in any way connected to the import machinery, but rather would allow us to create new namespaces within:

1) a module
2) a class

The proposal is probably best initially explained using some code, so here goes.

In its simplest form:

```
# some_module.py

namespace constants:
  RETRY_DELAY_SECONDS = 10
  RETRY_ATTEMPTS = 6
  ...  # more constants here
  
  
constants.RETRY_ATTEMPTS == 6  # True

```

At this point you're probably asking yourself something along the lines of 'So how is this any different from using the class attributes of `class Constants` rather than `namespace constants`'?

The answer in this toy example is...

Not much, except that:

```
import sys

vars(sys.modules[__name__])["constants.RETRY_ATTEMPTS"] == 6  # True
```

Essentially, in the same way that:

```
MODULE_LEVEL_CONSTANT = 3
```

Is syntactic sugar for:

```vars(sys.modules[__name__])["MODULE_LEVEL_CONSTANT"] = 3```

A namespace directive would like this:

```
namespace constants:
  NAMESPACED_CONSTANT = True
  ANOTHER_CONSTANT = "hi"
```

Would be syntactic sugar for:

```
vars(sys.modules[__name__])["constants.NAMESPACED_CONSTANT"] = True
vars(sys.modules[__name__])["constants.ANOTHER_CONSTANT"] = "hi"
```

However, this would come with the added bonus that the non-dynamic method of access using dot-notation `constants.MODULE_LEVEL_CONSTANT` would then be actually valid, rather than raising a `NameError: name 'constants' is not defined`. Furthermore, The value actually exists in the module `globals`, rather than belonging to a class that just lives within the module `globals`, which would be the case if we had used a `class` statement instead.

Basically, using class blocks for namespacing (by leveraging the syntactic sugar for declaring class attributes within a class block) is ubiquitous in python, but I would argue that this is only the case _precisely_ because we lack a dedicated namespacing mechanism.

'But why...?' I can already hear you asking, '...do we need a new (soft) keyword for this when the way it is currently done works perfectly fine?'.

The point I will try my hardest to convince you of here is that there are potentially some pretty cool usability improvements to be had in python namesacing (particularly for classes) that you _didn't even know you wanted_. In addition, I suppose there might also be some (very, very minor) performance benefits from not having to create new classes for namespacing purposes, and a more formal declaration of intent by using `namespace` rather than `class`.

# 
