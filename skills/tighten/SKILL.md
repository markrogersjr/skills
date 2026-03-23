---
name: tighten
description: Use a declarative style, locality of reference, and more to simplify code and make it more readable
---

# Tighten: Declarative Style, Locality, and Simplification

The following guidelines minimize two metrics from *Code Complete* by Steve McConnell that determine how much back-and-forth a reader must do to understand code:

- **span**, the number of lines between successive references to a variable, and
- **live time**, the total number of lines between a variable's first and last reference.

## Function Signatures

Add docstrings to functions and classes only, following a tweaked NumPy convention. Define `main()` as a regular function whose parameters are the CLI interface, dispatched via `fire` or `omegaconf` — never `argparse`. Use modern Python 3.10+ type annotations (`X | None`, builtins over `typing`).

> `references/function-signatures.md`

## Creating Variables

Spell out variable names in full. Never create a variable unless it is referenced more than once — inline single-use values directly. Define every variable as close as possible to where it is first used (locality of reference), using `try/except NameError` for lazy initialization inside loops. Variable shadowing is intentional for conciseness.

> `references/creating-variables.md`

## Declarative Style

Compose expressions inline rather than assigning them to intermediate variables. Cram as many nested expressions as possible into a single statement, eliminating named intermediaries that inflate span and live time.

> `references/declarative-style.md`

## Indentation

Indentation visually communicates the logical nesting depth of an expression. Each level corresponds to one level deeper in the expression tree. Consistent indentation is what makes deeply composed declarative expressions readable.

> `references/indentation.md`

## Creating Commands

Don't create a statement unless it is necessary. Pass options through constructors and initializers to collapse multiple statements into one.

> `references/creating-commands.md`

## DRY

Never duplicate code. If two blocks differ only in the data they operate on, factor the shared logic into a loop or generator expression.

> `references/dry.md`

## Pandas

Always pass `as_index=False` to `groupby` and shadow-rename the group variable to `frame`. Use `.to_dict('records')` instead of `.iterrows()`. Use `[lambda frame: ...]` for inline filtering when method-chaining.

> `references/pandas.md`

## Attribute and Method Chaining

When a chain exceeds 120 characters, break it so each line contains at most one attribute access or function call. The dot goes on the new line. Closing parentheses sit on their own line at the same indentation as the opening call.

> `references/chaining.md`

## Parentheses

Never nest more than two opening delimiters on the same line. When a third level would appear, break the inner expression to the next line.

> `references/parentheses.md`

## File Layout

Order module bodies: imports, type definitions, class/function definitions, then commands and assignments. Organize imports into four alphabetically-sorted blocks: builtins, third-party, private, and local.

> `references/file-layout.md`

## Miscellaneous

F-strings exclusively (single quotes). Comments are sparse, explain "why" not "what", lowercase, no trailing period. One arithmetic operation per line with operator at end. No trailing commas. Minimal vertical whitespace. Assertions for preconditions. Tests at module bottom behind a pytest guard with local test data. Multiprocessing for parallelizable work. `'a b c'.split()` for string lists. Type annotations on signatures only.

> `references/miscellaneous.md`
