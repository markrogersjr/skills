# Function Signatures

## Docstrings

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

---

## `main` Functions

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

---

## Type Annotations

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
