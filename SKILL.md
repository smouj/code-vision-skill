---
name: code-vision
description: Interactive code structure visualization and dependency mapping engine
version: 2.4.1
author: SMOUJBOT
type: analysis
dependencies:
  - graphviz>=2.50.0
  - python3>=3.9
  - pygments>=2.14.0
  - networkx>=3.0
  - matplotlib>=3.7.0
  - clang>=14.0 (optional, for C/C++)
required_env:
  - CV_OUTPUT_DIR: ${HOME}/.openclaw/code-vision/output
  - CV_MAX_NODES: 500
optional_env:
  - CV_THEME: dark|light (default: dark)
  - CV_ENGINE: dot|neato|fdp|sfdp|twopi|circo
  - CV_DEBUG: 0|1 (default: 0)
tags:
  - visualization
  - architecture
  - graphs
  - dependencies
  - structure
  - analysis
---

# Code Vision Skill

Interactive code structure visualization and dependency mapping engine that generates real-time graphs of code relationships, import/export chains, and architectural patterns.

## Purpose

Real use cases:
- **Module dependency analysis**: Detect circular dependencies in Python/TypeScript/Go projects before merge
- **Architecture drift detection**: Compare actual dependency graph against intended layered architecture
- **Impact analysis**: Visualize all affected files when changing a core module (e.g., modifying authentication service)
- **Codebase onboarding**: Generate high-level structure maps for new developers joining large monorepos
- **Technical debt identification**: Find "god modules" with >50 imports/exports across multiple directories
- **Microservice boundary validation**: Ensure services in `services/` don't accidentally import from other services
- **Package health audit**: Identify orphaned modules with zero incoming dependencies

## Scope

### Primary Commands

**`code-vision analyze <path>`**
- Analyzes codebase and generates interactive dependency graph
- `--lang <python|ts|js|go|java|rust|cpp>` (auto-detected if omitted)
- `--output <format>`: `html` (default), `svg`, `png`, `json`, `dot`
- `--depth <n>`: Follow dependencies up to N levels deep (default: 3, max: 10)
- `--filter <pattern>`: Only include files matching glob pattern (e.g., `src/**/*.ts`)
- `--exclude <pattern>`: Exclude files matching pattern (e.g., `node_modules`, `*.test.*`)
- `--group-by <dir|namespace|file>`: Node grouping strategy (default: dir)
- `--cluster`: Enable community detection clustering (Louville method)
- `--min-degree <n>`: Hide nodes with < n connections (default: 1)
- `--focus <file>`: Highlight path to specific file and its dependents/dependencies
- `--layout <engine>`: Override CV_ENGINE env var for this run

**`code-vision compare <commit1> <commit2>`**
- Compare dependency graphs between git commits
- `--metric <cyclomatic|imports|exports|coupling>`: What to compare (default: imports)
- `--threshold <0-1>`: Only show changes above similarity threshold (default: 0.1)
- `--output <html|terminal>`: Visual diff format (default: html)

**`code-vision serve <port>`**
- Launch interactive web dashboard (default port: 8080)
- `--host <ip>`: Bind address (default: 127.0.0.1)
- `--reload`: Watch filesystem for changes and auto-refresh
- `--open`: Open browser automatically

**`code-vision export <graph-id>`**
- Export previously generated graph to various formats
- `<graph-id>`: UUID from previous analysis or `latest`
- `--format <svg|png|pdf|json|dot|gexf>`
- `--zoom <1.0-5.0>`: Scale factor for raster formats
- `--with-legend`: Include graph legend in output

**`code-vision cycles <path>`**
- Detect circular dependencies with exact path tracing
- `--min-cycle <3>`: Minimum cycle length to report (default: 3)
- `--format <text|json|dot>`: Output format (default: text)
- `--fix-suggestions`: Show suggested refactoring to break cycles

### Supporting Commands

**`code-vision metrics <path>`**
- Output text report: total nodes, edges, density, average degree, max fan-in/fan-out
- `--top <n>`: Show top N most connected modules (default: 10)
- `--json`: Machine-readable output

**`code-vision validate <architecture-file>`**
- Validate current codebase against architectural constraints defined in YAML/JSON
- Example architecture file:
  ```yaml
  layers:
    - presentation: ['ui/', 'components/']
    - domain: ['core/', 'models/']
    - infrastructure: ['services/', 'repositories/']
  rules:
    - "presentation -> domain"
    - "domain -> infrastructure"
    - "forbidden: presentation -> infrastructure"
  ```

## Detailed Work Process

**1. Pre-analysis Discovery**
```
% code-vision analyze src/ --lang python --output json
```
- Scans filesystem using `find` with `--include` patterns per language
- Extracts imports using:
  - Python: `ast` module (safe), fallback to regex for syntax errors
  - TypeScript/JS: `esprima` parser, collects `import`, `require`, `export`
  - Go: `go list -json` + custom parser for imports
  - Java: `javap` or regex for `import` statements
  - Rust: `cargo metadata` + `syn` crate parsing
  - C/C++: `clang -Xclang -ast-dump` for includes
- Builds directed graph: node = file/module, edge = import/dependency

**2. Graph Cleaning & Filtering**
- Removes:
  - External/stdlib dependencies (configurable via `--external`)
  - Test files (unless `--include-tests` flag)
  - Nodes with degree 0 (unreachable) or > `CV_MAX_NODES`
  - Duplicate edges (multi-edge collapse)
- Applies `--min-degree` filter

**3. Layout & Rendering**
- Uses Graphviz layout engines:
  - `dot`: Hierarchical (default for layered architectures)
  - `neato`: Spring model (default for undirected or complex graphs)
  - `fdp`: Force-directed for large graphs (>200 nodes)
  - `sfdp`: Scalable force-directed for very large graphs (>1000 nodes)
  - `twopi`: Radial layout (good for core vs periphery)
  - `circo`: Circular layout (for ring architectures)
- Color scheme (dark theme default):
  - Blue: presentation layer
  - Green: domain/business logic
  - Orange: infrastructure/persistence
  - Red: utilities/shared
  - Purple: external dependencies
  - Gray: unclassified
- Node size proportional to fan-in + fan-out

**4. Output Generation**
- HTML: Interactive D3.js visualization with:
  - Pan/zoom
  - Click node to highlight neighbors
  - Hover for tooltip with file size, LOC, import count
  - Search box to filter nodes
  - Legend toggle
- SVG/PNG: Static high-resolution export
- JSON: Graph data for downstream tooling
- DOT: Source for manual Graphviz tweaking

**5. Interactive Dashboard (`serve`)**
- Flask/FastAPI backend on port 8080
- Serves:
  - `/`: Dashboard with graph history
  - `/graph/<uuid>`: Render specific analysis
  - `/api/analyze`: Trigger new analysis via HTTP POST
  - `/api/compare`: Compare two analyses
- WebSocket for live updates in `--reload` mode

## Golden Rules

1. **Always exclude `node_modules`, `vendor`, `target`, `build`, `.git` by default** unless explicitly overridden
2. **Respect language boundaries**: Do not infer dependencies across language ecosystems unless `--cross-lang` flag is provided
3. **Limit graph size**: Enforce `CV_MAX_NODES` hard limit (default 500). Suggest `--filter` if exceeded.
4. **Non-destructive**: Never modify source files. All artifacts written to `CV_OUTPUT_DIR` (default: `~/.openclaw/code-vision/output/` with timestamped subdirs)
5. **Security**: Do not parse files with suspicious extensions (`.exe`, `.bin`, `.so`) beyond hex dump for entropy check
6. **Performance**: Abort analysis if single file > 10MB (likely binary). Warn and skip.
7. **Git-aware**: When `code-vision compare` used, verify both commits exist locally; fetch from remote if configured
8. **Language detection**: Use file extension first, then shebang, then content sniffing. Always log detected language.
9. **Cycle detection**: Use Johnson's algorithm (all simple cycles) but limit results to top 50 longest cycles to avoid combinatorial explosion
10. **Deterministic output**: Graphviz runs must seed random generators consistently for reproducible layouts across runs on unchanged code

## Examples

### Example 1: Find circular dependencies in Python microservice
```bash
% code-vision cycles services/auth/ --lang python --format text
Circular dependencies detected (3 cycles):

Cycle 1: services/auth/models.py -> services/auth/api.py -> services/auth/decorators.py -> services/auth/models.py
Cycle 2: shared/utils.py -> services/auth/validators.py -> shared/utils.py

Break cycle #1 by:
  - Move decorators.py logic to separate module
  - Or inject decorators as callable instead of importing

--file-count: 127
--cycle-count: 2
```

### Example 2: Generate interactive HTML graph for TypeScript frontend
```bash
% code-vision analyze apps/web/ --lang ts --output html \
  --filter 'apps/web/src/**/*.tsx' \
  --exclude '**/*.test.*,**/*.spec.*' \
  --depth 2 \
  --cluster \
  --focus apps/web/src/components/Button.tsx \
  --theme dark

Analysis complete: 234 nodes, 512 edges
Output saved to: /home/user/.openclaw/code-vision/output/20260101_143022/graph.html
Interactive graph: file:///home/user/.openclaw/code-vision/output/20260101_143022/graph.html
```

### Example 3: Detect architecture violations against layered design
Given `architecture.yaml`:
```yaml
layers:
  - ui: ['src/ui/', 'src/components/']
  - domain: ['src/core/', 'src/models/']
  - infra: ['src/services/', 'src/repositories/']
rules:
  - "ui -> domain"
  - "domain -> infra"
  - "forbidden: ui -> infra"
```

```bash
% code-vision validate architecture.yaml
Validation result: FAILED
Violations:
  1. ui/src/components/Navigation.tsx -> infra/src/services/api.ts (forbidden direct import)
  2. infra/src/services/auth.ts -> domain/src/models/User.ts (reverse dependency)
  3. domain/src/models/Order.ts -> infra/src/repositories/ (infra should not be imported by domain)
  3 violations found. Consider refactoring to dependency inversion.
```

### Example 4: Compare dependency changes after refactor
```bash
% code-vision compare abc123def def456ghi --metric coupling --threshold 0.05
Comparing commits:
  Base:  abc123def (2025-12-01)
  Head:  def456ghi (2025-12-02)

Significant coupling changes (>5% threshold):
  - src/core/Engine.ts: +12 imports (from 8 to 20) [CRITICAL]
  - src/utils/helpers.ts: -9 exports (from 15 to 6) [OK]
  - src/repositories/*: No significant change

Recommendation: Review Engine.ts fan-in increase; may indicate responsibility leakage.
```

### Example 5: Serve live dashboard for team review
```bash
% code-vision serve 8080 --reload --open
Starting Code Vision dashboard...
Listening: http://127.0.0.1:8080
Watching: /home/user/projects/myapp (recursive)
Auto-reload: enabled
Open browser: http://127.0.0.1:8080

Dashboard ready. Recent analyses:
  1. 2026-01-01 14:30 - myapp (analyze) - 234 nodes
  2. 2026-01-01 13:15 - myapp (analyze) - 221 nodes
```

### Example 6: Identify "god modules" with excessive dependencies
```bash
% code-vision metrics src/ --top 10 --json
{
  "total_nodes": 456,
  "total_edges": 1234,
  "density": 0.012,
  "average_degree": 5.41,
  "top_modules_by_degree": [
    {"file": "src/core/Container.ts", "degree": 87, "type": "fan-out"},
    {"file": "src/utils/Global.ts", "degree": 65, "type": "fan-in"},
    {"file": "src/services/Api.ts", "degree": 54, "type": "fan-out"}
  ]
}
```

## Rollback Commands

If an analysis or visualization generation produces undesirable results:

**1. Remove specific analysis run**
```bash
% rm -rf ~/.openclaw/code-vision/output/20260101_143022
# or use timestamp from latest
% rm -rf ~/.openclaw/code-vision/output/$(ls -t ~/.openclaw/code-vision/output | head -1)
```

**2. Reset configuration to defaults**
```bash
% unset CV_OUTPUT_DIR CV_MAX_NODES CV_THEME CV_ENGINE CV_DEBUG
# or edit custom overrides:
% nano ~/.openclaw/config/ext/custom-overrides.json
# Remove any code-vision keys and re-run analysis
```

**3. Clear graphviz cache (if stale layouts)**
```bash
% rm -rf ~/.cache/graphviz/
# Graphviz will recreate layout cache on next run
```

**4. Undo accidental graph export overwrite**
```bash
# If you used --output graph.svg and want previous version:
% git checkout HEAD -- docs/architecture/graph.svg
# (assuming tracked in git)
```

**5. Stop running dashboard**
```bash
% pkill -f "code-vision serve"
# or get PID and kill:
% ps aux | grep code-vision
% kill <pid>
```

**6. Restore from known-good analysis snapshot**
```bash
% code-vision export snapshot-20251201 --format html --output restored.html
# where snapshot-20251201 is a previously saved graph-id
```

**7. Disable live reload if causing performance issues**
```bash
# Simply omit --reload flag in subsequent invocations
# Or set environment:
% export CV_RELOAD=0
```

## Troubleshooting

**Issue**: `Graphviz engine 'dot' not found`
**Fix**: Install graphviz: `apt-get install graphviz` or `brew install graphviz`

**Issue**: `Max nodes (500) exceeded. Analysis aborted.`
**Fix**: Use `--filter` to narrow scope or increase limit:
```bash
% export CV_MAX_NODES=2000
% code-vision analyze . --filter 'src/main/**/*.py'
```

**Issue**: `Parser error at line 234 in file X.py`
**Fix**: Check file for syntax errors: `python -m py_compile X.py`. Use `--skip-errors` to continue with partial graph.

**Issue**: Graph appears disconnected or missing edges
**Fix**: Ensure you're not excluding files that provide imports. Check `--exclude` patterns. Use `--include-tests` if test files define critical fixtures.

**Issue**: Layout engine hangs on large graph
**Fix**: Switch engine: `--layout fdp` for large graphs. Or use `--min-degree 2` to prune leaf nodes.

**Issue**: Cycles reported but code looks fine
**Fix**: False positives from dynamic imports (e.g., `importlib.import_module`). Use `--strict` to only report static analysis cycles or `--ignore-dynamic`.

**Issue**: Colors don't match layer expectations
**Fix**: Provide custom architecture YAML via `code-vision analyze . --architecture my-arch.yaml`. Define layer mappings.

**Issue**: Dashboard not accessible on network
**Fix**: Bind to 0.0.0.0: `code-vision serve 8080 --host 0.0.0.0`. Check firewall rules.

**Issue**: Memory exhausted during analysis
**Fix**: Reduce depth: `--depth 1`. Or analyze subdirectories separately. Ensure `ulimit -n` (open files) is > 4096.

**Issue**: `pygments` not found, syntax highlighting broken in HTML
**Fix**: `pip install pygments` or `apt-get install python3-pygments`

**Issue**: Git commit compare fails with "unknown revision"
**Fix**: Fetch missing commits: `git fetch origin <commit>` or ensure commit exists locally. Use `git log --oneline --all` to verify.
```