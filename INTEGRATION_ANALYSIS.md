# Beads + Spec-Workflow Integration Analysis

**Date:** 2025-12-29
**Branch:** wes/working

## Overview

Analysis of integrating beads (distributed issue tracker for AI agents) with spec-workflow-mcp (document-driven specification workflow) for complementary use.

## Tool Comparison

| Aspect | Beads | Spec-Workflow MCP |
|--------|-------|-------------------|
| **Core Purpose** | Distributed issue tracker for AI agent coordination | Document-driven specification workflow |
| **Philosophy** | "Persistent context machine" - issues as execution units | Requirements → Design → Tasks → Implementation pipeline |
| **Architecture** | CLI + Git-backed JSONL + SQLite | MCP server + Web dashboard + Markdown files |
| **Data Model** | Issues with dependency graph | Specs with hierarchical tasks |

### Complementary Workflow

```
Spec-Workflow: PLANNING PHASE
├── Create steering docs (vision, architecture)
├── Write requirements (WHAT)
├── Write design doc (HOW)
├── Get approvals at each stage
└── Generate task breakdown

Beads: EXECUTION PHASE
├── Import tasks as issues with dependencies
├── Multiple agents query `bd ready`
├── Parallel execution without conflicts
├── Track blocking relationships
└── Persist context across sessions
```

## Integration Surface Analysis

### Beads Capabilities

- **Export:** JSONL with rich filtering (`bd export`)
- **Import:** JSONL with collision detection (`bd import`)
- **API:** RPC daemon (40+ operations), `--json` CLI flag on all commands
- **External refs:** `external_ref` field links to other systems (e.g., `jira-ABC`, `linear-TEAM-123`)
- **Existing integrations:** Linear, Jira (bidirectional sync with mapping)

Key files:
- `/cmd/bd/export.go` - Export command
- `/cmd/bd/import.go` - Import command
- `/internal/types/types.go` - Core data structures
- `/internal/rpc/protocol.go` - RPC protocol (40+ operations)
- `/cmd/bd/linear.go`, `/cmd/bd/jira.go` - Existing integrations

### Spec-Workflow Capabilities

- **Export:** None (markdown files only)
- **Import:** None
- **API:** MCP tools only (spec-status, approvals, log-implementation)
- **Task format:** Markdown checkboxes with metadata

Task structure in `tasks.md`:
```markdown
- [ ] 1.1. Task Description
  _Requirements: req1, req2_
  _Leverage: existing-code_
  Files: src/file.ts
  - Purpose: Description
  _Prompt: Role: ... | Task: ... | Restrictions: ... | Success: ..._
```

Key files:
- `/src/core/task-parser.ts` - Parses tasks from markdown
- `/src/core/task-validator.ts` - Validates task format
- `/src/tools/index.ts` - MCP tool registration
- `/src/dashboard/implementation-log-manager.ts` - Implementation tracking

## Integration Options

### Option A: Minimal — Export Tool Only (Recommended Start)

**Effort:** Low (~200 lines, spec-workflow only)

Add MCP tool to spec-workflow that exports approved tasks to beads JSONL.

**New file:** `src/tools/export-to-beads.ts`

**Field Mapping:**

| Spec-Workflow | Beads |
|---------------|-------|
| Task ID (1.1, 1.2) | Issue ID (bd-xxxx) or hierarchical (bd-xxxx.1) |
| `[ ]` pending | `open` status |
| `[-]` in-progress | `in_progress` status |
| `[x]` completed | `closed` status |
| Task hierarchy (1.1 → 1.1.1) | `parent-child` dependency |
| Sequential tasks (1.1 before 1.2) | `blocks` dependency |
| `_Requirements: ..._` | Labels |
| `_Prompt: ..._` | Description field |
| Task description | Title |
| spec name + task ID | `external_ref: "spec:{specName}:{taskId}"` |

**Usage:**
```bash
# Export from spec-workflow
spec-workflow export-to-beads --spec user-auth > tasks.jsonl

# Import to beads
bd import -i tasks.jsonl
```

**Beads changes:** None

---

### Option B: Bidirectional Sync

**Effort:** Medium (~350 lines, spec-workflow only)

Add export + status sync back to spec-workflow.

**New files:**
- `src/tools/export-to-beads.ts` (~200 lines)
- `src/tools/sync-from-beads.ts` (~150 lines)

**Workflow:**
1. Export approved tasks → beads
2. Agents work via beads (`bd ready`, `bd close`)
3. Sync status back: `bd export --json | spec-workflow sync-from-beads`
4. tasks.md checkboxes updated automatically

**Beads changes:** None

---

### Option C: Native Beads Connector

**Effort:** Higher (~1050 lines total, mostly beads)

Add `bd spec-workflow` command to beads (mirrors Linear/Jira pattern).

**New files in beads:**
- `cmd/bd/spec-workflow.go` (~400-500 lines)
- `internal/specworkflow/client.go` (~200 lines)
- `internal/specworkflow/mapping.go` (~150 lines)
- `internal/specworkflow/sync.go` (~300 lines)

**Usage:**
```bash
bd config set specworkflow.path /path/to/project
bd spec-workflow pull --spec user-auth
bd spec-workflow push
bd spec-workflow sync
```

**Spec-workflow changes:** Minimal (maybe REST endpoints)

## Recommendation

**Start with Option A** — it's the 80/20 solution:

| Pros | Cons |
|------|------|
| Single file addition (~200 lines) | Manual re-export if tasks change |
| No beads changes required | One-way only |
| Immediate value | No auto-sync back |
| Can evolve to B/C later | — |

## Implementation Summary

| Option | Spec-Workflow Changes | Beads Changes | Total New Code |
|--------|----------------------|---------------|----------------|
| A | ~200 lines | 0 | ~200 lines |
| B | ~350 lines | 0 | ~350 lines |
| C | ~50 lines | ~1000 lines | ~1050 lines |

## Next Steps

1. Implement Option A (`export-to-beads` tool in spec-workflow)
2. Test with a real spec → beads workflow
3. Evaluate need for bidirectional sync (Option B)
4. Consider native connector (Option C) if heavy usage

## Repository Locations

- **Beads:** `/home/wes/furientis/dev/tools/beads` (branch: `wes/working`)
- **Spec-Workflow:** `/home/wes/furientis/dev/tools/spec-workflow-mcp`
