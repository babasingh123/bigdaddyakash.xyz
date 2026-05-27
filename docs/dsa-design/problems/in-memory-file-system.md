# Design In-Memory File System

!!! note "The disguise"
    Interviewer says: **"Design a file system."**
    They mean: **"Implement a Trie-like tree of HashMaps with path-parsing on top."**

## Problem

Support a subset of POSIX-like operations:

- `ls(path)` — if `path` is a directory, return its children (sorted). If it's a file, return the filename in a single-element list.
- `mkdir(path)` — create all directories along this path (recursive mkdir).
- `addContentToFile(path, content)` — create the file if missing, then append `content`.
- `readContentFromFile(path)` — return the file's content.

## What it really tests

- **Tree as nested HashMaps.** Each directory is just a `dict` mapping child name → subnode. That's structurally a trie keyed on path segments.
- **Path parsing.** Splitting `"/a/b/c.txt"` into `["a", "b", "c.txt"]` and walking the tree.
- **Distinguishing files from directories.** Each node knows whether it's a file (and holds content) or a directory (and holds children).
- **DFS-style traversal.** `mkdir` walks down, creating nodes as needed.

## Approach

The shape of the data is *literally a tree where edges are path components*. That makes it a trie keyed on strings (instead of characters).

**The node.** One class can represent both files and directories:

- `is_file: bool`
- `content: str` — only meaningful if `is_file`.
- `children: dict[str, Node]` — only meaningful if not a file.

Why one class for both? Because at parse time you don't yet know — `"/a/b"` could be a file or a dir. You just walk the tree and pick the right field at the end.

**Path parsing.** Treat `"/"` as the root. Split by `"/"` and drop empty strings to handle the leading slash. `"/a/b/c"` → `["a", "b", "c"]`.

**`mkdir(path)`** — walk component by component, inserting empty `Node()` as you go. Idempotent: if the directory exists, do nothing.

**`addContentToFile(path)`** — same walk, then on the final component:

- If the node doesn't exist, create one, mark `is_file=True`.
- Append to its `content`.

**`readContentFromFile(path)`** — walk, return content.

**`ls(path)`** — walk; if final node is a file, return `[basename(path)]`; if directory, return `sorted(children.keys())`.

**Why HashMap children and not list?** O(1) lookup per path component. Sorting only matters at `ls` time, where we sort the children's keys explicitly — a small price for fast walks.

## Code sketch

```python
class Node:
    __slots__ = ("is_file", "content", "children")
    def __init__(self):
        self.is_file = False
        self.content = ""
        self.children: dict[str, "Node"] = {}

class FileSystem:
    def __init__(self):
        self.root = Node()

    @staticmethod
    def _split(path: str) -> list[str]:
        return [p for p in path.split("/") if p]

    def _walk(self, parts: list[str], create: bool = False) -> Node | None:
        node = self.root
        for p in parts:
            if p not in node.children:
                if not create:
                    return None
                node.children[p] = Node()
            node = node.children[p]
        return node

    def ls(self, path: str) -> list[str]:
        parts = self._split(path)
        node = self._walk(parts)
        if node is None:
            return []
        if node.is_file:
            return [parts[-1]]
        return sorted(node.children.keys())

    def mkdir(self, path: str) -> None:
        self._walk(self._split(path), create=True)

    def addContentToFile(self, path: str, content: str) -> None:
        node = self._walk(self._split(path), create=True)
        node.is_file = True
        node.content += content

    def readContentFromFile(self, path: str) -> str:
        node = self._walk(self._split(path))
        return node.content if node else ""
```

## Complexity

Let *L* = number of path components, *k* = max children per directory, *N* = total content size.

- `mkdir(path)`: **O(L)** — one HashMap insert per component.
- `addContentToFile`: **O(L + |content|)**.
- `readContentFromFile`: **O(L + |content|)**.
- `ls(path)`: **O(L + k log k)** — walk + sort of immediate children.
- Space: **O(total path-segments + total content)**.

## Edge cases

- **Trailing/leading slashes.** Filter out empty components after `split("/")`.
- **Root listing.** `ls("/")` should return root's children, sorted.
- **`ls` on a file.** Returns just the filename (not the parent's full listing).
- **`addContentToFile` on an existing file.** Append, don't overwrite.
- **`mkdir` on an existing directory.** No-op.

## Linked concepts

- [Cache & Memory category overview](../by-category/cache-memory.md)
- [Word Dictionary](word-dictionary.md) — same tree-of-HashMaps idea, keyed on chars
- [Search Autocomplete](search-autocomplete.md) — adds DFS-over-trie
