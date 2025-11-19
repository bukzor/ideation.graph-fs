# Design Deep Dive

## Problem Statement

Filesystems force hierarchical organization. A file named `api-docs.md` must live in ONE location:
- `/projects/myapp/docs/api-docs.md`?
- `/documentation/apis/myapp.md`?
- `/reference/rest-apis/myapp.md`?

You pick one. The other organizational axes are lost. Symlinks help but are ad-hoc and don't express relationship semantics.

## Solution: Typed Edges

Make relationships first-class through the `.edge/` convention. Multiple typed relationships can coexist:

```
api-docs.md
├── contained by: /projects/myapp/docs/
├── referenced by: /code/server.py
├── related to: /specs/openapi.yaml
└── derived from: /code/routes.py (via docgen)
```

## Technical Design

### Path Resolution

Traditional path: `/a/b/c`
1. Start at root inode
2. Lookup `a` in root's directory entries
3. Lookup `b` in `a`'s directory entries
4. Lookup `c` in `b`'s directory entries

Graph-native path: `/a/@depends_on/c`
1. Start at root inode
2. Lookup `a` in root's directory entries
3. Lookup `.edge` in `a`'s directory (virtual or real)
4. Lookup `depends_on` in `.edge/`
5. Lookup `c` in `depends_on/` (symlink target)

**Same mechanism. Just explicit edge type in path.**

### Storage Model

#### Option 1: Pure Convention (Current)

```
/projects/alpha/
├── src/
├── docs/
└── .edge/              # Real directory
    ├── contains/       # Optional: explicit listing
    │   ├── src -> ../src
    │   └── docs -> ../docs
    └── depends_on/     # User-created
        ├── libssl -> /usr/lib/libssl.so
        └── libz -> /usr/lib/libz.so
```

Pros:
- Works today, zero implementation
- Standard tools compatible
- Clear, explorable structure

Cons:
- Manual edge management
- No enforcement of conventions
- Edge properties require extended attributes or sidecars

#### Option 2: FUSE Filesystem

Virtual `.edge/` on every inode:

```c
struct node {
    ino_t inode;
    char *name;
    struct edge *edges;  // Linked list of typed edges
};

struct edge {
    char *type;           // "depends_on", "contains", etc
    ino_t target;         // Target inode
    float weight;         // Optional: edge weight
    char *properties;     // JSON blob or key=value pairs
    struct edge *next;
};
```

FUSE operations:
- `readdir("/foo/.edge")` → list edge types
- `readdir("/foo/.edge/depends_on")` → list edges of that type
- `readlink("/foo/.edge/depends_on/bar")` → resolve target

Pros:
- Virtual `.edge/` always present
- Can enforce conventions
- Efficient storage (no redundant symlinks)
- Rich edge properties

Cons:
- Requires FUSE implementation
- More complex
- Dependency on FUSE daemon

#### Option 3: Kernel VFS Module

Full kernel-level implementation.

Pros:
- Maximum performance
- Native syscall support
- No userspace overhead

Cons:
- Kernel development complexity
- Deployment friction
- Platform-specific

### Recommended Path

1. **Phase 1**: Pure convention (Option 1) - validate concept
2. **Phase 2**: FUSE prototype - if conventions prove useful
3. **Phase 3**: Kernel module - if FUSE proves limiting

Start simple. Add complexity only when needed.

## Edge Semantics

### Directionality

Edges are directed:
- `/a/@depends_on/b` means "a depends on b"
- Inverse: `/b/@depended_on_by/a` means "b is depended on by a"

Conventions can establish inverse pairs:
- `depends_on` ↔ `depended_on_by`
- `references` ↔ `referenced_by`
- `derives_from` ↔ `generates`

Tooling can maintain bidirectional consistency:
```bash
ge add depends_on /lib/ssl
# Automatically creates inverse edge:
# /lib/ssl/.edge/depended_on_by/$(pwd)
```

### Edge Weights

Numeric weights (0.0-1.0) for:
- Importance/priority
- Confidence level
- Recency (timestamp normalized)
- Strength of relationship

Query by weight:
```bash
# Find high-confidence dependencies
find .edge/depends_on/ -exec getfattr -n user.weight {} \; | awk '$2 > 0.8'
```

### Transitive Edges

Edge types can imply transitivity:
- `depends_on`: transitive (if A depends on B, B on C → A depends on C)
- `contains`: transitive
- `references`: not transitive

Tooling can compute transitive closure:
```bash
ge transitive depends_on /app
# Returns all transitive dependencies
```

## Interaction with Existing POSIX

### Hardlinks

Hardlinks create multiple paths to same inode (same as graph-native). But:
- Can't hardlink directories
- Can't cross filesystems
- No semantic information

Graph-native uses symlinks in `.edge/` for semantic relationships while preserving hardlink behavior for the default `contains` edge.

### Symlinks

Existing symlinks are path-based redirects. Graph edges are relationship-based.

A symlink `/tmp/quicklink -> /projects/alpha/src` is a shortcut.

A graph edge `/projects/alpha/.edge/depends_on/libfoo` is a semantic relationship.

They can coexist. Symlinks for convenience, edges for semantics.

### Mount Points

`.edge/` convention works across mount points:

```bash
/mnt/external/data/
└── .edge/
    └── depends_on/
        └── config -> /etc/myapp/config  # Cross-filesystem
```

Symlinks handle cross-filesystem references naturally.

## Query Language

### Basic Traversal

```bash
# Direct edges
/a/@depends_on/

# Transitive edges (hypothetical syntax)
/a/@depends_on*/

# Any edge type
/a/@*/

# Inverse edges
/a/@^contains/    # "What contains a?"
```

### Filtering

```bash
# By weight
/a/@depends_on[weight>0.8]/

# By property
/a/@KNOWS[since>2020]/

# Combined
/a/@depends_on[weight>0.8,type=runtime]/
```

### Graph Patterns

Cypher-inspired:
```bash
# Files referenced by dependencies
/app/@depends_on/*/@references/

# Common dependencies
/app/@depends_on/ ∩ /other/@depends_on/
```

This is future work. Start with simple `ls .edge/type/` queries.

## Standard Edge Types

Proposed conventions:

### File Relationships
- `contains`: Parent-child (default)
- `symlink_to`: Traditional symlink (explicit)
- `hardlink_to`: Hardlink siblings

### Code Relationships
- `depends_on`: Build/runtime dependency
- `imports`: Source-level import
- `references`: Documentation/comment reference
- `tests`: Test coverage relationship
- `implements`: Interface implementation
- `derives_from`: Code generation source

### Semantic Relationships
- `similar_to`: Content similarity
- `related_to`: General relationship
- `replaces`: Version/deprecation
- `forks_from`: Source divergence

### Temporal Relationships
- `modifies`: Edit history
- `created_by`: Authorship
- `accessed_by`: Usage tracking

### User-Defined
Users can create arbitrary edge types. Standard library provides common ones.

## Portability

### Cross-Platform

Core convention (directories + symlinks) works on:
- Linux
- macOS
- BSDs
- Windows (symlinks require privileges, but junctions work)

Extended attributes for edge properties are platform-specific:
- Linux: `setfattr`/`getfattr`
- macOS: `xattr`
- Windows: NTFS alternate data streams

Fallback to sidecar `.properties` files for portability.

### Version Control

`.edge/` directories are just directories. Git tracks them:

```bash
git add .edge/depends_on/
git commit -m "Add dependency edges"
```

Symlinks in git are stored as symlink objects. Works naturally.

**Caveat**: Absolute symlinks break when repo moves. Use relative paths:
```bash
# Bad
ln -s /home/user/lib/foo.so .edge/depends_on/

# Good
ln -s ../../../lib/foo.so .edge/depends_on/
```

## Security Considerations

### Symlink Attacks

Standard symlink vulnerabilities apply:
- Time-of-check-time-of-use (TOCTTOU)
- Symlink following in world-writable directories

Mitigations:
- Use `O_NOFOLLOW` when appropriate
- Check ownership/permissions
- Same best practices as current symlink usage

### Information Disclosure

`.edge/` exposes relationships. If `project/.edge/depends_on/` lists proprietary libraries, that's metadata leakage.

Mitigations:
- Permissions on `.edge/` directories
- Encryption at rest
- Access control lists

### Denial of Service

Circular edge graphs could cause infinite loops:
```
/a/.edge/foo/b -> /b
/b/.edge/foo/a -> /a
```

Tooling should:
- Detect cycles
- Limit traversal depth
- Use visited-set during graph walks

## Performance

### Space Overhead

Each edge type directory + symlinks:
- Directory entry: ~256 bytes
- Symlink: ~60 bytes + path length

For 100 edges: ~30KB overhead per directory. Negligible.

### Time Overhead

- `readdir(.edge/)`: One extra directory listing
- `readlink()`: One extra syscall per edge traversal

Overhead is minimal. Similar to existing symlink resolution.

### Caching

OS already caches:
- Directory entries (dentry cache)
- Inodes (inode cache)
- Symlink targets (page cache)

Graph traversals benefit from existing kernel caching.

## Future Extensions

### Native Queries

Filesystem-level query language:
```sql
SELECT path FROM edges
WHERE type = 'depends_on'
  AND weight > 0.8
  AND source = '/app'
```

### Automatic Edge Inference

Tools that populate edges automatically:
- Static analysis → `imports`, `references`
- Build system → `depends_on`, `generates`
- Runtime tracing → `calls`, `loads`
- Content analysis → `similar_to`

### Graph Indexes

Dedicated indexes for fast queries:
- Inverted edge index: "what points to X?"
- Property index: "edges with weight > 0.9"
- Transitive closure cache

### Visualization

```bash
ge visualize /app/@depends_on
# Generates DOT graph, opens in viewer
```

### Synchronization

Sync edge metadata across systems:
```bash
ge push    # Upload edges to remote
ge pull    # Fetch edges from remote
ge merge   # Resolve edge conflicts
```

---

This design balances pragmatism (works today) with vision (extensible future). Start with conventions, evolve as needed.
