---
name: code-cleanup
description: Static analysis for Clojure code quality violations
---

# /code-cleanup

Analyze code for violations of Clojure best practices. Uses a two-layer architecture: mechanical agents for deterministic tooling, principle-driven agents that read skill content as their detection criteria.

**Note:** When both pyze-workflow and pyze-clojure are installed, run both `/pyze-workflow:code-cleanup` (for general workflow/design violations) and `/pyze-clojure:code-cleanup` (for Clojure-specific violations) for full coverage.

## Architecture

```
/code-cleanup (orchestrator)
    │
    ├── Layer 1: Mechanical Analysis (deterministic tooling)
    │   └── Static Analysis (clj-kondo)
    │
    ├── Layer 2: Principle-Driven Analysis (skill content as criteria)
    │   ├── One agent per skill registry entry
    │   └── Generic template + skill content = detection instructions
    │
    ▼
Collect findings → Deduplicate → Create GitHub issues by category
```

**Layer 1** agents use deterministic tools (clj-kondo). Their detection criteria are tool-derived, not principle-derived.

**Layer 2** agents receive skill content as context and apply judgment to find violations. Adding principles to a skill automatically extends cleanup — no command changes needed.

## Usage

```bash
/code-cleanup           # Analyze full project
/code-cleanup src/viz/  # Analyze specific directory
```

---

## Layer 1: Mechanical Analysis

These agents use deterministic tooling. They do not derive from skills.

### Mechanical Agent 1: Static Analysis (clj-kondo)

**Concern:** Machine-detectable issues via clj-kondo.
**Detection method:** Run clj-kondo, parse structured output.

**Detects:**
- Unused bindings and imports
- Reflection warnings
- Unresolved symbols
- Arity mismatches
- Redundant expressions

**Instructions for agent:**

```
Run clj-kondo static analysis on the project:

  clj-kondo --lint src test --config '{:output {:format :edn}}' > /tmp/kondo-results.edn

Parse the EDN output. Categorize findings by severity:

**High priority:**
- Reflection warnings (performance impact)
- Unresolved symbols (potential runtime errors)
- Arity mismatches (will crash at runtime)

**Medium priority:**
- Unused bindings (code clarity)
- Unused imports (namespace hygiene)
- Redundant expressions

Return findings as a list of {file, line, category, severity, message}.
Do NOT create GitHub issues — just return findings.
```

---

## Layer 2: Principle-Driven Analysis

Each row in the skill registry spawns one analysis agent. The agent receives the skill content as context and applies judgment to find violations.

**Skill files live in the plugin, not the project.** Read them from the plugin's skills directory (the same location as this command file — look for `skills/` as a sibling of `commands/`). These skills are loaded by Claude Code automatically, so they're available in the current context — you can read them directly.

### Skill Registry

| Skill | File to Load | Detection Focus |
|-------|-------------|-----------------|
| clojure-coding-standards | CODE-ORGANIZATION.md | Function/namespace size, nesting, arity |
| clojure-coding-standards | FUNCTIONAL-PRINCIPLES.md | Mutations, purity, defaults, bang suffix, hidden state |
| clojure-coding-standards | COLLECTION-PATTERNS.md | Transducer anti-patterns |
| clojure-coding-standards | IDIOMS.md | Threading, control flow anti-patterns |
| integrant-lifecycle | SKILL.md | Missing halt-key!, lifecycle violations |
| error-handling-clojure | SKILL.md | Fail-fast violations, swallowed exceptions (Clojure-specific) |
| decomplection-clojure | SKILL.md | Entanglement in Clojure code |
| caching-and-purity-clojure | SKILL.md | Impure cached functions (Clojure-specific) |
| testing-patterns-clojure | SKILL.md | Private var access, config divergence, passthrough gaps, mocking (Clojure-specific) |
| bdd-scenarios-clojure | SKILL.md | BDD scenario quality in clojure.test |
| tool-selection-clojure | SKILL.md | Wrong tool choice (grep vs REPL vs LSP) |

To add a new analysis concern: add the skill to this table. If the skill doesn't exist yet, create it first.

### Generic Agent Prompt Template

Each principle-driven agent receives this prompt with the skill content injected:

```
You are a code quality auditor. Your task is to find violations of the
principles described in the skill content below.

**Skill content:**
{skill_content}

**Detection focus:** {focus_hint}

**Target directory:** {target_dir}

**Instructions:**
1. Read files in the target directory, prioritizing largest files first (wc -l).
2. For each principle, rule, anti-pattern, or "WRONG" example in the skill:
   - Search the codebase for instances that match the anti-pattern
   - Use grep for textual patterns, Read for contextual judgment
   - Check for "Detection:" notes in the skill for specific search strategies
   - Verify each finding — check for approved exceptions, edge cases
3. Return findings as: {file, line, principle, violation, severity}
   - HIGH: Correctness risk (runtime errors, wrong behavior, silent bugs)
   - MEDIUM: Maintainability risk (coupling, unclear code, missed conventions)
   - LOW: Style or hygiene (naming, minor idiom divergence)
4. Do NOT create GitHub issues — just return findings.
```

---

## Orchestrator Behavior

After dispatching all agents:

1. **Dispatch mechanical agents** (Layer 1)
2. **Load skill registry** — for each skill in the registry, load the skill content (these are plugin skills, already available in your context)
3. **Dispatch principle-driven agents** (one per registry row, in parallel)
4. **Merge findings** from all agents into a single collection
5. **Deduplicate** — same file+line from multiple agents = single finding
6. **Group by severity** (high -> medium -> low)
7. **Create one GitHub issue per category** with a task-list checklist
8. **Skip empty categories** — don't create issues for categories with zero findings
9. **Flag recurring patterns** — if any single violation type appears in 3+ files,
   append to the GitHub issue:

   > **Recurring pattern detected:** [violation] appears in N files. Consider running
   > `/five-whys` to determine if a skill or CLAUDE.md update would prevent this.
   > Relevant skill: [skill-name].

   For principle-driven findings, the relevant skill is the skill that spawned
   the agent. For mechanical findings, use:
   - Static analysis -> clj-kondo documentation (external)

### Issue Categories

For mechanical agents, use fixed category names:
- "Static analysis findings"

For principle-driven agents, use the skill path as category:
- "clojure-coding-standards/FUNCTIONAL-PRINCIPLES violations"
- "integrant-lifecycle violations"
- "testing-patterns-clojure violations"
- etc.

### Issue Format

```markdown
## [Category] Code Cleanup Findings

**Severity:** [High/Medium/Low counts]

### High Priority
- [ ] `file.clj:42` — Description of finding

### Medium Priority
- [ ] `file.clj:88` — Description of finding
```

## Priority Levels

| Severity | Examples |
|----------|---------|
| **High** | Reflection warnings, unapproved mutations, tests bypassing public API via `#'`, test helper passthrough gaps, arity mismatches |
| **Medium** | Unused bindings, function size >80 LOC, scattered defaults, collection anti-patterns, config divergence, excessive mocking, entanglement |
| **Low** | Namespace size warnings, nesting depth, missing bang suffix, minor idiom divergence |

## Auto-Fix Scripts

Agents can optionally produce bb + rewrite-clj scripts that fix violations, not just report them. See [rewrite-clj-transforms](../skills/rewrite-clj-transforms/) for the recipe library.

When an agent produces a fix script, the orchestrator includes it in the GitHub issue for user approval before running. Fix scripts should be self-contained bb scripts that can be run with `bb fix-script.clj`.

## Related Skills

Skills in the registry are the standards these agents enforce. Mechanical agents relate to:
- `rewrite-clj-transforms` — Structural code modification for auto-fix scripts
