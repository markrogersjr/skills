---
name: tighten
description: Use a declarative style, locality of reference, and more to simplify code and make it more readable
---

# Tighten: Declarative Style, Locality, and Simplification

The following guidelines minimize two metrics from *Code Complete* by Steve McConnell that determine how much back-and-forth a reader must do to understand code:

- **span**, the number of lines between successive references to a variable, and
- **live time**, the total number of lines between a variable's first and last reference.

## Function Signatures

### Docstrings

Add docstrings to functions and classes only. Do not add module-level docstrings to `__init__.py` files.

Treat docstrings as Markdown. The description section supports LaTeX math using `$$...$$` for display equations and `$...$` for inline math. Use raw strings (`r"""..."""`) for docstrings containing LaTeX to avoid backslash escaping issues.

Follow the NumPy convention with some tweaks. Start with one or more paragraphs describing the function. Each paragraph should be a logical unit with multiple sentences, just like normal English writing. Don't put each sentence in its own paragraph. Break lines at 120 characters. Use backticks around code identifiers like `frozendict`, `dict`, `None`, etc. Then add a Parameters section (and optionally a Returns section).

For parameters, mimic the type annotation from the function signature. Unlike NumPy style, don't put a space before the colon. Default parameter values in docstrings must always match the actual defaults in the code.

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

For structured/nested parameters, define the parent with a colon and indent the children. This is similar to YAML but not technically YAML:

```python
def send_message(config: dict):
    """
    Send a message to a recipient.

    Parameters
    ----------
    recipient:
        name: str
            The recipient's name
        email: str
            The recipient's email address
    message:
        subject: str
            The message subject
        body: str
            The message body
    """
```

Use a period after description sentences. Parameter descriptions should only end with a period if there's more than one sentence.

### `main` Functions

Define `main()` as a regular function whose parameters are the command-line arguments. The function signature *is* the CLI interface, so the parameter names, types, and defaults define what the user can pass on the command line. This means `main()` follows all the same docstring and type-annotation conventions as any other function. Use `fire` or `omegaconf` to dispatch CLI arguments to the function's parameters. Never use `argparse`, which duplicates the information already expressed in the function signature.

```python
# bad — argparse duplicates the function signature
import argparse

def main(path: str, batch_size: int | None = 32, learning_rate: float | None = 1e-3):
    ...

parser = argparse.ArgumentParser()
parser.add_argument('--path', type=str, required=True)
parser.add_argument('--batch_size', type=int, default=32)
parser.add_argument('--learning_rate', type=float, default=1e-3)
args = parser.parse_args()
main(args.path, args.batch_size, args.learning_rate)
```

```python
# good — the function signature is the CLI interface
import fire
from omegaconf import OmegaConf


def main(
    path: str,
    batch_size: int | None = 32,
    learning_rate: float | None = 1e-3
):
    """
    Train a model on the given dataset.

    Parameters
    ----------
    path: str
        Path to the training data
    batch_size: int | None = 32
        Number of samples per batch
    learning_rate: float | None = 1e-3
        Optimizer learning rate
    """
    ...


# fire
fire.Fire(main)

# omegaconf
main(
    **OmegaConf.to_container(
        OmegaConf.from_cli()
    )
)
```

### Type Annotations

Use modern Python 3.10+ syntax for type annotations in function signatures. Use builtins (`list`, `dict`, `tuple`) instead of their `typing` analogs, `X | Y` instead of `Union[X, Y]`, and `X | None` instead of `Optional[X]`. Use `X | None` for any parameter with a default value, including booleans.

```python
# bad
from typing import Optional, Union, List, Dict
def process(
    data: List[str],
    limit: Optional[int] = None,
    mode: Union[str, int] = 'default',
    disable: Optional[bool] = True
) -> Dict[str, int]:

# good
def process(
    data: list[str],
    limit: int | None = None,
    mode: str | int = 'default',
    disable: bool | None = True
) -> dict[str, int]:
```

## Creating Variables

Spell out variable names in full (`temporary_directory` not `temp_dir`, `current_index` not `idx`, `output_path` not `out`). Don't use adjectives in variable names unless necessary to distinguish from other variable names (`columns` not `original_columns`). When naming a new variable, use the snake_case form of its class name. Use `k` as the loop variable when iterating over columns.

Never create a variable unless it is referenced more than once. If a value is only used once, inline it directly into the expression where it is consumed. A variable that is assigned and then immediately passed to a single call site is wasted: it forces the reader to jump back to the definition to understand what it holds, and it adds a line of code that communicates nothing. The same applies to functions and classes: don't define them unless they are called from more than one place. Use a lambda for a one-off function.

```python
# bad — config is only used once
config = {'lr': 0.001, 'weight_decay': 1e-5}
optimizer = torch.optim.Adam(model.parameters(), **config)

# good
optimizer = torch.optim.Adam(
    model.parameters(),
    lr=0.001,
    weight_decay=1e-5
)
```

```python
# bad — normalize is only called once
def normalize(frame):
    return frame.assign(score=frame['score'] / frame['score'].max())
result = normalize(frame)

# good
result = frame.assign(score=frame['score'] / frame['score'].max())
```

Define every variable as close as possible to where it is first used. This principle, called locality of reference, minimizes the distance the reader must scan to find a variable's meaning. Instead of initializing a variable before a loop, initialize it on the first iteration:

```python
# bad
score = float('inf')
for step in range(steps):
    ...
    score = min(score, current_score)

# good
for step in range(steps):
    ...
    try:
        score
    except NameError:
        score = float('inf')
    score = min(score, current_score)
```

The `try/except NameError` pattern is the preferred way to lazily initialize a variable inside a loop. It also combines naturally with `StopIteration` for iterator exhaustion:

```python
# bad
batches = iter(data_loaders['train'])
for step in range(steps):
    try:
        batch = next(batches)
    except StopIteration:
        batches = iter(data_loaders['train'])
        batch = next(batches)

# good
for step in range(steps):
    try:
        batch = next(batches)
    except (NameError, StopIteration):
        batch = next(
            batches := iter(data_loaders['train'])
        )
```

Variable shadowing is intentional for conciseness. Do not refactor code to avoid shadowing unless it causes a bug. Shadowing builtins (`callable`, `object`, `format`) as parameter names is acceptable.

```python
# bad — renamed to avoid shadowing
for _, filtered_frame in frame.groupby('category', as_index=False):
    process(filtered_frame)

# good — shadow frame intentionally
for _, frame in frame.groupby('category', as_index=False):
    process(frame)
```

## Declarative Style

The declarative style is to be contrasted with the imperative style, where you incrementally define new variables to build up a complex expression. When a variable is referenced, the reader must scroll back to find its definition, understand it, and then scroll forward to resume reading. The declarative style minimizes span and live time by composing expressions inline rather than assigning them to intermediate variables.

Practically, this means cramming as many nested expressions as possible into a single statement. Instead of breaking a computation into named intermediaries, compose it as one expression tree. Don't create a variable when it is an O(1) operation to recalculate the value at the point of use.

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

```python
# bad — imperative: four sequential assignments
raw_text = fetch_document(url)
cleaned_text = raw_text.strip().lower()
tokens = cleaned_text.split()
count = len(tokens)

# good — declarative: single nested expression
count = len(
    fetch_document(url)
    .strip()
    .lower()
    .split()
)
```

## Indentation

Indentation is not just for satisfying Python's syntactic requirements. It visually communicates the logical nesting depth of an expression. Each indentation level corresponds to one level deeper in the expression tree. When reading indented code, the reader can immediately see that an inner expression is an argument or component of the outer expression. This works hand-in-hand with the declarative style: when you compose deeply nested expressions, consistent indentation is what makes them readable.

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

Don't create a statement unless it is necessary. If an attribute is available as a constructor parameter, set it at instantiation rather than in a follow-up assignment. More generally, pass options through constructors and initializers to collapse multiple statements into one. The declarative style and attribute and method chaining are also effective tools for reducing command count: a chain of attribute accesses and method calls is a single statement, and nesting expressions eliminates the assignments that would otherwise precede them.

```python
# bad — three statements to configure one object
optimizer = torch.optim.Adam(model.parameters())
optimizer.lr = 0.001
optimizer.weight_decay = 1e-5

# good — one statement
optimizer = torch.optim.Adam(
    model.parameters(),
    lr=0.001,
    weight_decay=1e-5
)
```

```python
# bad — mutating the frame after creation
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

Never duplicate code. If two blocks differ only in the data they operate on, factor the shared logic into a loop or generator expression. Use nested control flow to handle variations within a single code path rather than copying the path and modifying each copy.

```python
# bad — identical logic duplicated for two variables
source_embeddings = (
    torch
    .tensor(
        source_frame['embedding'].tolist()
    )
    .half()
    .to(rank)
)
destination_embeddings = (
    torch
    .tensor(
        destination_frame['embedding'].tolist()
    )
    .half()
    .to(rank)
)

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

Always pass `as_index=False` to `groupby`. Shadow-rename the group variable to `frame` in the loop body so that downstream code reads naturally:

```python
# bad
for name, group in frame.groupby('category'):
    process(group)

# good
for _, frame in frame.groupby('category', as_index=False):
    process(frame)
```

Always use `.to_dict('records')` to iterate over rows. Never use `.iterrows()`:

```python
# bad
for index, row in frame.iterrows():
    process(row)

# good
for record in frame.to_dict('records'):
    process(record)
```

Use `[lambda frame: ...]` for inline filtering when method-chaining. This avoids creating a temporary mask variable:

```python
# bad
mask = frame['score'] > 0.5
filtered = frame[mask]
result = filtered.reset_index(drop=True)

# good
result = (
    frame
    [lambda frame: frame['score'] > 0.5]
    .reset_index(drop=True)
)
```

Use `.sample(frac=1)` to shuffle a frame. Use `.assign()` to add or overwrite columns in a chain.

## Attribute and Method Chaining

When an attribute or method chain is too long to fit on a single line (120 characters), break it so that each line contains at most one attribute access or function call. The dot goes on the new line, immediately before the method name. The opening parenthesis of the preceding call sits at the end of its line (Kernighan-Ritchie braces), and the closing parenthesis sits on its own line at the same indentation level as the call that opened it.

```python
# good
foo(
    ...
)
.bar(x=3)
.baz()

# bad — closing paren and dot on the same line
foo(
    ...
).bar(
    x=3
).baz()
```

If the entire chain fits on one line without exceeding 120 characters, there is no need to break it up:

```python
# good — fits on one line
result = text.strip().lower().split()
```

When a chain includes attribute accesses, each attribute goes on its own line just like method calls:

```python
# good
source = (
    Path(
        importlib.import_module(module).__file__
    )
    .parent
    .parent
)

# bad — attribute accesses crammed onto one line
source = Path(importlib.import_module(module).__file__).parent.parent
```

When a chain starts from a module or submodule, put each level on its own line:

```python
# good
result = (
    pd
    .concat(frames)
    .sample(frac=1)
)

# good
batch = (
    pyarrow
    .parquet
    .ParquetFile(path)
    .iter_batches()
)
```

## Parentheses

Never nest more than two opening delimiters (`(`, `[`, `{`) on the same line. When a third level of nesting would appear, break the inner expression to the next line.

```python
# good
foo(
    bar(
        baz(x=1, y=2, z=3)
    )
)

# bad — three levels on one line
foo(bar(baz(x=1, y=2, z=3)))

# bad — three opening delimiters on the first line
foo(bar(baz(
    x=1,
    y=2,
    z=3
)))
```

## File Layout

The body of a module is ordered: imports, type definitions, class and function definitions, then commands and assignments. Type annotations that are used by subsequent classes and functions may appear before them; all other variables go at the bottom. Organize imports into four blocks separated by blank lines: builtins, third-party, private, and local. Each block is sorted alphabetically. If you see `import foo.bar`, that means `foo` is also imported; do not add a redundant `import foo`. Separate the import section from the rest of the module with two blank lines. Separate each top-level class and function definition with two blank lines.

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


Record = dict[str, str | int | float]


@dataclass
class Config:
    """
    Configuration for data processing.

    Parameters
    ----------
    path: str
        Path to the data directory
    batch_size: int | None = 32
        Number of records per batch
    """

    path: str
    batch_size: int | None = 32

    def __post_init__(self):
        self.path = Path(self.path)


def load(path: str, columns: list[str] | None = None) -> pd.DataFrame:
    """
    Load a CSV file and optionally select columns.

    Parameters
    ----------
    path: str
        Path to the CSV file
    columns: list[str] | None = None
        Columns to select. If `None`, all columns are returned.
    """
    return pd.read_csv(path, usecols=columns)


def transform(frame: pd.DataFrame, threshold: float | None = 0.5) -> pd.DataFrame:
    """
    Filter rows by score and reset the index.

    Parameters
    ----------
    frame: pd.DataFrame
        The input frame
    threshold: float | None = 0.5
        Minimum score to keep
    """
    return (
        frame
        [lambda frame: frame['score'] > threshold]
        .reset_index(drop=True)
    )


identity = lambda _: _
```

## Miscellaneous

Use f-strings exclusively for string formatting. Prefer single quotes.

```python
# bad
path = str(index).zfill(9) + '.arrows'
message = 'Processing {} records'.format(count)

# good
path = f'{index:09d}.arrows'
message = f'Processing {count:,d} records'
```

Comments are sparse and explain "why", not "what". Use lowercase. No trailing period.

```python
# bad — describes what the code does
# Initialize the DataFrame with empty values.
frame = pd.DataFrame()

# good — explains why this line exists
# load one shard in the background while streaming another
if shard_size:
    frame = pd.DataFrame()
```

Write one arithmetic operation per line, with the operator at the end of the line. Indent continuation lines when an operand itself requires parentheses.

```python
# bad
result = [0] * min(a, b) + [1] * max(0, b - a)

# good
result = (
    [0] *
    min(a, b) +
    [1] *
    max(
        0,
        b - a
    )
)
```

Never use trailing commas in the last element of a list, dict, tuple, function call, or function signature.

```python
# bad
config = {
    'name': 'bob',
    'age': 12,
}

# good
config = {
    'name': 'bob',
    'age': 12
}
```

Avoid unnecessary vertical whitespace. Only use blank lines to separate functions, imports, and code blocks that have a comment header.

Use assertions for preconditions and parameter validation.

```python
# bad
if column is None and columns is None:
    raise ValueError('Either column or columns must be provided')

# good
assert column is not None or columns is not None
```

```python
# bad
if not ((ordering == 'random') or (column is not None)):
    raise ValueError('...')

# good
assert (ordering == 'random') ^ (column is not None)
```

Embed tests at the bottom of the module behind a pytest guard. Add a blank line after the guard and before the imports. Define test data inside each test function rather than in a shared block at the top of the test section, so that the data is local to the test that uses it.

```python
# bad — test data defined far from where it is used
if 'pytest' in sys.modules:

    import pytest

    frame = pd.DataFrame({'a': [1, 2, 3], 'b': [4, 5, 6]})

    def test_add_columns():
        result = add_columns(frame)
        assert 'c' in result.columns

# good — test data is local to the test
if 'pytest' in sys.modules:

    import pytest

    def test_add_columns():
        result = add_columns(
            pd.DataFrame({'a': [1, 2, 3], 'b': [4, 5, 6]})
        )
        assert 'c' in result.columns
```

Always use multiprocessing or multithreading for parallelizable CPU-bound or IO-bound computations.

```python
# bad
results = []
for item in items:
    results.append(process(item))

# good
with multiprocessing.Pool() as pool:
    results = list(
        pool.imap(process, items)
    )
```

Use `'a b c'.split()` instead of `['a', 'b', 'c']` for short string lists.

```python
# bad
columns = ['name', 'age', 'email']

# good
columns = 'name age email'.split()
```

Use type annotations on function signatures only, not on inline variable assignments.

```python
# bad
def load(path: str) -> pd.DataFrame:
    buffer: list[torch.Tensor] = []

# good
def load(path: str) -> pd.DataFrame:
    buffer = []
```
