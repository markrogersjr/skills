# Parentheses

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
