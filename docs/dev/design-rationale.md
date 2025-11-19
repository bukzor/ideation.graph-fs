# Design Rationale

**Read this when:**
- Questioning why a design decision was made
- Considering alternative approaches
- Understanding tradeoffs in the current design

## Core Decisions

### Decision: Use `.edge/` Convention vs New Path Syntax

**Decision:** Express edge types through `.edge/<type>/` directories rather than new path syntax like `@type/` or `/path:type:to/file`.

**Alternatives Considered:**

**Option A: Inline sigils** (`/projects/@depends_on/lib`)
- Pros: Shorter paths, clear visual distinction
- Cons: Ambiguous with shell history expansion (@), requires path parser changes

**Option B: Colon separators** (`/projects:depends_on/lib`)
- Pros: Common in other systems (Neo4j uses this)
- Cons: Already has meaning in POSIX (device:path), breaks tab completion

**Option C: `.edge/` directories** (chosen)
- Pros: Pure POSIX, discoverable (`ls .edge/`), no parser changes needed
- Cons: Slightly longer paths, `.` prefix makes invisible by default

**Rationale:**

The `.edge/` approach works TODAY with zero implementation. Users can start using it immediately:

```bash
mkdir -p .edge/depends_on
ln -s /lib/foo .edge/depends_on/
ls .edge/  # Discoverable: shows all edge types
```

Every alternative requires either:
- Shell modifications (sigil expansion)
- Path parser changes (colon handling)
- User education on new syntax

**Tradeoffs:**

Gave up: Brevity, visual distinctiveness
Gained: Immediate usability, discoverability, zero dependencies

### Decision: Symlinks vs Hardlinks for Edges

**Decision:** Use symlinks to represent edges, not hardlinks.

**Alternatives Considered:**

**Option A: Hardlinks**
- Pros: Same inode = truly "same file", changes appear everywhere
- Cons: Can't link directories (limitation on most systems), can't cross filesystems, no semantic distinction from "duplicate"

**Option B: Symlinks** (chosen)
- Pros: Work for directories, cross filesystems, show relationship (`->` in ls), breakage is visible
- Cons: Can break if target moves, slightly slower resolution

**Option C: Hybrid** (hardlinks for files, symlinks for directories)
- Pros: Best of both
- Cons: Inconsistent semantics, confusing mental model

**Rationale:**

Edges represent relationships, not identity. Symlinks semantically mean "this points to that" which matches edge semantics better than hardlinks' "these are the same thing."

Practical requirements:
- Must work for directories (build graph includes directory nodes)
- Must cross filesystems (dependencies often in /usr/lib, projects in /home)
- Broken edges should be visible (dependency removed → find dangling symlinks)

**Tradeoffs:**

Gave up: Guaranteed consistency (broken symlinks possible), minor performance
Gained: Directory support, cross-filesystem, semantic clarity

### Decision: User-Defined Edge Types vs Fixed Schema

**Decision:** No fixed schema. Users create arbitrary edge types by mkdir'ing them.

**Alternatives Considered:**

**Option A: Fixed set** (only `contains`, `depends_on`, `references`)
- Pros: Consistent, tooling simpler, interoperability guaranteed
- Cons: Limiting, can't handle unforeseen use cases

**Option B: Extensible schema** (register new types via config)
- Pros: Balance of structure and flexibility
- Cons: Requires central registry, tooling coordination, version management

**Option C: Fully user-defined** (chosen)
- Pros: Maximum flexibility, no coordination needed, emergent conventions
- Cons: No guaranteed interoperability, namespace conflicts possible

**Rationale:**

Early in concept exploration. Don't know what edge types people actually need. Better to observe usage patterns and standardize later than constrain prematurely.

Precedent: File extensions started ad-hoc, conventions emerged (`.txt`, `.jpg`), later formalized (MIME types). Same trajectory here.

**Tradeoffs:**

Gave up: Guaranteed tool compatibility, namespace management
Gained: Experimentation freedom, quick adoption, organic convention evolution

### Decision: Convention-First vs Implementation-First

**Decision:** Establish convention using existing POSIX primitives before building tooling or filesystem implementations.

**Alternatives Considered:**

**Option A: FUSE first**
- Pros: Better UX from day 1, enforce consistency, richer features
- Cons: Deployment friction, requires daemon, debugging complexity

**Option B: Kernel module first**
- Pros: Maximum performance, native integration
- Cons: Huge development effort, platform-specific, deployment impossible

**Option C: Convention first** (chosen)
- Pros: Zero deployment, immediate validation, organic adoption possible
- Cons: Manual edge management, no enforcement, tool integration DIY

**Rationale:**

Unknown if concept is useful. Better to validate with zero-friction approach (just use mkdir/ln) before investing in implementation.

If convention fails, we learned cheaply. If it succeeds, usage patterns inform implementation design.

Precedent: Markdown started as convention (use * for emphasis), later got parsers. RSS started as XML convention, later got dedicated tools.

**Tradeoffs:**

Gave up: Polished UX, automatic features, enforcement
Gained: Rapid validation, zero barriers to experimentation, implementation guidance from real usage

### Decision: Edge Properties Storage

**Decision:** Support multiple methods (xattrs, sidecar files, filename encoding) rather than mandate one.

**Alternatives Considered:**

**Option A: Extended attributes only**
- Pros: Proper metadata storage, efficient
- Cons: Not portable (platform-specific APIs), not in version control, permission complications

**Option B: Sidecar `.properties` files**
- Pros: Portable, version-controllable, human-readable
- Cons: Extra files clutter directory, easy to desync from symlink

**Option C: Filename encoding** (`target:weight=0.9`)
- Pros: Self-contained, simple, works everywhere
- Cons: Limited expressiveness, filename length limits, parsing complexity

**Option D: Support all** (chosen)
- Pros: Use best method for each context
- Cons: Inconsistency, tooling must handle all formats

**Rationale:**

Different use cases have different constraints:
- xattrs best for performance-critical, machine-managed edges
- Sidecar files best for version-controlled, human-edited edges
- Filename encoding best for simple cases (single numeric weight)

Early to lock in. Let usage inform what's actually needed.

**Tradeoffs:**

Gave up: Consistency, simple tooling
Gained: Flexibility, broader applicability, platform independence options

## Related Decisions

### Why Not Use Existing Solutions?

**Semantic filesystems (BeOS, WinFS):** Required OS replacement. Too heavyweight. Didn't gain adoption.

**Tagging systems (macOS tags, GNFS):** Single axis (tags), no typed relationships, proprietary.

**Git object model:** Perfect for versioned content, wrong abstraction for general filesystem organization.

**Graph databases (Neo4j):** Excellent for queries, terrible for storing actual files. Complementary, not replacement.

**Union mounts (Plan 9):** Different problem (namespace composition vs relationship semantics).

## Open Questions

### Should edge types be namespaced?

Current: `depends_on` is global. Conflicts possible if two tools use same name differently.

Options:
- Namespace prefixes: `.edge/tool:depends_on/`
- Tool-specific subdirs: `.edge/mytool/depends_on/`
- Leave as-is, let conventions emerge

Impact: Tooling interoperability vs simplicity.

### How to handle bidirectional edges?

Current: Convention suggests pairs (depends_on ↔ depended_on_by) but no enforcement.

Options:
- Tooling automatically maintains inverses
- Filesystem-level hooks (inotify) keep in sync
- Leave manual, accept inconsistency

Impact: Consistency guarantees vs implementation complexity.

### Should `.edge/` always exist?

Current: Only created when needed. Missing `.edge/` = implicit contains edge.

Options:
- Virtual `.edge/` on all directories (requires FUSE)
- Auto-create on directory creation
- Leave as-is

Impact: Discoverability vs filesystem clutter.

## Evolution Path

Phase 1 decisions optimize for **validation** (zero friction, maximum flexibility).

Phase 2+ decisions will optimize for **usability** (consistent patterns, tool support).

Expect refinement as usage patterns emerge. This document will update with rationale for changes.

---

**Last Updated:** 2025-11-19
