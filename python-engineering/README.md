# python-engineering

## The problem

AI-generated Python code has a characteristic failure mode: it works, and it's wrong. It passes the immediate test, satisfies the prompt, and produces the output you asked for. What it doesn't do is hold up — because it wasn't designed, it was generated.

Classes that instantiate their own dependencies. Config objects passed five layers deep. Registries with `if/elif` chains. Broad `except Exception` blocks with no justification. Code that cannot be extended without being rewritten.

This is AI slop. It looks like code. It behaves like code. It becomes a liability the moment requirements change.

## What this skill does

Installs senior Python engineering standards into Claude. Before writing a line, Claude names design problems and surfaces tradeoffs. If a request would produce an inferior design, it says so.

Covers: dependency inversion, Strategy and Registry patterns, ABCs, composition roots, type hints, resource management, logging, exception handling, config architecture, and testing. Python 3.10+.

The standard: adding a new implementation should require no changes to any other file. The composition root is the only place that knows what is concrete.

There are code review skills, linting skills, and style skills everywhere. Most operate after the code is written. This skill operates before — at the design level, before the wrong structure gets built.

## When to use it

- Any Python project where the code has to last
- When you know what you want to build but not the correct Python approach
- When AI keeps producing code that works now but can't be extended
- When you want to enforce engineering discipline without knowing every pattern yourself

## Files

- `python-engineering.skill` — install into Claude
- `python_engineering_prompt.md` — activation prompt and usage guide
