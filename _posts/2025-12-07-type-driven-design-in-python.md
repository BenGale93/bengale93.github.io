## Type driven design in Python

Type driven design is the idea of trying to put as much of the logic of your
program into your types. It allows you to enforce business logic at compile
time rather than runtime.

Python is famous for its dynamism. You can create classes at runtime and commit
all sorts of other crimes. What benefits could type driven design possibly have
for Python developers?

If you are a willing to use type hints and stay away from the more dynamic
features, quite a lot.

### Better Enums

Enums are a data type that represent a fix set, or enumeration, of values. This
can be useful for representing concepts like: days of the week, cardinal
directions, logging levels.

In many programming languages, Enum variants can hold arbitrary data. This
allows you to model defined sets of objects that might be represented by very
different structures.

In the Rust example below, we are modelling the situation where a section or
subsection needs to be extracted from a Markdown file. However, we don’t
necessarily know which. The behaviour of our code that searches the Markdown
file for the section or subsection depends on which we get.

```rust
enum MarkdownScope {
  SectionOrSubsection(String),
  Section(String),
  Subsection(String),
  SectionAndSubsection(String, String)
}
```

We can then define a method that searches for the particular section or
subsection.

```rust
impl MarkdownScope {
  fn find(&self, contents: String) -> String {
    // logic here
  }
}
```

Each variant can then be tackled separately, and the logic can precisely depend
on the nature of that variant.

This enum also makes it impossible to get into a state where neither a section
nor subsection is known. If we had used a struct like below. It would be
possible to initialise a struct where incompatible fields are None.

```rust
struct MarkdownScope {
  unknown: Option<String>,
  section: Option<String>,
  subsection: Option<String>,
}
```

Sadly, Python enums cannot hold data like section or subsection names, how can
we apply this powerful pattern? We can use typing.Union.

```python
from dataclasses import dataclass

@dataclass
class SectionOrSubsection:
    title: str

@dataclass
class Section:
    title: str

@dataclass
class Subsection:
    title: str

@dataclass
class SectionAndSubsection:
    section: str
    subsection: str

# Here we use Python +3.10 syntax to define the union
MarkdownScope = SectionOrSubsection | Section | Subsection | SectionAndSubsection
```

Each variant is defined as a dataclass and then grouped together in a union.
Any function or method that then operates on this data, should be type hinted
with MarkdownScope

Unlike Rust, you cannot then define methods on MarkdownScope. However, you can
instead rely on polymorphism and define a common method on each variant. If it
has an identical signature, you can then safely call it on an instance of
MarkdownScope. Try it yourself, and you will see, mypy won’t shout at you.

```python
@dataclass
class SectionOrSubsection:
    title: str

    def find(self, contents: str) -> str:
        """logic here"""

@dataclass
class Section:
    title: str

    def find(self, contents: str) -> str:
        """logic here"""

# etc ...

# Usage
def validate_structure(
    document: str,
    expected: list[MarkdownScope]
) -> list[str]:
    issues = []
    for scope in expected:
        text = scope.find(document)
        if not text:
            issues.append(f"Missing or empty: {scope}")
    return issues
```

The drawback of this approach is the verbosity, which could be even worse
without the datatclasses library. This is a small price to pay, especially
given how quickly we can write code these days.

#### An Aside on Result an Option

The two most widely used enums in Rust are the Result and Option types.
Functions in Rust that can go wrong return a Result rather than throw an
exception. This lets all callers know exactly how the function could go wrong,
not something you can do in Python. Whilst this pattern is extremely useful for
writing robust code, I would hesitate to recommend it in Python due to it not
being a built-in part of the language. If you are interested in trying it out
and potentially annoying your colleagues, check out:
[Returns](https://github.com/dry-python/returns?tab=readme-ov-file#result-container)

Options are essentially a much better version of T | None. The difference is
that the

Option<T> type in Rust offers a bunch of helper methods to make your code nice
and concise. Again, the Returns library offers something similar.

### Parse, don't validate

Say you have a list in your program that should always be non-empty. Depending
on how defensive you are being, you can either just assume this:

```python
non_empty = [1, 3, 10]

head = non_empty[0] # yolo
```

Or you can verify this assumption everywhere you are about to access the first
element. This way you can at least control the error message and make it more
descriptive.

```python
non_empty = [1, 3, 10]

assert non_empty, "That list we expected to be non-empty isn't..."

head =  non_empty[0]
```

Clearly, making this assertion everywhere you may want to access the first
element is incredibly prone to issues. What if you want to change the error
message? What if you forget to use it? Instead, let’s encapsulate
“non-emptiness” into the type system. Then, when we parse our data into the
relevant object, we can be confident it always upholds this property. As best
we can in Python, of course.

For this pattern, I tend to reach for Pydantic models. In our example:

```python
import typing as t

from pydantic import RootModel, model_validator

# Python 3.12 generics syntax being used here
class NonEmptyList[T](RootModel[list[T]]):

    @model_validator(mode="after")
    def non_empty(self) -> t.Self:
        if not self.root:
            msg = "Expected at least one element"
            raise ValueError(msg)
        return self

works_str = NonEmptyList(["test", "here"])
print(works_str)

works_num = NonEmptyList([1, 3, 4])
print(works_num)

NonEmptyList([]) # Errors
```

Assuming you don’t mutate the list or mess with the root attribute, you are
safe to assume this list is non-empty. In fact, you can use Pydantic’s
`ConfigDict(frozen=True)` to get some faux immutability. As always, Python
enforces nothing, so there has to be a degree of discipline in your codebase
for this to work. If you want the list to be mutable, you can add methods to
your `RootModel` that continue to enforce the invariant.

```python
import typing as t

from pydantic import RootModel, model_validator

class NonEmptyList[T](RootModel[list[T]]):

    @model_validator(mode="after")
    def non_empty(self) -> t.Self:
        if not self.root:
            msg = "Expected at least one element"
            raise ValueError(msg)
        return self

    def remove(self, value: T) -> None:
        if len(self.root) == 1:
            msg = f"Cannot remove: '{value}' the list is of length one."
            raise ValueError(msg)
        self.root.remove(value)

works_num = NonEmptyList([1, 3, 4])

works_num.remove(1)
works_num.remove(3)
works_num.remove(4) # Errors
```

For all the methods that can break the invariant, you check your condition and
then perform the equivalent method call on the `root` attribute. This has the
advantage of collecting all your checks into one place in the codebase, rather
than scattering them where the type is being used. Once again, this can involve
a lot of boilerplate as you duplicate common methods.

For a much deeper dive on this topic, please read the article that inspired
this section [Parse, don’t
validate](https://lexi-lambda.github.io/blog/2019/11/05/parse-don-t-validate/).

### Enriching Standard Library Types

In the previous example, we enriched the list type by forcing it to be
non-empty. We can go even further with this concept by enriching domain types
that are special lists, dictionaries, integers etc.

What do I mean by enrich? Have you ever wished you could attach a method to a
list of specific objects, particularly objects you defined in your app or
library? This pattern allows you to couple that behaviour with the container of
objects, rather than it being a free floating function.

Aside: in Rust, you can achieve this with traits.

In our example, we are modelling a Document as a list of Sections. Rather than
using the built-in list and having a free function get_section we can inherit
from list and add it as a method. When a user then runs help on this type, they
can see all the available methods, rather than having to dig around the module
or library.

```python
from dataclasses import dataclass

@dataclass
class Section:
    title: str
    content: str

class Document(list[Section]):
    def get_section(self, title: str) -> Section:
        for section in self:
            # Assumes uniqueness so maybe we need some validation too
            if section.title == title:
                return section
        msg = f"Section title: '{title}' not present in the document."
        raise ValueError(msg)

d = Document([Section("Test", "Test"), Section("Other", "Other")])
print(d.get_section("Test"))

d.append(Section("New", "New"))
print(d)
```

It also gives us all the list methods and dunder methods for free, unlike the
pattern above. You can of course override any existing methods you want to
change the behaviour of. This is almost certainly overkill for a single method,
but it gives you the flexibility to add as many as you want.

### Conclusion

Python is an incredibly flexible language with a vast ecosystem. However,
sometimes that flexibility can cause issues. If you can’t/won’t leave the
ecosystem behind, try adopting some of the patterns explained above to make
your code more robust, reliable, and readable.
