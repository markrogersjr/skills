# Attribute and Method Chaining

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
