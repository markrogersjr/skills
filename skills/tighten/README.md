# Tighten

A coding style skill for coding agents that produces tighter, more readable Python.
We optimize for two metrics from *Code Complete* (McConnell) — **span**, the number of lines between successive references to a variable, and **live time**, the total lines between a variable's first and last reference — through declarative composition, aggressive inlining of single-use variables, and locality of reference.

## Examples

We gave the same prompt to Claude under three configurations: **tighten**, Anthropic's **code-simplifier**, and **no skill** (default).

> Write a Python script that trains a small neural network on synthetic classification data. The script has three parts:
>
> 1. **Data pipeline** — generate synthetic data, split into train/validation/test sets, and batch it for training
> 2. **Training loop** — mini-batch SGD over multiple epochs, tracking train/validation loss, saving the best model to disk
> 3. **Evaluation harness** — load the saved checkpoint, run it on the test set, and print a classification report
>
> Accept hyperparameters and the output path from the command line.

All three produce working scripts of ~170 lines. The structural choices diverge in ways that directly affect span and live time.

The default output splits logic across five functions (`parse_args`, `build_dataloaders`, `train`, `evaluate`, `main`), each called exactly once. This inflates live time: `model` is created in `main()` at line 161, passed to `train()` at 162, then to `evaluate()` at 163 — the reader must jump across three function boundaries to follow its lifecycle. `best_val_loss` is initialized 28 lines before its first use inside the loop. Every parameter is defined twice: once in the argparse block, once in the function signature that receives it.

The code-simplifier output has the same function structure but extracts a `run_epoch` helper to deduplicate the train/validation loops. This is its clearest intervention. Span and live time are unchanged for most variables — `best_val_loss` is still initialized before the loop, parameters are still defined twice in argparse, and `model` still crosses the same function boundaries.

Tighten collapses everything into a single `main()` since no function is called more than once. This eliminates cross-function span entirely — `model` is created at line 37 and every reference through evaluation stays in one scope. `best_validation_loss` is lazily initialized via `try/except NameError` *inside* the loop body, 1 line before its use, instead of 28 lines above. `fire` replaces argparse, deleting 12 lines of duplicated parameter definitions and reducing the span between a parameter's declaration and its first use to zero (the function signature *is* the CLI). The phase loop (`'train validation'.split()`) and the evaluation expression tree further eliminate intermediate variables that the other two versions create and reference only once.

### Tighten

```python
import sys
from pathlib import Path

import fire
import torch
import torch.nn as nn
from sklearn.datasets import make_classification
from sklearn.metrics import classification_report
from sklearn.model_selection import train_test_split


class Network(nn.Module):
    """
    Feedforward network with two hidden layers for classification.

    Parameters
    ----------
    input_features: int
        Number of input features
    hidden_size: int
        Number of units in each hidden layer
    output_classes: int
        Number of output classes
    """

    def __init__(self, input_features: int, hidden_size: int, output_classes: int):
        super().__init__()
        self.layers = nn.Sequential(
            nn.Linear(input_features, hidden_size),
            nn.ReLU(),
            nn.Linear(hidden_size, hidden_size),
            nn.ReLU(),
            nn.Linear(hidden_size, output_classes),
        )

    def forward(self, x: torch.Tensor) -> torch.Tensor:
        return self.layers(x)


def main(
    output_path: str,
    samples: int | None = 2000,
    features: int | None = 20,
    classes: int | None = 4,
    hidden_size: int | None = 64,
    learning_rate: float | None = 1e-2,
    batch_size: int | None = 64,
    epochs: int | None = 50,
    seed: int | None = 42,
):
    """
    Train a small neural network on synthetic classification data. Generates a synthetic dataset, splits it into
    train/validation/test sets, trains with mini-batch SGD while tracking losses, saves the best checkpoint to disk,
    then evaluates on the test set and prints a classification report.

    Parameters
    ----------
    output_path: str
        Directory to save the best model checkpoint
    samples: int | None = 2000
        Number of synthetic samples to generate
    features: int | None = 20
        Number of input features
    classes: int | None = 4
        Number of target classes
    hidden_size: int | None = 64
        Number of units per hidden layer
    learning_rate: float | None = 1e-2
        SGD learning rate
    batch_size: int | None = 64
        Mini-batch size
    epochs: int | None = 50
        Number of training epochs
    seed: int | None = 42
        Random seed for reproducibility
    """
    torch.manual_seed(seed)
    Path(output_path).mkdir(parents=True, exist_ok=True)
    # data pipeline: generate, split 60/20/20, batch
    x, y = make_classification(
        n_samples=samples,
        n_features=features,
        n_classes=classes,
        n_informative=features // 2,
        n_redundant=0,
        random_state=seed,
    )
    x_trainval, x_test, y_trainval, y_test = train_test_split(
        x, y,
        test_size=0.2,
        random_state=seed,
        stratify=y,
    )
    x_train, x_val, y_train, y_val = train_test_split(
        x_trainval, y_trainval,
        test_size=0.25,
        random_state=seed,
        stratify=y_trainval,
    )
    loaders = {
        split: torch.utils.data.DataLoader(
            torch.utils.data.TensorDataset(
                torch.as_tensor(x_split, dtype=torch.float32),
                torch.as_tensor(y_split, dtype=torch.long),
            ),
            batch_size=batch_size,
            shuffle=(split == 'train'),
        )
        for split, x_split, y_split in [
            ('train', x_train, y_train),
            ('validation', x_val, y_val),
            ('test', x_test, y_test),
        ]
    }
    # training loop
    model = Network(features, hidden_size, classes)
    optimizer = torch.optim.SGD(
        model.parameters(),
        lr=learning_rate,
    )
    checkpoint_path = Path(output_path) / 'best_model.pt'
    for epoch in range(epochs):
        losses = {}
        for phase in 'train validation'.split():
            if phase == 'train':
                model.train()
            else:
                model.eval()
            running_loss = 0.0
            for batch_features, batch_labels in loaders[phase]:
                with torch.set_grad_enabled(phase == 'train'):
                    loss = nn.functional.cross_entropy(
                        model(batch_features),
                        batch_labels,
                    )
                if phase == 'train':
                    optimizer.zero_grad()
                    loss.backward()
                    optimizer.step()
                running_loss += loss.item()
            losses[phase] = running_loss / len(loaders[phase])
        print(
            f'epoch {epoch + 1:3d}/{epochs}'
            f'  train {losses["train"]:.4f}'
            f'  val {losses["validation"]:.4f}'
        )
        try:
            best_validation_loss
        except NameError:
            best_validation_loss = float('inf')
        if losses['validation'] < best_validation_loss:
            best_validation_loss = losses['validation']
            torch.save(model.state_dict(), checkpoint_path)
            print(f'  -> saved checkpoint (val loss {best_validation_loss:.4f})')
    # evaluation: load best checkpoint, report on test set
    model.load_state_dict(
        torch.load(checkpoint_path, weights_only=True)
    )
    model.eval()
    with torch.no_grad():
        print(
            f'\ntest results ({len(loaders["test"].dataset)} samples):\n'
            + classification_report(
                loaders['test'].dataset.tensors[1].numpy(),
                model(
                    loaders['test'].dataset.tensors[0]
                )
                .argmax(dim=1)
                .numpy(),
            )
        )


fire.Fire(main)
```

### Default

```python
#!/usr/bin/env python3
"""Train a small neural network on synthetic classification data."""

import argparse
import json
from pathlib import Path

import numpy as np
import torch
import torch.nn as nn
from sklearn.datasets import make_classification
from sklearn.metrics import classification_report
from sklearn.model_selection import train_test_split
from torch.utils.data import DataLoader, TensorDataset


# ── Model ────────────────────────────────────────────────────────────────────

class Classifier(nn.Module):
    def __init__(self, n_features: int, n_hidden: int, n_classes: int):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(n_features, n_hidden),
            nn.ReLU(),
            nn.Linear(n_hidden, n_hidden),
            nn.ReLU(),
            nn.Linear(n_hidden, n_classes),
        )

    def forward(self, x):
        return self.net(x)


# ── Data pipeline ───────────────────────────────────────────────────────────

def build_dataloaders(n_samples, n_features, n_classes, batch_size, seed):
    X, y = make_classification(
        n_samples=n_samples,
        n_features=n_features,
        n_informative=max(2, n_features // 2),
        n_classes=n_classes,
        n_clusters_per_class=1,
        random_state=seed,
    )
    X_train, X_rest, y_train, y_rest = train_test_split(
        X, y, test_size=0.3, stratify=y, random_state=seed
    )
    X_val, X_test, y_val, y_test = train_test_split(
        X_rest, y_rest, test_size=0.5, stratify=y_rest, random_state=seed
    )

    def to_loader(X_np, y_np, shuffle):
        ds = TensorDataset(
            torch.tensor(X_np, dtype=torch.float32),
            torch.tensor(y_np, dtype=torch.long),
        )
        return DataLoader(ds, batch_size=batch_size, shuffle=shuffle)

    return (
        to_loader(X_train, y_train, shuffle=True),
        to_loader(X_val, y_val, shuffle=False),
        to_loader(X_test, y_test, shuffle=False),
    )


# ── Training loop ───────────────────────────────────────────────────────────

def train(model, train_loader, val_loader, epochs, lr, output_dir):
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
    model.to(device)
    criterion = nn.CrossEntropyLoss()
    optimizer = torch.optim.SGD(model.parameters(), lr=lr, momentum=0.9)

    best_val_loss = float("inf")
    history = []
    ckpt_path = output_dir / "best_model.pt"

    for epoch in range(1, epochs + 1):
        # — train —
        model.train()
        train_loss = 0.0
        for X_batch, y_batch in train_loader:
            X_batch, y_batch = X_batch.to(device), y_batch.to(device)
            optimizer.zero_grad()
            loss = criterion(model(X_batch), y_batch)
            loss.backward()
            optimizer.step()
            train_loss += loss.item() * X_batch.size(0)
        train_loss /= len(train_loader.dataset)

        # — validate —
        model.eval()
        val_loss = 0.0
        with torch.no_grad():
            for X_batch, y_batch in val_loader:
                X_batch, y_batch = X_batch.to(device), y_batch.to(device)
                val_loss += criterion(model(X_batch), y_batch).item() * X_batch.size(0)
        val_loss /= len(val_loader.dataset)

        history.append({"epoch": epoch, "train_loss": train_loss, "val_loss": val_loss})
        marker = ""
        if val_loss < best_val_loss:
            best_val_loss = val_loss
            torch.save(model.state_dict(), ckpt_path)
            marker = " *"
        print(f"epoch {epoch:>3d}  train_loss={train_loss:.4f}  val_loss={val_loss:.4f}{marker}")

    with open(output_dir / "history.json", "w") as f:
        json.dump(history, f, indent=2)
    print(f"\nBest val loss: {best_val_loss:.4f}  (saved to {ckpt_path})")
    return ckpt_path


# ── Evaluation harness ──────────────────────────────────────────────────────

def evaluate(model, ckpt_path, test_loader):
    device = torch.device("cuda" if torch.cuda.is_available() else "cpu")
    model.load_state_dict(torch.load(ckpt_path, map_location=device, weights_only=True))
    model.to(device)
    model.eval()

    all_preds, all_labels = [], []
    with torch.no_grad():
        for X_batch, y_batch in test_loader:
            preds = model(X_batch.to(device)).argmax(dim=1).cpu()
            all_preds.append(preds)
            all_labels.append(y_batch)

    y_true = torch.cat(all_labels).numpy()
    y_pred = torch.cat(all_preds).numpy()
    print("\n── Test set classification report ──")
    print(classification_report(y_true, y_pred))


# ── CLI ──────────────────────────────────────────────────────────────────────

def parse_args():
    p = argparse.ArgumentParser(description=__doc__)
    p.add_argument("--output-dir", type=Path, default=Path("output"))
    p.add_argument("--n-samples", type=int, default=2000)
    p.add_argument("--n-features", type=int, default=20)
    p.add_argument("--n-classes", type=int, default=4)
    p.add_argument("--hidden-size", type=int, default=64)
    p.add_argument("--batch-size", type=int, default=64)
    p.add_argument("--epochs", type=int, default=50)
    p.add_argument("--lr", type=float, default=0.01)
    p.add_argument("--seed", type=int, default=42)
    return p.parse_args()


def main():
    args = parse_args()
    torch.manual_seed(args.seed)
    np.random.seed(args.seed)
    args.output_dir.mkdir(parents=True, exist_ok=True)

    train_loader, val_loader, test_loader = build_dataloaders(
        args.n_samples, args.n_features, args.n_classes, args.batch_size, args.seed
    )

    model = Classifier(args.n_features, args.hidden_size, args.n_classes)
    ckpt_path = train(model, train_loader, val_loader, args.epochs, args.lr, args.output_dir)
    evaluate(model, ckpt_path, test_loader)


if __name__ == "__main__":
    main()
```

### Anthropic's Code Simplifier

```python
#!/usr/bin/env python3
"""Train a small neural network on synthetic classification data."""

import argparse
import json
import pathlib

import numpy as np
import torch
import torch.nn as nn
from sklearn.datasets import make_classification
from sklearn.metrics import classification_report
from sklearn.model_selection import train_test_split
from torch.utils.data import DataLoader, TensorDataset


def parse_args():
    p = argparse.ArgumentParser(description="Train a small NN on synthetic data")
    p.add_argument("--output-dir", type=pathlib.Path, default=pathlib.Path("output"))
    p.add_argument("--n-samples", type=int, default=5000)
    p.add_argument("--n-features", type=int, default=20)
    p.add_argument("--n-classes", type=int, default=4)
    p.add_argument("--hidden-size", type=int, default=64)
    p.add_argument("--lr", type=float, default=1e-2)
    p.add_argument("--epochs", type=int, default=50)
    p.add_argument("--batch-size", type=int, default=64)
    p.add_argument("--seed", type=int, default=42)
    return p.parse_args()


# --- Data pipeline ---


def make_loaders(n_samples, n_features, n_classes, batch_size, seed):
    X, y = make_classification(
        n_samples=n_samples,
        n_features=n_features,
        n_informative=n_features // 2,
        n_classes=n_classes,
        random_state=seed,
    )
    X_train, X_tmp, y_train, y_tmp = train_test_split(
        X, y, test_size=0.3, random_state=seed, stratify=y
    )
    X_val, X_test, y_val, y_test = train_test_split(
        X_tmp, y_tmp, test_size=0.5, random_state=seed, stratify=y_tmp
    )

    def to_loader(X, y, shuffle=False):
        ds = TensorDataset(
            torch.from_numpy(X).float(),
            torch.from_numpy(y),
        )
        return DataLoader(ds, batch_size=batch_size, shuffle=shuffle)

    return (
        to_loader(X_train, y_train, shuffle=True),
        to_loader(X_val, y_val),
        to_loader(X_test, y_test),
    )


# --- Model ---


class Net(nn.Module):
    def __init__(self, n_features, hidden_size, n_classes):
        super().__init__()
        self.net = nn.Sequential(
            nn.Linear(n_features, hidden_size),
            nn.ReLU(),
            nn.Linear(hidden_size, hidden_size),
            nn.ReLU(),
            nn.Linear(hidden_size, n_classes),
        )

    def forward(self, x):
        return self.net(x)


# --- Training ---


def run_epoch(model, loader, criterion, optimizer=None):
    is_train = optimizer is not None
    model.train(is_train)
    total_loss, n = 0.0, 0
    with torch.set_grad_enabled(is_train):
        for xb, yb in loader:
            if is_train:
                optimizer.zero_grad(set_to_none=True)
            logits = model(xb)
            loss = criterion(logits, yb)
            if is_train:
                loss.backward()
                optimizer.step()
            total_loss += loss.item() * len(yb)
            n += len(yb)
    return total_loss / n


def train(model, train_loader, val_loader, epochs, lr, checkpoint_path):
    criterion = nn.CrossEntropyLoss()
    optimizer = torch.optim.SGD(model.parameters(), lr=lr)
    best_val_loss = float("inf")
    history = []

    for epoch in range(1, epochs + 1):
        train_loss = run_epoch(model, train_loader, criterion, optimizer)
        val_loss = run_epoch(model, val_loader, criterion)
        history.append({"epoch": epoch, "train_loss": train_loss, "val_loss": val_loss})
        if val_loss < best_val_loss:
            best_val_loss = val_loss
            torch.save(model.state_dict(), checkpoint_path)
        if epoch % 10 == 0 or epoch == 1:
            print(f"Epoch {epoch:3d}  train_loss={train_loss:.4f}  val_loss={val_loss:.4f}")

    print(f"\nBest validation loss: {best_val_loss:.4f}")
    return history


# --- Evaluation ---


def evaluate(model, checkpoint_path, test_loader):
    model.load_state_dict(torch.load(checkpoint_path, weights_only=True))
    model.eval()
    all_preds, all_targets = [], []
    with torch.no_grad():
        for xb, yb in test_loader:
            all_preds.append(model(xb).argmax(dim=1))
            all_targets.append(yb)
    preds = torch.cat(all_preds).numpy()
    targets = torch.cat(all_targets).numpy()
    n_classes = model.net[-1].out_features
    print(
        "\n"
        + classification_report(
            targets, preds, target_names=[f"class_{i}" for i in range(n_classes)]
        )
    )


# --- Main ---


def main():
    args = parse_args()
    args.output_dir.mkdir(parents=True, exist_ok=True)
    checkpoint_path = args.output_dir / "best_model.pt"

    torch.manual_seed(args.seed)
    np.random.seed(args.seed)

    train_loader, val_loader, test_loader = make_loaders(
        args.n_samples, args.n_features, args.n_classes, args.batch_size, args.seed
    )

    model = Net(args.n_features, args.hidden_size, args.n_classes)
    history = train(model, train_loader, val_loader, args.epochs, args.lr, checkpoint_path)

    with open(args.output_dir / "history.json", "w") as f:
        json.dump(history, f, indent=2)

    evaluate(model, checkpoint_path, test_loader)


if __name__ == "__main__":
    main()
```
