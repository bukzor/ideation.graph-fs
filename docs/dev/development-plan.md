# Development Plan

## Overview

This project progresses through phases: concept validation → tooling → applications → implementation. Each phase informs the next.

## Milestones

### Milestone 1: Documentation and Validation

**Status:** In Progress

**Goal:** Establish core concept, document design, validate through real-world usage.

**Deliverables:**
- [x] Document core concept and design rationale
- [x] Create repository and project structure
- [x] Restructure documentation following LLM-collaborative standards
- [ ] Apply `.edge/` convention to this repository (self-hosting)
- [ ] Use convention on at least one real project
- [ ] Gather feedback from early adopters
- [ ] Document common usage patterns and pain points

**Dependencies:** None (starting milestone)

**Success Criteria:**
- Documentation is clear and comprehensive
- At least 2-3 real projects using the convention
- Common patterns identified and documented
- No fundamental blockers discovered

**Estimated Effort:** 2-4 weeks

### Milestone 2: CLI Tooling (`ge`)

**Status:** Not Started

**Goal:** Build command-line tool to simplify edge management and querying.

**Deliverables:**
- [ ] Design `ge` CLI interface
- [ ] Implement basic operations (add, remove, list)
- [ ] Implement query operations (find, transitive)
- [ ] Add validation (detect cycles, broken links)
- [ ] Support property management (xattrs, sidecar files)
- [ ] Add visualization (DOT output)
- [ ] Write tests for core functionality
- [ ] Document CLI usage

**Dependencies:** Milestone 1 (validate convention before building tools)

**Success Criteria:**
- Tool reduces friction for common operations
- Properties can be managed consistently
- Query operations are fast and accurate
- Users prefer `ge` over manual mkdir/ln

**Estimated Effort:** 2-3 weeks

### Milestone 3: Shell Integration

**Status:** Not Started

**Goal:** Make edge navigation feel native in shell environments.

**Deliverables:**
- [ ] Bash/Zsh completion for `.edge/` directories
- [ ] Shell aliases and functions (`@type` expansion)
- [ ] Prompt integration (show edge context)
- [ ] Navigation helpers (cd with edge awareness)
- [ ] Install scripts for common shells

**Dependencies:** Milestone 2 (CLI tools provide backend)

**Success Criteria:**
- Tab completion works for edge types
- Edge navigation feels natural
- No shell performance degradation
- Works across bash, zsh, fish

**Estimated Effort:** 1-2 weeks

### Milestone 4: Application - Neo4j Integration

**Status:** Not Started

**Goal:** Demonstrate value by enabling graph database exports to filesystem.

**Deliverables:**
- [ ] Design export format (nodes, edges, properties)
- [ ] Implement `neo4j-to-fs` exporter
- [ ] Implement `fs-to-neo4j` importer
- [ ] Handle node labels and relationship types
- [ ] Preserve properties (weights, metadata)
- [ ] Support incremental sync
- [ ] Write integration tests
- [ ] Document usage and examples

**Dependencies:** Milestone 2 (use `ge` for edge management)

**Success Criteria:**
- Round-trip Neo4j → FS → Neo4j preserves data
- Exported graphs are browsable with standard tools
- Performance acceptable for medium graphs (100K nodes)
- At least one real Neo4j user adopts it

**Estimated Effort:** 2-3 weeks

### Milestone 5: Application - Build System Integration

**Status:** Not Started

**Goal:** Automatic edge generation from build graphs.

**Deliverables:**
- [ ] Design integration approach (wrapper vs native)
- [ ] Implement Make integration
- [ ] Implement Ninja integration
- [ ] Track source → object edges (`derives_from`)
- [ ] Track header dependencies (`depends_on`)
- [ ] Generate incremental build insights
- [ ] Write examples for common build systems
- [ ] Document integration patterns

**Dependencies:** Milestone 2 (CLI tools)

**Success Criteria:**
- Build graph automatically becomes `.edge/` structure
- Incremental builds can query edge graph
- Works with existing Makefiles/build.ninja
- Negligible build time overhead (<5%)

**Estimated Effort:** 2-3 weeks

### Milestone 6: Application - Documentation System

**Status:** Not Started

**Goal:** Reference implementation for documentation with bidirectional links.

**Deliverables:**
- [ ] Design doc linking conventions
- [ ] Implement `doc-graph` tool
- [ ] Automatic backlink generation
- [ ] Dead link detection
- [ ] Orphan document finder
- [ ] Visual documentation map generator
- [ ] Static site generator integration
- [ ] Example documentation site using convention

**Dependencies:** Milestone 2 (CLI tools)

**Success Criteria:**
- Documentation graphs are browsable
- Backlinks automatically maintained
- Integration with common doc tools (MkDocs, Hugo, etc.)
- At least one real doc site using it

**Estimated Effort:** 2-3 weeks

### Milestone 7: FUSE Prototype

**Status:** Not Started

**Goal:** Implement virtual `.edge/` for enhanced features and UX.

**Deliverables:**
- [ ] Design FUSE architecture
- [ ] Implement virtual `.edge/` on all nodes
- [ ] Implement property storage backend
- [ ] Add query interface (pattern matching, filtering)
- [ ] Implement edge indexing for performance
- [ ] Add automatic edge inference hooks
- [ ] Write performance benchmarks
- [ ] Document deployment and configuration

**Dependencies:** Milestones 1-6 (understand usage patterns first)

**Success Criteria:**
- Performance comparable to pure convention (<10% overhead)
- Virtual `.edge/` always present
- Query interface demonstrably more powerful
- Backward compatible with pure convention

**Estimated Effort:** 4-6 weeks

### Milestone 8: Standardization Exploration

**Status:** Not Started

**Goal:** Explore path to broader adoption and standardization.

**Deliverables:**
- [ ] Write formal specification
- [ ] Document reference implementation
- [ ] Create comprehensive test suite
- [ ] Engage with POSIX working groups
- [ ] Present at conferences (FOSDEM, etc.)
- [ ] Build community consensus
- [ ] Evaluate kernel module feasibility

**Dependencies:** Milestone 7 (need proven implementation)

**Success Criteria:**
- Specification is clear and complete
- Multiple independent implementations possible
- Community interest demonstrated
- Path to standardization identified

**Estimated Effort:** Ongoing (months)

## Research Questions

Throughout development, investigate:

### Semantics
- How to handle edge type namespacing?
- Should edge types be extensible via plugins?
- How to version edge type schemas?
- What's the minimal standard library?

### Performance
- How do large edge graphs perform?
- What's the optimal indexing strategy?
- Cache hit rates for edge traversals?
- Scalability limits?

### Usability
- What edge types do users actually need?
- Common confusion points?
- How to teach this to new users?
- Tool integration friction?

### Integration
- How to integrate with existing tools?
- What breaks with the convention?
- Migration paths for existing projects?
- Adoption barriers?

## Experimentation

### Benchmarks
- Edge creation performance (target: <1ms per edge)
- Edge traversal performance (target: <10% overhead)
- Query performance at scale (target: <100ms for 10K edges)
- Storage overhead (target: <5% for typical projects)

### User Studies
- Usability testing with developers
- Interviews about workflow integration
- Case studies from real projects
- Comparison with alternatives (tags, symlinks, etc.)

### Proof of Concepts
- Static analyzer generating edges
- IDE integration showing edges visually
- Graph visualization dashboard
- Real-time edge inference

## Risks and Mitigations

### Risk: Convention too manual, low adoption

**Mitigation:** Milestone 2 CLI tools reduce friction early

### Risk: Performance unacceptable at scale

**Mitigation:** Benchmark early (Milestone 1), FUSE optimization (Milestone 7)

### Risk: Limited real-world use cases

**Mitigation:** Build concrete applications (Milestones 4-6) before generalizing

### Risk: Standards bodies uninterested

**Mitigation:** Prove value first (Milestones 1-7), engage incrementally

## Dependencies Graph

```
Milestone 1 (Validation)
    ├─→ Milestone 2 (CLI)
    │       ├─→ Milestone 3 (Shell)
    │       ├─→ Milestone 4 (Neo4j)
    │       ├─→ Milestone 5 (Build)
    │       └─→ Milestone 6 (Docs)
    └─→ Milestone 7 (FUSE)
            └─→ Milestone 8 (Standards)
```

## Timeline Estimate

**Aggressive:** 6 months (Milestones 1-7)
**Realistic:** 9-12 months (Milestones 1-7)
**Conservative:** 18 months (Milestones 1-8)

Assumes part-time development effort. Full-time could cut timelines by 50%.

---

**Last Updated:** 2025-11-19
