---
name: tighten
description: Use a declarative style, locality of reference, and more to simplify code and make it more readable
---

# Tighten: Declarative Style, Locality, and Simplification

The following guidelines minimize two metrics from *Code Complete* by Steve McConnell that determine how much back-and-forth a reader must do to understand code:

- **span**, the number of lines between successive references to a variable, and
- **live time**, the total number of lines between a variable's first and last reference.

## Function Signatures

Add docstrings to functions and classes only, following a tweaked NumPy convention. Mimic the type annotation from the function signature in the docstring. Unlike NumPy style, don't put a space before the colon. Default values in docstrings must match the code.

```python
def greet(name: str, greeting: str | None = 'Hello'):
    """
    Greet a person by name.

    Parameters
    ----------
    name: str
        The person's name
    greeting: str | None = 'Hello'
        The greeting to use
    """
```

Define `main()` as a regular function whose parameters are the CLI interface, dispatched via `fire` or `omegaconf` — never `argparse`. Use modern Python 3.10+ type annotations (`X | None`, builtins over `typing`).

> `references/function-signatures.md` — structured/nested parameter docstrings, `fire`/`omegaconf` dispatch examples, LaTeX math in docstrings with raw strings

## Creating Variables

Spell out variable names in full. Use `k` as the loop variable when iterating over columns. Use the snake_case form of a class name when naming a new variable. Don't use adjectives unless necessary to distinguish (`columns` not `original_columns`). Never create a variable unless it is referenced more than once — inline single-use values directly. Same for functions and classes: don't define them unless called from more than one place; use a lambda for a one-off function. Define every variable as close as possible to where it is first used (locality of reference), using `try/except NameError` for lazy initialization inside loops. Variable shadowing is intentional.

```python
# walrus operator with try/except for lazy init and iterator cycling
for step in range(steps):
    try:
        batch = next(batches)
    except (NameError, StopIteration):
        batch = next(
            batches := iter(data_loaders['train'])
        )
```

> `references/creating-variables.md` — inlining single-use values, locality examples, variable shadowing with groupby

## Declarative Style

Compose expressions inline rather than assigning them to intermediate variables. Cram as many nested expressions as possible into a single statement, eliminating named intermediaries that inflate span and live time. Don't create a variable when it is an O(1) operation to recalculate the value at the point of use.

```python
# bad — imperative: three intermediate variables with nonzero span
paths = list_paths(directory)
frames = [load(path) for path in paths]
result = pd.concat(frames, ignore_index=True)

# good — declarative: one expression tree, zero intermediate variables
result = (
    pd
    .concat(
        [
            load(path)
            for path in list_paths(directory)
        ],
        ignore_index=True
    )
)
```

## Indentation

Indentation visually communicates the logical nesting depth of an expression. Each level corresponds to one level deeper in the expression tree.

```python
# good — indentation reveals the expression tree
result = (
    pd
    .concat(
        [
            frame.assign(
                score=frame['value'].rank()
            )
            for frame in frames
        ]
    )
    .reset_index(drop=True)
)

# bad — indentation obscures the nesting structure
result = (pd.concat([frame.assign(
    score=frame['value'].rank()) for frame in frames
]).reset_index(drop=True))
```

## Creating Commands

Don't create a statement unless it is necessary. Pass options through constructors and initializers to collapse multiple statements into one.

```python
# bad — mutating after creation
frame = pd.DataFrame()
frame['input_ids'] = input_ids
frame['labels'] = labels

# good — pass data through the constructor
frame = pd.DataFrame(
    {
        'input_ids': input_ids,
        'labels': labels
    }
)
```

## DRY

Never duplicate code. If two blocks differ only in the data they operate on, factor the shared logic into a loop or generator expression.

```python
# bad — identical logic duplicated
source_embeddings = torch.tensor(source_frame['embedding'].tolist()).half().to(rank)
destination_embeddings = torch.tensor(destination_frame['embedding'].tolist()).half().to(rank)

# good — shared logic expressed once
source_embeddings, destination_embeddings = (
    torch
    .tensor(
        frame['embedding'].tolist()
    )
    .half()
    .to(rank)
    for frame in (source_frame, destination_frame)
)
```

## Pandas

Always pass `as_index=False` to `groupby` and shadow-rename the group variable to `frame`. Use `.to_dict('records')` instead of `.iterrows()`. Use `[lambda frame: ...]` for inline filtering when method-chaining. Use `.sample(frac=1)` to shuffle. Use `.assign()` to add or overwrite columns in a chain.

```python
# groupby with shadow rename
for _, frame in frame.groupby('category', as_index=False):
    process(frame)

# inline lambda filtering
result = (
    frame
    [lambda frame: frame['score'] > 0.5]
    .reset_index(drop=True)
)

# iterate rows
for record in frame.to_dict('records'):
    process(record)
```

## Attribute and Method Chaining

When a chain exceeds 120 characters, break it so each line contains at most one attribute access or function call. The dot goes on the new line. Opening parenthesis sits at the end of its line (KR style); closing parenthesis sits on its own line at the same indentation level as the call that opened it.

```python
# good — module chain
result = (
    pd
    .concat(frames)
    .sample(frac=1)
)

# bad — closing paren and dot on the same line
foo(
    ...
).bar(
    x=3
).baz()
```

> `references/chaining.md` — attribute chains on own lines, module/submodule chains

## Parentheses

Never nest more than two opening delimiters (`(`, `[`, `{`) on the same line. When a third level would appear, break the inner expression to the next line.

```python
# good
foo(
    bar(
        baz(x=1, y=2, z=3)
    )
)

# bad — three levels on one line
foo(bar(baz(x=1, y=2, z=3)))
```

## File Layout

Order module bodies: imports, type definitions, class/function definitions, then commands and assignments. Type annotations used by subsequent classes go before them; all other variables go at the bottom. Separate the import section from the rest with two blank lines. Separate each top-level definition with two blank lines. Organize imports into four alphabetically-sorted blocks: builtins, third-party, private, local. If you see `import foo.bar`, `foo` is also imported; don't add a redundant `import foo`.

```python
import sys
from dataclasses import dataclass
from pathlib import Path

import pandas as pd
from tqdm import tqdm

import my_private_package
import my_private_package.utils

import this_current_package.models
import this_current_package.io
```

> `references/file-layout.md` — full module layout example with type definitions, classes, functions, and lambdas

## Miscellaneous

F-strings exclusively (single quotes). Comments are sparse, explain "why" not "what", lowercase, no trailing period. One arithmetic operation per line with operator at end. No trailing commas. Minimal vertical whitespace. Assertions for preconditions. Tests at module bottom behind a pytest guard with local test data. Multiprocessing for parallelizable work. `'a b c'.split()` for string lists. Type annotations on signatures only.

```python
# no trailing commas
config = {
    'name': 'bob',
    'age': 12
}

# string lists
columns = 'name age email'.split()

# type annotations on signatures only — not inline
def load(path: str) -> pd.DataFrame:
    buffer = []  # not buffer: list[torch.Tensor] = []
```

> `references/miscellaneous.md` — arithmetic line breaks, multiprocessing, assertions, pytest guard, comment style
