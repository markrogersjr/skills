# Creating Variables

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

---

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

---

Variable shadowing is intentional for conciseness. Do not refactor code to avoid shadowing unless it causes a bug. Shadowing builtins (`callable`, `object`, `format`) as parameter names is acceptable.

```python
# bad — renamed to avoid shadowing
for _, filtered_frame in frame.groupby('category', as_index=False):
    process(filtered_frame)

# good — shadow frame intentionally
for _, frame in frame.groupby('category', as_index=False):
    process(frame)
```
