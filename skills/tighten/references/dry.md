# DRY

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
