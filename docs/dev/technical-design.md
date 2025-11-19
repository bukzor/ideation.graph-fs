# Technical Design

**Read this when:**
- Understanding how the system works
- Implementing tools or integrations
- Exploring implementation options

## System Overview

Graph-native filesystem adds typed, weighted edges to POSIX filesystems using the `.edge/` directory convention. Files and directories become graph nodes, with edges expressed through subdirectories containing symlinks.

## Architecture

```
/directory/                    ← Node (filesystem object)
├── child1                     ← Traditional child
├── child2
└── .edge/                     ← Edge namespace
    ├── contains/              ← Edge type
    │   ├── child1 -> ../child1   ← Explicit edge (optional)
    │   └── child2 -> ../child2
    ├── depends_on/            ← Custom edge type
    │   ├── lib1 -> /usr/lib/lib1.so
    │   └── lib2 -> /usr/lib/lib2.so
    └── references/            ← Another custom type
        └── doc -> /docs/api.md
```

## Path Resolution

### Traditional POSIX Path

Path: `/a/b/c`

Resolution:
1. Start at root inode
2. Lookup `a` in root's directory entries
3. Lookup `b` in `a`'s directory entries
4. Lookup `c` in `b`'s directory entries

### Graph-Native Path

Path: `/a/.edge/depends_on/c`

Resolution:
1. Start at root inode
2. Lookup `a` in root's directory entries
3. Lookup `.edge` in `a` (real directory or virtual)
4. Lookup `depends_on` in `.edge/`
5. Lookup `c` in `depends_on/` → follow symlink target

**Key insight:** Same resolution mechanism. `.edge/` is just a directory, edge types are subdirectories, edges are symlinks.

### Implicit Edge Type

Path: `/a/b/c` implicitly means `/a/.edge/contains/b/.edge/contains/c`

Missing `.edge/` directory: assume implicit `contains` edge type that mirrors parent's children.

## Storage Models

### Phase 1: Pure Convention (Current)

**Implementation:** Standard directories and symlinks, no special tooling.

**Structure:**
```bash
mkdir -p .edge/depends_on
ln -s /target .edge/depends_on/foo
```

**Advantages:**
- Works immediately with zero setup
- Compatible with all POSIX tools
- Version controllable (git tracks symlinks)
- No runtime dependencies

**Limitations:**
- Manual edge management
- No consistency enforcement
- No virtual `.edge/` (must create explicitly)
- Edge properties require workarounds

### Phase 2: FUSE Filesystem (Future)

**Implementation:** Userspace filesystem providing virtual `.edge/` and query interfaces.

**Data structures:**
```c
struct edge {
    char *type;              // Edge type name
    ino_t target;            // Target inode number
    float weight;            // Optional weight (0.0-1.0)
    char *properties;        // JSON metadata
};

struct node {
    ino_t inode;
    char *name;
    struct edge *edges;      // Edge list
};
```

**FUSE operations:**
- `readdir("/.edge")` → list edge types for current node
- `readdir("/.edge/type")` → list edges of given type
- `readlink("/.edge/type/name")` → resolve target path
- `getxattr("/.edge/type/name", "user.weight")` → get edge property

**Advantages:**
- Virtual `.edge/` always present
- Enforce conventions
- Efficient storage (backend database vs many symlinks)
- Rich property support

**Limitations:**
- Requires FUSE daemon running
- More complexity
- Debugging harder than pure files

### Phase 3: Kernel VFS Module (Long-term)

**Implementation:** Native kernel support for graph semantics.

**Interface:** New syscalls for edge operations.

**Advantages:**
- Maximum performance
- No userspace daemon
- Native tool integration

**Limitations:**
- Platform-specific
- Kernel development complexity
- Deployment friction

## Edge Semantics

### Directionality

Edges are directed: A → B means "A relates to B via edge type."

Inverse relationships by convention:
- `depends_on` ↔ `depended_on_by`
- `references` ↔ `referenced_by`
- `contains` ↔ `contained_by`

**Implementation:** Tooling can maintain bidirectional consistency.

Example:
```bash
# Create edge A -> B
ln -s /B /A/.edge/depends_on/B

# Tooling auto-creates inverse B -> A
ln -s /A /B/.edge/depended_on_by/A
```

### Edge Weights

Numeric values (typically 0.0-1.0) expressing:
- Importance/priority
- Confidence level
- Recency (normalized timestamp)
- Strength of relationship

**Storage options:**

**Extended attributes:**
```bash
ln -s /target .edge/type/name
setfattr -n user.weight -v 0.9 .edge/type/name
setfattr -n user.confidence -v 0.7 .edge/type/name
```

**Sidecar files:**
```bash
.edge/type/
├── name -> /target
└── name.properties    # weight=0.9, confidence=0.7
```

**Filename encoding:**
```bash
.edge/type/
└── name:weight=0.9 -> /target
```

### Transitivity

Some edge types are transitive, others not:

**Transitive:**
- `depends_on`: A depends on B, B depends on C → A transitively depends on C
- `contains`: A contains B, B contains C → A transitively contains C
- `references`: Often transitive (doc A refs B, B refs C → A indirectly refs C)

**Non-transitive:**
- `conflicts_with`: A conflicts with B, B conflicts with C ≠> A conflicts with C
- `similar_to`: Similarity not guaranteed transitive

**Implementation:** Tooling computes transitive closure for transitive edge types.

## Standard Edge Types

Proposed conventions (not enforced):

### Structural
- `contains`: Parent-child (default, backward compatible)
- `symlink_to`: Traditional symlink made explicit
- `hardlink_to`: Hardlink siblings

### Dependencies
- `depends_on`: Build/runtime dependency
- `depended_on_by`: Inverse of depends_on
- `requires`: Mandatory dependency
- `optional`: Optional dependency
- `conflicts_with`: Incompatible with

### Code Relationships
- `imports`: Source-level import
- `references`: Documentation/comment reference
- `implements`: Interface implementation
- `tests`: Test coverage
- `derives_from`: Generated from source
- `generates`: Produces output

### Semantic
- `similar_to`: Content similarity
- `related_to`: General relationship
- `replaces`: Deprecation/version
- `forks_from`: Source divergence

### Temporal
- `modifies`: Edit history
- `created_by`: Authorship
- `accessed_by`: Usage tracking

## Query Patterns

### Direct Queries

List edge types:
```bash
ls .edge/
```

List edges of type:
```bash
ls .edge/depends_on/
```

Follow edge:
```bash
cd .edge/depends_on/foo
```

### Find Operations

All dependencies in tree:
```bash
find . -path '*/.edge/depends_on/*' -type l
```

All nodes that depend on X:
```bash
find / -path '*/.edge/depends_on/X' -type l
```

Broken edges (dangling symlinks):
```bash
find . -path '*/.edge/*' -type l -xtype l
```

### Transitive Queries

Recursive dependencies (shell):
```bash
get-deps() {
    local node=$1
    echo "$node"
    for dep in "$node/.edge/depends_on/"*; do
        [[ -L "$dep" ]] && get-deps "$(readlink -f "$dep")"
    done
}
```

**Note:** Future query language will handle this more elegantly.

## Data Flow

### Edge Creation

```
User creates edge:
  mkdir -p /A/.edge/depends_on
  ln -s /B /A/.edge/depends_on/B

Result:
  /A/.edge/depends_on/B -> /B

Optionally add properties:
  setfattr -n user.weight -v 0.9 /A/.edge/depends_on/B
```

### Edge Traversal

```
User accesses edge:
  ls /A/.edge/depends_on/B

Filesystem:
  1. Resolve /A (standard lookup)
  2. Resolve /A/.edge (real directory)
  3. Resolve /A/.edge/depends_on (real directory)
  4. Resolve /A/.edge/depends_on/B (symlink)
  5. Follow symlink -> /B
  6. Return /B's metadata
```

### Edge Query

```
User queries edges:
  find /project -path '*/.edge/depends_on/*'

Filesystem:
  1. Walk directory tree
  2. For each .edge directory found
  3. For each subdirectory matching 'depends_on'
  4. For each symlink inside
  5. Report path
```

## Portability

### Cross-Platform

**Core convention works on:**
- Linux (all distros)
- macOS
- BSDs (FreeBSD, OpenBSD, NetBSD)
- Windows (with limitations)

**Platform-specific considerations:**

**Linux:**
- Extended attributes via `setfattr`/`getfattr`
- Symlinks fully supported

**macOS:**
- Extended attributes via `xattr`
- Symlinks fully supported

**Windows:**
- Symlinks require admin privileges (use junctions instead)
- NTFS alternate data streams for properties
- Path length limits more restrictive

### Version Control

**Git handles the convention naturally:**

```bash
# Edges are tracked as symlinks
git add .edge/depends_on/
git commit -m "Add dependency edges"

# Symlinks stored as objects
git ls-tree HEAD .edge/
# 120000 blob abc123  .edge/depends_on/foo -> /usr/lib/foo
```

**Best practices:**
- Use relative symlinks for portability: `../../lib/foo` not `/absolute/lib/foo`
- Add `.edge/` to avoid ignoring accidentally: `!.edge/` in `.gitignore`
- Document absolute symlinks in README if necessary

### Tool Compatibility

**Works with standard tools:**
- `ls`, `find`, `grep`: Navigate edges naturally
- `tar`, `zip`: Archive edges (as symlinks)
- `rsync`: Sync edges (`-l` for symlinks)
- `tree`: Visualize structure

**Breaks with:**
- Tools that ignore symlinks by default
- Tools with hardcoded `.` prefix filtering (overcome with `-a`)

## Security Considerations

### Symlink Attacks

Standard POSIX symlink vulnerabilities apply:
- TOCTTOU (time-of-check-time-of-use)
- Symlink following in world-writable directories
- Privilege escalation via symlink manipulation

**Mitigations:**
- Standard symlink best practices apply
- Use `O_NOFOLLOW` when appropriate
- Check ownership/permissions before following
- Avoid world-writable `.edge/` directories

### Information Disclosure

`.edge/` directories expose relationship metadata:
- Dependencies reveal architectural choices
- References show information flow
- Access patterns leak usage information

**Mitigations:**
- Appropriate permissions on `.edge/` directories
- Encryption at rest for sensitive graphs
- Access control lists for relationship data

### Cycle Detection

Circular graphs can cause infinite loops:

```
/A/.edge/foo/B -> /B
/B/.edge/foo/A -> /A
```

**Mitigations:**
- Graph traversal tools must maintain visited set
- Depth limits on recursive operations
- Validation tools detect cycles

## Performance

### Space Overhead

Per edge:
- Directory entry: ~256 bytes
- Symlink: ~60 bytes + path length

Example: 100 edges across 10 types
- 10 directories (type names): ~2.5 KB
- 100 symlinks: ~6 KB
- Total: ~8.5 KB overhead

Negligible compared to file content.

### Time Overhead

Edge resolution:
- `.edge/` lookup: 1 directory traversal
- Type lookup: 1 subdirectory traversal
- Symlink resolution: 1 `readlink()` + 1 path resolution

Overhead: ~3 extra syscalls per edge traversal.

Cached by OS:
- dentry cache: directory entries
- inode cache: inodes
- page cache: symlink targets

Typical performance impact: <5% for edge-heavy operations.

### Scalability

**Small scale** (< 1000 edges per node): No issues

**Medium scale** (1000-10000 edges): Directory listing becomes slower

**Large scale** (> 10000 edges): Consider FUSE implementation with indexing

## Future Extensions

### Query Language

Proposed syntax for advanced queries:

```bash
# Pattern matching
/.query/*/depends_on/*/lib*.so

# Property filtering
/.query/edges[weight>0.8]/

# Transitive closure
/.query/transitive/depends_on/
```

### Automatic Inference

Tools that populate edges automatically:
- Static analysis → `imports`, `references`
- Build system → `depends_on`, `generates`
- Runtime tracing → `calls`, `loads`
- Content analysis → `similar_to`

### Graph Algorithms

Filesystem-level graph operations:
- Shortest path between nodes
- Centrality metrics (most depended-on)
- Community detection (related clusters)
- Cycle detection and breaking

### Visualization

```bash
ge visualize /project --type depends_on --format dot
# Generates GraphViz DOT file
# Optionally opens in viewer
```

---

**Last Updated:** 2025-11-19
