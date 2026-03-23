# Creating Commands

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
