# Nucleus Symbol Reference

Complete reference for all Nucleus mathematical symbols.

## Ontological Principles

These define what the system IS - its fundamental nature.

| Symbol | Unicode | ASCII | Meaning | Use When |
|--------|---------|-------|---------|----------|
| phi | U+03C6 | `phi` | Self-reference, natural proportions | System should recognize patterns in its own processing |
| fractal | - | `fractal` | Self-similarity, hierarchical structure | Problems have recursive/nested structure |
| e | - | `e` / `euler` | Growth, compounding effects | Solutions should build incrementally |
| tau | U+03C4 | `tau` / `tao` | Observer-observed unity, minimal essence | Emphasize simplicity and core principles |
| pi | U+03C0 | `pi` | Cycles, periodicity, completeness | Recognize recurring patterns |
| mu | U+03BC | `mu` | Least fixed point, minimal recursion | Find simplest solution that works |

### Ontological Combinations

| Combination | Interpretation |
|-------------|----------------|
| `[phi fractal]` | Recursive self-reference - meta-level reasoning |
| `[mu tau]` | Minimal essence - simplest possible solution |
| `[e phi]` | Growing self-awareness - incremental understanding |
| `[pi mu]` | Complete minimal cycles - full but minimal solutions |

## Operational Directives

These define HOW the system acts - its execution methods.

| Symbol | Unicode | ASCII | Meaning | Use When |
|--------|---------|-------|---------|----------|
| Delta | U+0394 | `Delta` | Gradient descent, optimization | Iteratively improve toward goal |
| lambda | U+03BB | `lambda` | Pattern matching, abstraction | Extract and apply patterns |
| inf | U+221E | `inf` | Infinity, unbounded | Consider unlimited cases |
| 0 | - | `0` | Zero, nothing | Consider edge/null cases |
| epsilon | U+03B5 | `eps` | Approximation | Good-enough solutions |
| Sigma | U+03A3 | `Sigma` | Summation, feature addition | Accumulate capabilities |
| c | - | `c` | Speed (light speed) | Optimize for performance |
| h | - | `h` | Planck constant, atomic | Optimize for cleanliness |

### Tension Pairs

Tension operators (forward slash `/`) create explicit tradeoffs:

| Tension | Interpretation | Balance |
|---------|----------------|---------|
| `inf/0` | Infinity vs zero | Handle both unbounded and edge cases |
| `eps/phi` | Approximation vs perfection | Balance good-enough against exact |
| `Sigma/mu` | Addition vs reduction | Balance features against complexity |
| `c/h` | Speed vs cleanliness | Balance performance against atomic operations |

### Operational Combinations

| Combination | Interpretation |
|-------------|----------------|
| `[Delta lambda]` | Optimize through pattern recognition |
| `[Delta lambda inf/0]` | Optimize patterns including edge cases |
| `[lambda eps/phi]` | Pattern match with practical tolerance |
| `[Delta Sigma/mu c/h]` | Full optimization with all tensions |

## Control Loops

Execution patterns defining iteration style:

| Loop | Full Form | Steps | Best For |
|------|-----------|-------|----------|
| `OODA` | Observe-Orient-Decide-Act | 4-step military decision cycle | Dynamic situations, adaptation |
| `REPL` | Read-Eval-Print-Loop | 4-step interactive cycle | Exploration, experimentation |
| `RGR` | Red-Green-Refactor | 3-step TDD cycle | Test-driven development |
| `BML` | Build-Measure-Learn | 3-step lean cycle | Research, hypothesis testing |

### Loop Selection Guide

| Context | Recommended Loop |
|---------|------------------|
| Production code | OODA or RGR |
| Exploration | REPL |
| Feature development | RGR |
| Research/experimentation | BML |
| Dynamic problem solving | OODA |

## Collaboration Operators

Define Human-AI relationship modes:

| Symbol | Unicode | ASCII | Name | Mode |
|--------|---------|-------|------|------|
| tensor | U+2297 | `tensor` | Tensor product | All constraints simultaneously |
| comp | U+2218 | `comp` | Composition | Human bounds constrain AI |
| pipe | U+007C | `pipe` | Parallel | Equal partnership |
| and | U+2227 | `and` | Intersection | Both must agree |
| xor | U+2295 | `xor` | XOR | Clear handoff |
| arrow | U+2192 | `arrow` | Implication | Conditional automation |

### Collaboration Selection Guide

| Scenario | Operator | Reason |
|----------|----------|--------|
| One-shot perfect execution | `tensor` | All constraints must be satisfied |
| Human-supervised work | `comp` | Human sets boundaries |
| Pair programming style | `pipe` | Equal contribution |
| Approval-gated workflow | `and` | Requires agreement |
| Clear handoffs | `xor` | No overlap in responsibility |
| Automated with triggers | `arrow` | If-then automation |

## Symbol Encoding

When writing nucleus notation:

### Unicode (preferred for display)
```
[phi fractal e] | [Delta lambda Sigma/mu] | OODA
Human tensor AI
```

### ASCII (for compatibility)
```
[phi fractal euler] | [Delta lambda inf/0 | eps/phi Sigma/mu c/h] | OODA
Human tensor AI
```

### Minimal Form
For simple constraints, abbreviate:
```
[mu] | [Delta] | RGR
```

## Data Model Symbols

Nucleus notation can encode data models (classes, fields, relationships) in addition to procedures.

### Reused Ontological Symbols for Data Models

| Symbol | Procedure Use | Data Model Use | Example |
|--------|---------------|----------------|---------|
| `mu` | Least fixed point | Primary key (minimal identity) | `id:mu:id` |
| `tau` | Minimal essence | Required/core field (essential) | `name:tau:s!` |
| `phi` | Self-reference | Self-referencing FK | `parent:phi:o?` |
| `fractal` | Hierarchical structure | Inheritance hierarchy | `@BaseEntity` |

### Data Model Type Abbreviations

| Symbol | Type | Example |
|--------|------|---------|
| `:s` | String/Text | `name:s!` |
| `:n` | Number/Integer/Decimal | `count:n` |
| `:b` | Boolean | `active:b!` |
| `:d` | Date | `birthDate:d` |
| `:dt` | DateTime | `createdAt:dt!` |
| `:o` | Object/Reference | `customer:o` |
| `:id` | Identifier (key type) | `id:mu:id` |
| `:e` | Enum | `status:e!` |
| `:$` | Currency/Decimal | `total:$!` |
| `:%` | Percentage | `rate:%` |
| `:?` | Unknown/Any | `data:?` |

### Field Modifier Symbols

| Symbol | Meaning | Position | Example |
|--------|---------|----------|---------|
| `mu` | Primary key | Before type | `id:mu:id` |
| `tau` | Required/essential | Before type | `name:tau:s!` |
| `phi` | Self-referencing | Before type | `parent:phi:o?` |
| `!` | Non-null/required | After type | `name:s!` |
| `?` | Optional/nullable | After type | `phone:s?` |
| `[]` | Collection/array | Before type | `items:[]o` |

### Relationship Symbols

| Symbol | Meaning | Example |
|--------|---------|---------|
| `⊂` | Inheritance (is-a) | `Customer ⊂ BaseEntity` |
| `--1→` | One-to-one relationship | `Order --1→ Customer` |
| `--*→` | One-to-many relationship | `Customer --*→ Order` |
| `--?→` | Unknown cardinality | `Entity --?→ Other` |
| `⊂→` | Composition (owns) | `Order ⊂→ LineItem` |
| `○→` | Aggregation (shared) | `Employee ○→ Department` |

### Entity Markers

| Symbol | Meaning | Example |
|--------|---------|---------|
| `@` | Abstract class/base entity | `@BaseEntity[id:mu:id]` |
| `#` | Concrete class/entity | `#Customer⊂BaseEntity[...]` |

### Constraint Encoding

| Pattern | Meaning | Example |
|---------|---------|---------|
| `<>=0>` | Greater than or equal to 0 | `amount:$!<>=0>` |
| `<>0>` | Greater than 0 | `quantity:n!<>0>` |
| `<a\|b\|c>` | Enum values | `status:e!<active\|pending\|closed>` |
| `<len<=100>` | Length constraint | `name:s!<len<=100>` |

### Compact Data Model Format

```
DM:[
  @BaseEntity[id:mu:id, createdAt:dt!, updatedAt:dt!]
  #Customer⊂BaseEntity[name:tau:s!, email:s!, phone:s?]
  #Order⊂BaseEntity[status:e!, total:$!<>=0>]
]
REL:[Customer--*→Order]
```

## Cross-Reference

- Main skill documentation: [SKILL.md](SKILL.md)
- Tool patterns: [PATTERNS.md](PATTERNS.md)
