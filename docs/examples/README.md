# Practical Examples

## Example 1: Software Project Dependencies

### Setup

```bash
cd ~/projects/myapp

# Create edge types
mkdir -p .edge/depends_on
mkdir -p .edge/references
mkdir -p .edge/tests
```

### Add Dependencies

```bash
# Runtime dependencies
ln -s /usr/lib/x86_64-linux-gnu/libssl.so.3 .edge/depends_on/
ln -s /usr/lib/x86_64-linux-gnu/libcrypto.so.3 .edge/depends_on/
ln -s /usr/lib/python3/dist-packages/flask .edge/depends_on/

# Documentation references
ln -s ~/docs/api-design.md .edge/references/
ln -s ~/docs/architecture.md .edge/references/

# Test coverage
ln -s tests/test_auth.py .edge/tests/
ln -s tests/test_crypto.py .edge/tests/
```

### Query

```bash
# What does this project depend on?
ls -l .edge/depends_on/

# What documentation exists?
ls -l .edge/references/

# Find all projects that depend on libssl
find ~/projects -path '*/.edge/depends_on/libssl*' -type l
```

### Add Metadata

```bash
# Mark critical dependencies
setfattr -n user.weight -v 1.0 .edge/depends_on/libssl.so.3
setfattr -n user.critical -v true .edge/depends_on/libssl.so.3

# Mark optional dependencies
setfattr -n user.weight -v 0.3 .edge/depends_on/flask
setfattr -n user.optional -v true .edge/depends_on/flask
```

## Example 2: Documentation System

### Structure

```
/docs/
├── guides/
│   ├── getting-started.md
│   └── .edge/
│       └── references/
│           ├── api.md -> ../../api/rest.md
│           └── config.md -> ../../reference/config.md
├── api/
│   ├── rest.md
│   └── .edge/
│       ├── references/
│       │   └── auth.md -> ../../security/auth.md
│       └── implements/
│           └── server.py -> /src/server.py
└── reference/
    └── config.md
```

### Navigation

```bash
cd /docs/guides/getting-started.md

# What does this guide reference?
ls ../.edge/references/

# Follow a reference
cat ../.edge/references/api.md

# What implements this API?
ls ../../api/.edge/implements/
```

### Bidirectional Links

```bash
# When creating a reference, create inverse
cd /docs/guides
ln -s ../../api/rest.md .edge/references/api.md

cd /docs/api
mkdir -p .edge/referenced_by
ln -s ../../guides/getting-started.md .edge/referenced_by/

# Now queries work both ways
# "What references the API?" → check api/.edge/referenced_by/
# "What does the guide reference?" → check guides/.edge/references/
```

## Example 3: Neo4j Database Export

### Neo4j Graph

```cypher
CREATE (alice:Person {name: "Alice", age: 30})
CREATE (bob:Person {name: "Bob", age: 28})
CREATE (acme:Company {name: "Acme Corp"})
CREATE (alice)-[:KNOWS {since: 2020}]->(bob)
CREATE (bob)-[:KNOWS {since: 2020}]->(alice)
CREATE (alice)-[:WORKS_AT {role: "Engineer"}]->(acme)
CREATE (bob)-[:WORKS_AT {role: "Manager"}]->(acme)
```

### Filesystem Export

```bash
mkdir -p /tmp/neo4j-export/nodes

# Create nodes
mkdir -p /tmp/neo4j-export/nodes/alice
echo "name=Alice" > /tmp/neo4j-export/nodes/alice/.properties
echo "age=30" >> /tmp/neo4j-export/nodes/alice/.properties
echo "labels=Person" >> /tmp/neo4j-export/nodes/alice/.properties

mkdir -p /tmp/neo4j-export/nodes/bob
echo "name=Bob" > /tmp/neo4j-export/nodes/bob/.properties
echo "age=28" >> /tmp/neo4j-export/nodes/bob/.properties
echo "labels=Person" >> /tmp/neo4j-export/nodes/bob/.properties

mkdir -p /tmp/neo4j-export/nodes/acme
echo "name=Acme Corp" > /tmp/neo4j-export/nodes/acme/.properties
echo "labels=Company" >> /tmp/neo4j-export/nodes/acme/.properties

# Create edges
mkdir -p /tmp/neo4j-export/nodes/alice/.edge/KNOWS
ln -s ../../bob /tmp/neo4j-export/nodes/alice/.edge/KNOWS/
echo "since=2020" > /tmp/neo4j-export/nodes/alice/.edge/KNOWS/bob.properties

mkdir -p /tmp/neo4j-export/nodes/alice/.edge/WORKS_AT
ln -s ../../acme /tmp/neo4j-export/nodes/alice/.edge/WORKS_AT/
echo "role=Engineer" > /tmp/neo4j-export/nodes/alice/.edge/WORKS_AT/acme.properties

mkdir -p /tmp/neo4j-export/nodes/bob/.edge/KNOWS
ln -s ../../alice /tmp/neo4j-export/nodes/bob/.edge/KNOWS/
echo "since=2020" > /tmp/neo4j-export/nodes/bob/.edge/KNOWS/alice.properties

mkdir -p /tmp/neo4j-export/nodes/bob/.edge/WORKS_AT
ln -s ../../acme /tmp/neo4j-export/nodes/bob/.edge/WORKS_AT/
echo "role=Manager" > /tmp/neo4j-export/nodes/bob/.edge/WORKS_AT/acme.properties
```

### Queries

```bash
cd /tmp/neo4j-export/nodes

# Who does Alice know?
ls alice/.edge/KNOWS/
# bob

# Where does Bob work?
ls bob/.edge/WORKS_AT/
# acme

# What's Alice's role at Acme?
cat alice/.edge/WORKS_AT/acme.properties
# role=Engineer

# Find all Person nodes
grep -l "labels=Person" */. properties
# alice/.properties
# bob/.properties

# Find all WORKS_AT relationships
find . -path '*/.edge/WORKS_AT/*' -type l
# ./alice/.edge/WORKS_AT/acme
# ./bob/.edge/WORKS_AT/acme
```

## Example 4: Build System Integration

### Project Structure

```
/project/
├── src/
│   ├── main.c
│   ├── utils.c
│   └── .edge/
│       └── depends_on/
│           ├── utils.h -> ../utils.h
│           └── config.h -> ../../include/config.h
├── include/
│   └── config.h
└── build/
    ├── main.o
    └── .edge/
        └── derives_from/
            └── main.c -> ../../src/main.c
```

### Automatic Edge Generation

```bash
#!/bin/bash
# build-with-edges.sh - Wrapper around make that populates edges

# Parse dependencies from compiler
gcc -MM src/main.c | while read dep; do
    if [[ $dep == *.h ]]; then
        mkdir -p src/.edge/depends_on
        ln -sf "$(realpath "$dep")" src/.edge/depends_on/
    fi
done

# Record what generated what
make
for src in src/*.c; do
    obj="build/$(basename "$src" .c).o"
    if [[ -f "$obj" ]]; then
        mkdir -p "$(dirname "$obj")/.edge/derives_from"
        ln -sf "$(realpath "$src")" "$(dirname "$obj")/.edge/derives_from/"
    fi
done
```

### Incremental Builds

```bash
# Find what needs rebuilding
for obj in build/*.o; do
    src=$(readlink "$(dirname "$obj")/.edge/derives_from/$(basename "$obj" .o).c")
    if [[ "$src" -nt "$obj" ]]; then
        echo "Rebuild: $obj (source changed: $src)"
    fi

    # Check dependencies too
    for dep in "$(dirname "$src")/.edge/depends_on/"*; do
        if [[ "$dep" -nt "$obj" ]]; then
            echo "Rebuild: $obj (dependency changed: $dep)"
        fi
    done
done
```

## Example 5: Content Management

### Blog Posts with Tags and Relations

```
/blog/
├── posts/
│   ├── 2025-01-intro-to-graphfs.md
│   │   └── .edge/
│   │       ├── tagged/
│   │       │   ├── filesystem -> ../../../tags/filesystem
│   │       │   └── graphs -> ../../../tags/graphs
│   │       └── references/
│   │           └── related-post -> ../2025-02-neo4j-export.md
│   └── 2025-02-neo4j-export.md
│       └── .edge/
│           ├── tagged/
│           │   ├── graphs -> ../../../tags/graphs
│           │   └── databases -> ../../../tags/databases
│           └── references/
│               └── graphfs-intro -> ../2025-01-intro-to-graphfs.md
└── tags/
    ├── filesystem/
    │   └── .edge/
    │       └── tagged_in/
    │           └── intro -> ../../posts/2025-01-intro-to-graphfs.md
    ├── graphs/
    │   └── .edge/
    │       └── tagged_in/
    │           ├── intro -> ../../posts/2025-01-intro-to-graphfs.md
    │           └── neo4j -> ../../posts/2025-02-neo4j-export.md
    └── databases/
        └── .edge/
            └── tagged_in/
                └── neo4j -> ../../posts/2025-02-neo4j-export.md
```

### Queries

```bash
# Find all posts tagged "graphs"
ls /blog/tags/graphs/.edge/tagged_in/

# Find all tags for a post
ls /blog/posts/2025-01-intro-to-graphfs.md/.edge/tagged/

# Find related posts
ls /blog/posts/2025-01-intro-to-graphfs.md/.edge/references/

# Find posts that reference this one (inverse)
find /blog/posts -path '*/.edge/references/*intro-to-graphfs*'
```

## Example 6: Package Manager

### Package Structure

```
/packages/
├── flask/
│   ├── __init__.py
│   └── .edge/
│       ├── depends_on/
│       │   ├── werkzeug -> ../../werkzeug
│       │   ├── jinja2 -> ../../jinja2
│       │   └── click -> ../../click
│       └── provides/
│           └── web-framework -> ../../.categories/web-framework
├── werkzeug/
│   └── .edge/
│       └── depended_on_by/
│           ├── flask -> ../../flask
│           └── other-apps...
└── .categories/
    └── web-framework/
        └── .edge/
            └── provided_by/
                ├── flask -> ../../../flask
                ├── django -> ../../../django
                └── fastapi -> ../../../fastapi
```

### Dependency Resolution

```bash
# Find all dependencies of flask (direct)
ls /packages/flask/.edge/depends_on/

# Find transitive dependencies
find-deps() {
    local pkg=$1
    local visited=$2

    # Avoid cycles
    if [[ "$visited" == *"$pkg"* ]]; then
        return
    fi
    visited="$visited $pkg"

    echo "$pkg"
    for dep in "/packages/$pkg/.edge/depends_on/"*; do
        if [[ -L "$dep" ]]; then
            find-deps "$(basename "$dep")" "$visited"
        fi
    done
}

find-deps flask ""

# Find what depends on werkzeug
ls /packages/werkzeug/.edge/depended_on_by/

# Find all web frameworks
ls /packages/.categories/web-framework/.edge/provided_by/
```

## Example 7: Research Paper References

```
/papers/
├── gentzen-1964/
│   ├── paper.pdf
│   └── .edge/
│       ├── cites/
│       │   ├── frege-1879 -> ../../frege-1879
│       │   └── hilbert-1928 -> ../../hilbert-1928
│       └── cited_by/
│           ├── prawitz-1965 -> ../../prawitz-1965
│           └── martin-lof-1984 -> ../../martin-lof-1984
├── frege-1879/
│   └── .edge/
│       └── cited_by/
│           ├── gentzen-1964 -> ../../gentzen-1964
│           └── ...
└── prawitz-1965/
    └── .edge/
        └── cites/
            └── gentzen-1964 -> ../../gentzen-1964
```

### Citation Analysis

```bash
# Find citation count
ls /papers/gentzen-1964/.edge/cited_by/ | wc -l

# Find most-cited papers
for paper in /papers/*/; do
    count=$(ls "$paper/.edge/cited_by/" 2>/dev/null | wc -l)
    echo "$count $(basename "$paper")"
done | sort -rn | head

# Find papers that cite both Gentzen and Frege
comm -12 \
    <(ls /papers/gentzen-1964/.edge/cited_by/ | sort) \
    <(ls /papers/frege-1879/.edge/cited_by/ | sort)

# Build citation graph (transitive)
# "What papers influenced this paper's influences?"
for cite in /papers/gentzen-1964/.edge/cites/*; do
    echo "=== $(basename "$cite") ==="
    ls "$cite/.edge/cites/" 2>/dev/null
done
```

---

These examples show the convention works for real use cases using only standard POSIX tools.
