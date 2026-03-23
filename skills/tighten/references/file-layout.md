# File Layout

The body of a module is ordered: imports, type definitions, class and function definitions, then commands and assignments. Type annotations that are used by subsequent classes and functions may appear before them; all other variables go at the bottom. Organize imports into four blocks separated by blank lines: builtins, third-party, private, and local. Each block is sorted alphabetically. If you see `import foo.bar`, that means `foo` is also imported; do not add a redundant `import foo`. Separate the import section from the rest of the module with two blank lines. Separate each top-level class and function definition with two blank lines.

```python
import sys
from dataclasses import dataclass
from pathlib import Path

import pandas as pd
from tqdm import tqdm

import my_private_package
import my_private_package.utils

import this_current_package.models
import this_current_package.io


Record = dict[str, str | int | float]


@dataclass
class Config:
    """
    Configuration for data processing.

    Parameters
    ----------
    path: str
        Path to the data directory
    batch_size: int | None = 32
        Number of records per batch
    """

    path: str
    batch_size: int | None = 32

    def __post_init__(self):
        self.path = Path(self.path)


def load(path: str, columns: list[str] | None = None) -> pd.DataFrame:
    """
    Load a CSV file and optionally select columns.

    Parameters
    ----------
    path: str
        Path to the CSV file
    columns: list[str] | None = None
        Columns to select. If `None`, all columns are returned.
    """
    return pd.read_csv(path, usecols=columns)


def transform(frame: pd.DataFrame, threshold: float | None = 0.5) -> pd.DataFrame:
    """
    Filter rows by score and reset the index.

    Parameters
    ----------
    frame: pd.DataFrame
        The input frame
    threshold: float | None = 0.5
        Minimum score to keep
    """
    return (
        frame
        [lambda frame: frame['score'] > threshold]
        .reset_index(drop=True)
    )


identity = lambda _: _
```
