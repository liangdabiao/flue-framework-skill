---
title: Sessions and operations
description: Continue conversations and run focused actions inside a harness.
sidebar:
  order: 2
tableOfContents: false
---

A session is a named conversation scope inside a harness. Its operations include `prompt()`, `skill()`, `task()`, and `shell()` where enabled by the harness.

## Operations

| Operation     | Use for                                                    |
| ------------- | ---------------------------------------------------------- |
| `prompt(...)` | Interactive model work that may take multiple turns.       |
| `task(...)`   | Focused detached delegation with separate message history. |
| `shell(...)`  | Approved commands in the configured workspace.             |

A workflow can initialize more than one named harness when its orchestration needs isolation between scopes.
