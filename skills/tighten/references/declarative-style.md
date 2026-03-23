# Declarative Style

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
