# Graph-Native Filesystem

A graph-native filesystem design that extends POSIX conventions with typed, weighted edges while maintaining full backward compatibility.

## Core Concept

Traditional filesystems enforce tree structures. You choose one parent for each file, sacrificing other organizational axes. Graph-native filesystems allow multiple typed relationships between nodes (files/directories).

**Key insight**: POSIX paths already support graphs through hardlinks and symlinks. We just need conventions for expressing typed edges.

## The `.edge/` Convention

Every directory contains a virtual `.edge/` subdirectory exposing typed relationships:

```bash
/projects/alpha/
├── src/
├── docs/
└── .edge/
    ├── contains/      # Default: mirrors ../
    ├── depends_on/    # User-defined edges
    └── references/
```

### Path Syntax

```bash
# Implicit :contains edge (backward compatible)
/projects/alpha/src/main.rs

# Explicit edge traversal
/projects/alpha/.edge/depends_on/libssl

# Short form (@ expands to .edge/)
/projects/alpha/@depends_on/libssl
```

## Implementation: Works Today

This is **pure convention** using existing POSIX primitives:

```bash
cd ~/projects/myapp

# Create typed edges
mkdir -p .edge/depends_on
ln -s /usr/lib/libssl.so .edge/depends_on/
ln -s /usr/lib/libz.so .edge/depends_on/

mkdir -p .edge/references
ln -s ~/docs/architecture.md .edge/references/

# Query relationships
ls .edge/depends_on/
find . -path '*/.edge/depends_on/*' -type l
```

No kernel changes. No FUSE. Just directories and symlinks.

## Design Decisions

### Edge Types Are User-Defined

Common conventions:
- `contains`: Traditional parent-child (default)
- `depends_on`: Build/runtime dependencies
- `references`: Documentation links
- `derives_from`: Generated files
- `related_to`: Semantic similarity

### Symlinks vs Hardlinks

**Use symlinks** for edges:
- Work for directories
- Cross filesystem boundaries
- Semantic is "points to" not "is the same inode"
- Clear visual indicator (`->`)

### Missing `.edge/`

If `.edge/` doesn't exist, assume implicit `contains -> ..` mapping. Legacy directories work as-is.

### Edge Properties

Store metadata on edges using:

1. **Extended attributes** (most flexible):
```bash
ln -s /lib/openssl .edge/depends_on/openssl
setfattr -n user.weight -v 0.9 .edge/depends_on/openssl
setfattr -n user.since -v 2020 .edge/depends_on/openssl
```

2. **Sidecar files** (portable):
```bash
.edge/depends_on/
├── openssl -> /lib/openssl
└── openssl.properties    # weight=0.9, since=2020
```

3. **Filename encoding** (simple):
```bash
.edge/depends_on/
└── openssl:weight=0.9:since=2020 -> /lib/openssl
```

## Operations

### pwd with Multiple Parents

```bash
$ pwd
/projects/alpha/src

# Show node identity
$ stat -c %i .
8675309

# Find all paths to this node
$ find / -inum 8675309
/projects/alpha/src
/.edge/rust/sources/alpha-src
/.edge/recent/accessed/2025-11-17
```

Existing semantics handle multiple paths correctly.

### cd with Multiple Parents

`cd ..` follows the inverse of your traversal path:
- Came via `alpha/@contains/src` → go back to `alpha`
- Path context determines parent

### Querying Graphs

```bash
# Standard tools work
ls .edge/                    # List edge types
ls .edge/depends_on/         # List dependencies
find . -path '*/.edge/KNOWS/*'  # Find relationships

# With globs
ls @depends_on/*/lib*.so     # Transitive dependencies
```

## Use Case: Neo4j to Filesystem

Export graph databases as explorable filesystems:

```bash
# Neo4j: (alice:Person)-[:KNOWS]->(bob:Person)
/nodes/
  alice/
    .properties           # name=Alice, age=30
    .edge/
      KNOWS/
        bob -> ../../bob
      WORKS_AT/
        acme -> ../../acme
  bob/
    .properties
```

Browse with standard tools:
```bash
cd /nodes/alice
ls .edge/KNOWS/              # Who does Alice know?
cat .edge/KNOWS/bob/.properties
```

## Tooling Ideas

### Helper CLI

```bash
#!/bin/bash
# ge (graph-edge): manage filesystem edges

case $1 in
  add)
    mkdir -p ".edge/$2"
    ln -s "$3" ".edge/$2/"
    ;;
  list)
    ls -l ".edge/$2" 2>/dev/null || echo "No $2 edges"
    ;;
  query)
    find . -path "*/.edge/$2/*" -type l
    ;;
esac
```

Usage:
```bash
ge add depends_on /usr/lib/libssl.so
ge list depends_on
ge query depends_on
```

### Shell Integration

```bash
# Expand @type to .edge/type
alias @='cd .edge'
# Or shell function for transparent rewriting
```

### FUSE Layer (Optional)

For richer semantics:
- Virtual `.edge/` on all directories
- Default edge type configuration
- Edge weight filtering
- Query language support

But the convention works **without** FUSE.

## Next Steps

1. **Establish conventions**: Document common edge types
2. **Build tooling**: CLI helpers for edge management
3. **Experiment**: Use on real projects, gather feedback
4. **FUSE prototype**: If conventions prove useful, add filesystem-level support
5. **Graph queries**: Develop path query language for traversals

## Related Work

- **Git's object model**: Directed acyclic graph with typed edges (tree, commit, parent)
- **Semantic filesystems**: Content-based organization
- **WinFS**: Microsoft's attempt at relational filesystem (abandoned)
- **Plan 9 namespaces**: Union directories, different abstraction
- **Graph databases**: Neo4j, but for filesystems

## Contributing

This is an ideation repo. Contributions welcome:
- Convention refinements
- Tool implementations
- Real-world usage examples
- FUSE prototype experiments

---

**Created**: 2025-11-19
**Status**: Concept exploration
**License**: Public domain / CC0
