# Pandas

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
