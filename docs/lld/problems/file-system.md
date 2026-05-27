# Design a File System

> **Time:** 45–60 minutes

## Problem

Design an in-memory file system (Unix-style). The system must:

- Support **files** and **directories** in a hierarchical structure.
- Support **CRUD** operations: create, read, update, delete, move, rename.
- Allow **search** by name, extension, modification time.
- Compute **directory size** recursively.
- Enforce **file permissions** (read/write/execute, owner/group/other).

## Real-world intuition

The Unix file system, JSON document trees, DOM trees, organizational charts — they all share the same shape: a recursive tree of nodes where some nodes are leaves (files) and some nodes are containers (directories). This is the **Composite pattern**'s canonical use case. Get this hierarchy right and the rest of the design falls out naturally.

## Key classes

- `Entry` (abstract, Composite component) — base type with name, parent, permissions, timestamps; defines `size()`, `walk(Visitor)`.
- `File` (leaf) — holds content (bytes); `size()` = content length.
- `Directory` (composite) — holds child `Entry`s by name; `size()` = sum of children's sizes.
- `FileSystem` — singleton; owns the root directory; entry point for path-based operations.
- `Path` — utility for parsing `/a/b/c` into a sequence of names.
- `Permission` — `read`/`write`/`execute` flags for `owner`/`group`/`other`.
- `User` — identity; checks permissions on operations.
- `Visitor` (interface) — traversal pattern (for search, total size, find-largest, etc.).

## Design patterns used

- **Composite (Entry → File / Directory)** — the canonical pattern. Files and directories are treated uniformly through the `Entry` base. Recursive operations (size, walk, search) collapse to single methods.
- **Visitor (search, sizing, indexing)** — traversal logic for new operations (find files matching a regex, build a search index, count by extension) is added as new `Visitor` implementations without modifying `Entry` subclasses.
- **Singleton (FileSystem)** — there's one root; the FS object is the entry point. Legitimate global.

## Class diagram (sketch)

```text
           ┌──────────────┐
           │ FileSystem   │ «Singleton»
           ├──────────────┤
           │ -root: Dir   │
           │ +mkdir(path) │
           │ +touch(path) │
           │ +rm(path)    │
           └──────────────┘
                  │
                  ▼
           ┌──────────────┐
           │    Entry     │ «abstract / composite»
           ├──────────────┤
           │ -name        │
           │ -parent      │
           │ -perms       │
           │ +size()      │
           │ +accept(v)   │
           └──────△───────┘
                  │
        ┌─────────┴─────────┐
        │                   │
   ┌────────┐         ┌──────────┐
   │  File  │ «leaf»  │Directory │ «composite»
   ├────────┤         ├──────────┤
   │-content│         │-children │◇──▶ Entry (recursive)
   └────────┘         └──────────┘

   Visitor «interface»
      △
      ┊
   SearchVisitor   SizeVisitor   IndexVisitor
```

## Code sketch

```java
abstract class Entry {
    protected final String name;
    protected Directory parent;
    protected Permission perms;
    protected Instant createdAt, modifiedAt;

    protected Entry(String name) { this.name = name; }

    public abstract long size();
    public abstract <T> T accept(Visitor<T> v);          // Visitor hook
}

class File extends Entry {
    private byte[] content = new byte[0];

    public File(String name) { super(name); }
    public long size()                 { return content.length; }
    public <T> T accept(Visitor<T> v)  { return v.visitFile(this); }

    public byte[] read()                  { return content.clone(); }
    public void write(byte[] data)        { this.content = data.clone(); modifiedAt = Instant.now(); }
    public void append(byte[] data)       { /* ... */ }
}

class Directory extends Entry {
    private final Map<String, Entry> children = new LinkedHashMap<>();

    public Directory(String name) { super(name); }

    public long size() {
        return children.values().stream().mapToLong(Entry::size).sum();
    }

    public <T> T accept(Visitor<T> v) { return v.visitDirectory(this); }

    public void add(Entry e)              { children.put(e.name, e); e.parent = this; }
    public void remove(String name)       { Entry e = children.remove(name); if (e != null) e.parent = null; }
    public Entry get(String name)         { return children.get(name); }
    public Collection<Entry> children()   { return children.values(); }
}

interface Visitor<T> {
    T visitFile(File f);
    T visitDirectory(Directory d);
}

class SearchVisitor implements Visitor<List<File>> {
    private final Predicate<File> match;
    public SearchVisitor(Predicate<File> m) { this.match = m; }

    public List<File> visitFile(File f) {
        return match.test(f) ? List.of(f) : List.of();
    }
    public List<File> visitDirectory(Directory d) {
        List<File> out = new ArrayList<>();
        for (Entry e : d.children()) out.addAll(e.accept(this));
        return out;
    }
}

class FileSystem {
    private static final FileSystem INSTANCE = new FileSystem();
    private final Directory root = new Directory("/");

    public static FileSystem get() { return INSTANCE; }

    public void mkdir(String path) { /* parse, walk, create missing parents */ }
    public File create(String path) {
        Path p = Path.parse(path);
        Directory parent = (Directory) resolve(p.parent());
        File f = new File(p.last());
        parent.add(f);
        return f;
    }
    public Entry resolve(Path p) { /* walk from root */ return null; }
    public long sizeOf(String path) { return resolve(Path.parse(path)).size(); }
    public List<File> find(String startPath, Predicate<File> match) {
        Entry start = resolve(Path.parse(startPath));
        return start.accept(new SearchVisitor(match));
    }
}
```

## Key design decisions

1. **Composite is the entire design** — `File` and `Directory` share `Entry`. Every recursive operation (size, walk, search, permission check) is one polymorphic call. Without Composite, you'd write `if (entry instanceof File) ... else if (entry instanceof Directory) ...` in every method.
2. **Visitor for read-only traversals** — adding "count files by extension" or "find files >10MB" should not require modifying `Entry`, `File`, or `Directory`. The Visitor pattern makes traversal logic extensible while leaving the entry hierarchy stable (OCP).
3. **Parent pointers** — moving a file requires knowing where it currently lives, and computing absolute paths needs upward traversal. Parent pointer is cheap and worth it.
4. **`size()` recursion on Directory** — naive but correct. For very large trees, cache the size and invalidate on writes; for an interview answer, recursion is the right starting point.
5. **Permission checks at the operation layer** — `FileSystem.read()` checks the user against the entry's `Permission`. Putting the check inside `File.read()` couples permission policy to the file; keeping it in the FS layer keeps `File` a dumb data holder.
6. **Singleton FS** — there's one root, one mount; the FS as a whole is global. Genuine fit.

## Extensibility

**Add symbolic links:**

1. `class Symlink extends Entry` whose `size()` is the link length, and which carries a `target` path.
2. `Visitor` gets `visitSymlink(Symlink)`; existing visitors handle it (e.g., search follows links once, sizing skips them to avoid cycles).

**Add hard links (multiple names for the same content):**

1. `File`'s content becomes a reference to a shared `Inode` object.
2. Multiple `File` entries can share an inode; deletion only frees the inode when the last link is removed.

**Add quotas per user:**

1. Walk the tree once via a `QuotaVisitor` that aggregates size by owner.
2. Enforce at write time by re-running on the affected branch.

**Add a real index (search by content):**

1. `IndexVisitor implements Visitor<Void>` builds a full-text index on directory contents.
2. Run on demand, or hook into `File.write()` for incremental updates.

**Add transparent compression:**

1. Wrap `File` reads/writes in a `CompressedFile extends File` decorator.
2. Existing visitors still work.

## Linked concepts

- [OOP](../fundamentals/oop.md) — composite hierarchy is OOP done well.
- [SOLID](../fundamentals/solid.md) — OCP via Visitor; SRP for FileSystem orchestration vs Entry data model.
- [Design Patterns](../fundamentals/design-patterns.md) — Composite (canonical), Visitor, Singleton.
