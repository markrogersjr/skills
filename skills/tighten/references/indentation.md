# Indentation

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
