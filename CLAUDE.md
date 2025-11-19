# Graph-FS - Development Guide for Claude

## Quick Reference

**Current state:** See [STATUS.md](STATUS.md) for milestone, blockers, and next actions.

**Core concept:** `.edge/` directories express typed relationships between filesystem nodes using standard POSIX (mkdir, symlinks). Works today without kernel changes.

**Key insight:** POSIX paths already support graphs (hardlinks prove this). We just add convention for typed edges.

## Common Tasks

**Understanding the design philosophy:**
Why `.edge/` convention vs other approaches, tradeoffs considered. → See [docs/dev/design-rationale.md](docs/dev/design-rationale.md)

**Understanding how it works:**
Path resolution, storage models, edge semantics, query patterns. → See [docs/dev/technical-design.md](docs/dev/technical-design.md)

**Seeing practical applications:**
Real-world examples across use cases. → See [docs/examples/](docs/examples/)

**Tracking progress:**
Milestones, deliverables, next phases. → See [docs/dev/development-plan.md](docs/dev/development-plan.md)

## Architecture Overview

Graph-native filesystem using `.edge/` convention to add typed, weighted edges to POSIX filesystems:

```
/directory/
├── file1
├── file2
└── .edge/              # Every directory can have this
    ├── contains/       # Default edge type (mirrors ../)
    ├── depends_on/     # Custom edge type
    │   └── target -> /path/to/target
    └── references/     # Another custom type
```

**Path traversal:** `/a/.edge/depends_on/b` explicitly follows "depends_on" edge from a to b.

**Default behavior:** `/a/b/c` implicitly uses "contains" edges (backward compatible).

**Implementation phases:**
1. **Pure convention** (current): Just directories + symlinks, no tooling required
2. **CLI tooling**: Helper commands for edge management, queries, visualization
3. **FUSE layer**: Virtual `.edge/`, query interface, automatic edge inference
4. **Kernel module**: Native VFS integration (long-term)

**Key design choices:**
- Symlinks (not hardlinks) for cross-filesystem support
- User-defined edge types (no fixed schema)
- Edge properties via xattrs or sidecar files
- Bidirectional edges by convention (not enforcement)

See [docs/dev/technical-design.md](docs/dev/technical-design.md) for complete architecture.

## Key Files

**Current state:**
- `STATUS.md` - Where we are, what's next
- `docs/dev/devlog/` - Session-by-session progress

**Core documentation:**
- `docs/dev/design-rationale.md` - Why this design (alternatives, tradeoffs)
- `docs/dev/technical-design.md` - How the system works
- `docs/dev/development-plan.md` - Roadmap with milestones

**Examples:**
- `docs/examples/` - Practical use cases (Neo4j, build systems, docs, etc.)

## Use Cases

**Software dependencies:** Track runtime/build dependencies via `depends_on` edges.

**Documentation graphs:** Cross-reference docs with `references` edges, find backlinks.

**Neo4j export:** Export graph databases as browsable filesystems.

**Build systems:** Track source→object relationships with `derives_from` edges.

**Package managers:** Dependency resolution using edge traversal.

**Content management:** Tag relationships, related posts via typed edges.

See [docs/examples/](docs/examples/) for detailed implementations.

## Conventions

**Edge naming:** lowercase_with_underscores (e.g., `depends_on`, `referenced_by`)

**Inverse edges:** Pairs like `depends_on` ↔ `depended_on_by` maintained by convention/tooling

**Default edge:** `contains` represents traditional parent-child relationships

**Missing `.edge/`:** Interpreted as implicit `contains` → parent mapping

**Edge weights:** 0.0-1.0 via extended attributes or sidecar `.properties` files

## Development Phases

**Phase 1 (Current): Validation**
- Document concept, gather feedback
- Use convention on real projects
- Identify pain points and common patterns

**Phase 2: CLI Tooling**
- `ge` (graph-edge) command for add/list/query/visualize
- Shell integration (tab completion, aliases)
- Git integration (hooks, merge strategies)

**Phase 3: Applications**
- Neo4j exporter/importer
- Build system integration
- Package manager prototype
- Documentation system

**Phase 4: FUSE Prototype**
- Virtual `.edge/` on all inodes
- Query interface (pattern matching, filtering)
- Automatic edge inference
- Performance optimization

**Phase 5: Standardization**
- Kernel module
- POSIX extension proposal
- Cross-platform compatibility

See [docs/dev/development-plan.md](docs/dev/development-plan.md) for detailed milestones.

## Testing

Currently concept validation phase. Future testing will include:
- Edge creation/traversal performance benchmarks
- Storage overhead measurements
- Query performance at scale
- Usability studies

## Latest Session

See most recent entry in [docs/dev/devlog/](docs/dev/devlog/) for last session's work.
