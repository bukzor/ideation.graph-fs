# TODO / Roadmap

## Phase 1: Validation (Current)

- [x] Document core concept
- [x] Document design decisions
- [x] Provide concrete examples
- [ ] Use convention on real project
- [ ] Gather feedback from users
- [ ] Refine edge type conventions
- [ ] Document common pitfalls

## Phase 2: Tooling

### CLI Helper (`ge` - graph-edge)

```bash
ge add <type> <target>      # Create edge
ge rm <type> <target>       # Remove edge
ge list [type]              # List edges (all or by type)
ge query <type>             # Find all edges of type in tree
ge inverse <type>           # List inverse edge convention
ge validate                 # Check edge consistency
ge export [format]          # Export as DOT/JSON/etc
```

Features:
- [ ] Basic add/list/remove operations
- [ ] Bidirectional edge management
- [ ] Edge property support (xattrs or sidecar files)
- [ ] Validation (detect cycles, broken links)
- [ ] Graph visualization (DOT output)
- [ ] Import/export (JSON, YAML, DOT)

### Shell Integration

- [ ] Tab completion for `.edge/` directories
- [ ] `@type` alias expansion
- [ ] Prompt integration (show edge context)
- [ ] Directory navigation helpers

### Version Control Integration

- [ ] Git hooks for edge validation
- [ ] Edge conflict resolution
- [ ] Merge strategies for edges
- [ ] `.gitattributes` for edge properties

## Phase 3: Applications

### Neo4j Exporter

```bash
neo4j-to-fs --db mydb --output /tmp/graph
fs-to-neo4j --input /tmp/graph --db mydb
```

Features:
- [ ] Export Neo4j graph to filesystem
- [ ] Import filesystem graph to Neo4j
- [ ] Handle node labels as directories
- [ ] Handle relationship properties
- [ ] Incremental sync

### Build System Integration

```bash
# Makefile integration
make EDGE_TRACK=1

# Ninja integration
ninja -t graph > .edge/
```

Features:
- [ ] Generate edges from build graphs
- [ ] Track source → object relationships
- [ ] Track header dependencies
- [ ] Incremental build optimization
- [ ] Parallel build insights

### Package Manager

```bash
pkg-graph install flask
pkg-graph deps flask          # Show dependencies
pkg-graph rdeps flask         # Show reverse dependencies
pkg-graph why flask werkzeug  # Explain dependency path
```

Features:
- [ ] Dependency tracking via edges
- [ ] Conflict detection
- [ ] Transitive dependency resolution
- [ ] Virtual packages via edge types
- [ ] Version constraints as edge properties

### Documentation System

```bash
doc-graph link <source> <target>    # Create reference
doc-graph backlinks <doc>           # Find what links here
doc-graph orphans                   # Find unlinked docs
doc-graph graph                     # Visualize doc structure
```

Features:
- [ ] Bidirectional reference tracking
- [ ] Dead link detection
- [ ] Documentation coverage analysis
- [ ] Auto-generate relationship diagrams
- [ ] Integration with static site generators

## Phase 4: FUSE Prototype

### Core Functionality

- [ ] Virtual `.edge/` on all inodes
- [ ] Default edge type configuration
- [ ] Transparent edge resolution
- [ ] Property storage backend

### Query Interface

```bash
# Virtual query directories
ls /.query/edges/depends_on/weight\>0.8/

# Pattern matching
ls /.query/pattern/*/depends_on/*/contains/lib*.so
```

Features:
- [ ] Query language design
- [ ] Query optimization
- [ ] Result caching
- [ ] Property filtering

### Advanced Features

- [ ] Automatic edge inference (via inotify)
- [ ] Edge indexes for fast queries
- [ ] Transitive closure computation
- [ ] Graph algorithms (shortest path, centrality, etc.)
- [ ] Snapshot/versioning support

## Phase 5: Kernel Module (Long-term)

### VFS Integration

- [ ] New syscalls for edge operations
- [ ] Native edge storage format
- [ ] Performance optimization
- [ ] Security model

### Standardization

- [ ] POSIX extension proposal
- [ ] Cross-platform compatibility
- [ ] Reference implementation
- [ ] Test suite

## Research Questions

### Semantics

- [ ] How to handle edge type namespacing?
- [ ] Should edge types be extensible (plugin system)?
- [ ] How to version edge type schemas?
- [ ] What's the minimal standard library of edge types?

### Performance

- [ ] How do large edge graphs perform?
- [ ] What's the optimal indexing strategy?
- [ ] How to handle millions of edges per node?
- [ ] What's the cache hit rate for edge traversals?

### Usability

- [ ] What edge types do users actually need?
- [ ] How intuitive is the `.edge/` convention?
- [ ] What are common confusion points?
- [ ] How to teach this to new users?

### Integration

- [ ] How to integrate with existing tools?
- [ ] What breaks with the convention?
- [ ] How to migrate existing projects?
- [ ] What's the adoption path?

## Experiments

### Benchmarks

- [ ] Edge creation performance
- [ ] Edge traversal performance
- [ ] Query performance at scale
- [ ] Storage overhead measurements

### User Studies

- [ ] Usability testing
- [ ] Developer interviews
- [ ] Real-world case studies
- [ ] Comparison with alternatives

### Proof of Concepts

- [ ] FUSE prototype for query evaluation
- [ ] Static analyzer that generates edges
- [ ] IDE integration (show edges in UI)
- [ ] Graph visualization tool

## Documentation

- [ ] Tutorial: Getting started
- [ ] Tutorial: Migrating existing projects
- [ ] Best practices guide
- [ ] Edge type reference
- [ ] Tool comparison (vs symlinks, tags, etc)
- [ ] FAQ
- [ ] Video demos

## Community

- [ ] Create discussion forum
- [ ] Write blog posts
- [ ] Present at conferences
- [ ] Gather use cases
- [ ] Build community tools
- [ ] Establish conventions working group

---

## Immediate Next Steps

1. **Use it ourselves**: Apply convention to this repo
2. **Build `ge` CLI**: Minimal viable tool for edge management
3. **Write tutorial**: Step-by-step guide for first-time users
4. **Test on real project**: Use on non-trivial codebase
5. **Get feedback**: Share with others, iterate on design

## Contributing

Pick a TODO, open an issue, start a discussion. This is exploratory work - all ideas welcome.

## Success Metrics

- People using the convention without tooling (pure convention adoption)
- Projects organizing files with graph semantics
- Tools built on top of the convention
- Integration into existing tools (IDEs, build systems, etc)
- FUSE prototype demonstrates value beyond convention
- Standardization discussion begins

---

**Next Update**: After Phase 1 validation completes.
