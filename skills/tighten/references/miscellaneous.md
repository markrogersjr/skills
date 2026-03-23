# Miscellaneous

Use f-strings exclusively for string formatting. Prefer single quotes.

```python
# bad
path = str(index).zfill(9) + '.arrows'
message = 'Processing {} records'.format(count)

# good
path = f'{index:09d}.arrows'
message = f'Processing {count:,d} records'
```

---

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

---

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

---

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

---

Avoid unnecessary vertical whitespace. Only use blank lines to separate functions, imports, and code blocks that have a comment header.

---

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

---

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

---

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

---

Use `'a b c'.split()` instead of `['a', 'b', 'c']` for short string lists.

```python
# bad
columns = ['name', 'age', 'email']

# good
columns = 'name age email'.split()
```

---

Use type annotations on function signatures only, not on inline variable assignments.

```python
# bad
def load(path: str) -> pd.DataFrame:
    buffer: list[torch.Tensor] = []

# good
def load(path: str) -> pd.DataFrame:
    buffer = []
```
