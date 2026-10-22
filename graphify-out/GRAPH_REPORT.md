# Graph Report - touhou  (2026-10-23)

## Corpus Check
- 5 semantic source files; AST status: no_code. Exclusions and reuse evidence: build-audit.json.

## Summary
- 13 nodes · 8 edges · 5 communities (3 shown, 2 thin omitted)
- Extraction: 100% EXTRACTED · 0% INFERRED · 0% AMBIGUOUS
- Token usage: per-project usage unavailable (native Codex semantic extraction).

## Graph Freshness
- Built from commit: `3627021a`
- Run `git rev-parse HEAD` and compare to check if the graph is stale.
- Run `graphify update .` after code changes (no API cost).

## Community Hubs (Navigation)
- TH01 port specification
- OP dependency closure
- TH01 task DAG
- SDL3 hardware emulation
- th

## God Nodes (most connected - your core abstractions)
1. `TH01 port specification` - 3 edges
2. `TH01 task DAG` - 2 edges
3. `OP dependency closure` - 2 edges
4. `PC-98 compatibility` - 1 edges
5. `Integration verification` - 1 edges
6. `master.lib shim` - 1 edges
7. `TH01 SDL port` - 1 edges
8. `PC98 emulator` - 1 edges
9. `VramRef` - 1 edges
10. `SDL3 hardware emulation` - 1 edges

## Surprising Connections (you probably didn't know these)
- None detected - all connections are within the same source files.

## Communities (5 total, 2 thin omitted)

### Community 0 - "TH01 port specification"
Cohesion: 0.50
Nodes (4): Host frame presentation, PC98 emulator, TH01 port specification, VramRef

### Community 1 - "OP dependency closure"
Cohesion: 0.67
Nodes (3): master.lib shim, OP dependency closure, PC-98 compatibility

### Community 2 - "TH01 task DAG"
Cohesion: 0.67
Nodes (3): Integration verification, TH01 task DAG, Worker isolation

## Knowledge Gaps
- **10 isolated node(s):** `PC-98 compatibility`, `Integration verification`, `master.lib shim`, `TH01 SDL port`, `PC98 emulator` (+5 more)
  These have ≤1 connection - possible missing edges or undocumented components. (Counts symbols only; 10 node(s) total have ≤1 connection when file, concept and rationale nodes are included.)
- **2 thin communities (<3 nodes) omitted from report** — run `graphify query` to explore isolated nodes.

## Suggested Questions
_Questions this graph is uniquely positioned to answer:_

- **What connects `PC-98 compatibility`, `Integration verification`, `master.lib shim` to the rest of the system?**
  _10 weakly-connected nodes found - possible documentation gaps or missing edges._