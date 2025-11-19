# Graph-Native Filesystem

A POSIX-compatible convention for expressing typed, weighted edges between files using `.edge/` directories.

## Status

⚠️ **Concept Exploration:** This is an ideation project documenting a filesystem convention. Not production software.

See [STATUS.md](STATUS.md) for current focus and next steps.

## What Is This?

Traditional filesystems force tree structures. You pick one location for each file, losing other organizational axes.

This convention adds graph semantics to any POSIX filesystem:

```bash
# Create typed relationships
mkdir -p /project/.edge/depends_on
ln -s /usr/lib/libssl.so /project/.edge/depends_on/

# Query relationships
ls /project/.edge/depends_on/
```

Works today. No kernel changes. No FUSE. Just directories and symlinks.

## Core Idea

Every directory can contain `.edge/` with subdirectories for relationship types:

```
/project/
├── src/
├── docs/
└── .edge/
    ├── contains/      # Default: traditional children
    ├── depends_on/    # Custom: dependencies
    └── references/    # Custom: doc links
```

Paths traverse edges explicitly:

```bash
/project/.edge/depends_on/libssl    # Follow depends_on edge
```

## Quick Examples

**Software dependencies:**
```bash
mkdir .edge/depends_on
ln -s /lib/libssl.so .edge/depends_on/
```

**Documentation references:**
```bash
mkdir .edge/references
ln -s ~/docs/api.md .edge/references/
```

**Neo4j graph export:**
```bash
# (alice)-[:KNOWS]->(bob) becomes:
/nodes/alice/.edge/KNOWS/bob -> ../../bob
```

See [docs/examples/](docs/examples/) for detailed use cases.

## Why?

- **Multiple organization axes**: Same file accessible via different relationships
- **Semantic relationships**: Edge types express meaning (depends_on, references, etc.)
- **Works with existing tools**: Standard POSIX operations (mkdir, ln, ls, find)
- **Graph databases as filesystems**: Browse Neo4j exports with cd and ls

## Documentation

- **[CLAUDE.md](CLAUDE.md)** - Quick reference for development (LLM-focused)
- **[STATUS.md](STATUS.md)** - Current milestone and next actions
- **[docs/dev/design-rationale.md](docs/dev/design-rationale.md)** - Why this design
- **[docs/dev/technical-design.md](docs/dev/technical-design.md)** - How it works
- **[docs/dev/development-plan.md](docs/dev/development-plan.md)** - Roadmap and milestones
- **[docs/examples/](docs/examples/)** - Practical use cases

## Contributing

This is exploratory work. Ideas, feedback, and alternative approaches welcome.

Open an issue or discussion to participate.

## License

CC0 (Public Domain) - See [LICENSE](LICENSE)
