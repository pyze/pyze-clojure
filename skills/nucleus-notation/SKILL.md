---
name: nucleus-notation
description: Encode behavioral directives and data models using mathematical symbols for compressed AI prompts. Use when creating Gemini prompts with data model context (DM:[...] notation) or defining compressed workflow constraints.
---

# Nucleus Mathematical Prompting

Nucleus is a framework for encoding behavioral directives using mathematical symbols. It leverages high information density and strong model embeddings for compressed, unambiguous instructions.

## Skill Boundary

This skill covers two distinct but related encoding systems:

| System | When to Use | Example |
|--------|-------------|---------|
| **Behavioral encoding** | Defining workflow constraints, collaboration modes | `[mu tau] \| [Delta lambda] \| RGR` |
| **Data model encoding** | Adding class/entity context to Gemini prompts | `DM:[#Order[status:e!]]` |

**Use THIS skill if:**
- ✅ Creating Gemini prompts that need data model context
- ✅ Encoding entity relationships compactly (`REL:[Order--*→LineItem]`)
- ✅ Defining Human-AI collaboration modes
- ✅ Specifying compressed workflow constraints

**Use DIFFERENT skill if:**
- Documentation for humans → write prose instead
- Simple instructions → use natural language

## Why Nucleus

Mathematical symbols provide:

- **Compression**: Dense encoding of complex behavioral constraints
- **Universality**: Works across languages and model architectures
- **Composability**: Symbols combine with predictable semantics
- **Precision**: Minimal ambiguity compared to natural language

## Core Syntax

The fundamental expression encodes four layers:

```
[ontological] | [operational] | LOOP
Human ⊗ AI
```

**Full form:**
```
[phi fractal euler tao pi mu] | [Delta lambda inf/0 | eps/phi Sigma/mu c/h] | OODA
Human tensor AI
```

### Breaking Down Each Layer

| Layer | Purpose | Example |
|-------|---------|---------|
| Ontological `[...]` | What the system IS | `[phi fractal euler]` - self-referential, recursive, growing |
| Operational `[...]` | How it ACTS | `[Delta lambda inf/0]` - optimize, pattern-match, handle edges |
| Loop | Execution pattern | `OODA`, `REPL`, `RGR`, `BML` |
| Collaboration | Human-AI relationship | `Human tensor AI`, `Human comp AI` |

## Quick Reference

### Ontological Principles (System Nature)

| Symbol | Name | Meaning |
|--------|------|---------|
| `phi` | phi | Self-reference, natural proportions |
| `fractal` | fractal | Self-similarity, hierarchical structure |
| `e` | euler | Growth, compounding effects |
| `tau` | tao | Observer-observed unity, minimal essence |
| `pi` | pi | Cycles, periodicity, completeness |
| `mu` | mu | Least fixed point, minimal recursion |

### Operational Directives (Execution Methods)

| Symbol | Name | Meaning |
|--------|------|---------|
| `Delta` | delta | Gradient descent, optimization |
| `lambda` | lambda | Pattern matching, abstraction |
| `inf/0` | infinity/zero | Edge cases, boundary handling |
| `eps/phi` | epsilon/phi | Tension: approximation vs perfection |
| `Sigma/mu` | sigma/mu | Tension: feature addition vs complexity reduction |
| `c/h` | c/h | Tension: speed vs atomic cleanliness |

### Control Loops

| Loop | Origin | Pattern |
|------|--------|---------|
| `OODA` | Military | Observe -> Orient -> Decide -> Act |
| `REPL` | Computing | Read -> Eval -> Print -> Loop |
| `RGR` | TDD | Red -> Green -> Refactor |
| `BML` | Lean | Build -> Measure -> Learn |

### Collaboration Operators

| Symbol | Name | Mode |
|--------|------|------|
| `tensor` | tensor product | All constraints simultaneously (one-shot perfect) |
| `comp` | composition | Human bounds constrain AI output |
| `pipe` | parallel | Equal partnership |
| `and` | intersection | Both must agree |
| `xor` | XOR | Clear handoff, no overlap |
| `arrow` | implication | Conditional automation |

## Tension Operators

Forward slashes create explicit tensions forcing balanced choices:

- `eps/phi` (approximation vs perfection): Balance approximation (good enough) against perfection (exact)
- `Sigma/mu` (feature addition vs complexity reduction): Balance adding features against reducing complexity
- `c/h` (speed vs cleanliness): Balance execution speed against atomic cleanliness

These tensions make tradeoffs explicit in the encoded directive.

## Usage Modes

### Creative Work
```
[phi fractal euler beauty] | [Delta lambda eps/phi] | REPL
Human pipe AI
```
Emphasizes growth, self-similarity, beauty with equal collaboration.

### Production Code
```
[mu tau] | [Delta lambda inf/0 eps/phi Sigma/mu c/h] | OODA
Human comp AI
```
Emphasizes minimal solutions with human-bounded output.

### Research/Exploration
```
[exists! nabla euler] | [Delta lambda inf/0] | BML
Human tensor AI
```
Emphasizes unique solutions with full constraint satisfaction.

### TDD Implementation
```
[mu tau] | [Delta lambda eps/phi] | RGR
Human comp AI
```
Emphasizes minimal recursion through red-green-refactor.

## When to Use Behavioral Encoding

**Use when:**
- Creating compressed behavioral constraints for AI prompts
- Defining collaboration modes for Human-AI interaction
- Specifying execution patterns for complex workflows

**Skip when:**
- Writing documentation for humans to read
- Simple instructions with no special constraints
- Natural language is clearer for the context

## Integration with This Project

Nucleus notation can encode our standard workflows:

**TDD Workflow:**
```
[mu tau] | [Delta lambda RGR] | RGR
Human comp AI
```

**REPL Exploration:**
```
[phi fractal] | [lambda eps/phi] | REPL
Human pipe AI
```

**Production Implementation:**
```
[mu tau] | [Delta lambda inf/0 Sigma/mu c/h] | OODA
Human comp AI
```

## Encoding Data Models

Nucleus notation extends to data model encoding, enabling compact representation of classes, fields, and relationships.

### Entity Format

```
[Marker]ClassName [⊂ Parent][fields][constraints]
```

| Component | Description | Example |
|-----------|-------------|---------|
| Marker | `@` abstract, `#` concrete | `@BaseEntity`, `#Customer` |
| ClassName | Entity name | `Order` |
| ⊂ Parent | Inheritance | `⊂ BaseEntity` |
| [fields] | Field list | `[id:mu:id, name:s!]` |

### Field Format

```
fieldName:modifier:type[nullable][constraint]
```

| Component | Values | Example |
|-----------|--------|---------|
| fieldName | Field identifier | `customerId` |
| modifier | `mu`=key, `tau`=required, `phi`=self-ref, empty | `:mu:`, `:tau:` |
| type | `:s`, `:n`, `:b`, `:d`, `:dt`, `:o`, `:e`, `:$`, `:%` | `:s`, `:$` |
| nullable | `!`=required, `?`=optional | `!`, `?` |
| constraint | `<>=0>`, `<a\|b>`, etc. | `<>=0>` |

### Relationship Format

```
Source --cardinality→ Target
```

| Cardinality | Meaning |
|-------------|---------|
| `--1→` | One-to-one |
| `--*→` | One-to-many |
| `⊂→` | Composition (owns lifecycle) |
| `○→` | Aggregation (shared) |

### Quick Reference

**Common field patterns:**
- Primary key: `id:mu:id`
- Required string: `name:tau:s!`
- Optional phone: `phone:s?`
- Currency with constraint: `amount:$!<>=0>`
- Enum status: `status:e!<active|pending|closed>`
- Self-reference: `parent:phi:o?`

**Compact class encoding:**
```
#Order⊂BaseEntity[id:mu:id, status:e!, items:[]o, total:$!<>=0>]
```

**Combined format (chat context):**
```
DM:[#Order[status:e!, total:$!], #LineItem[qty:n!, price:$!]]
REL:[Order--*→LineItem]
```

### When to Use Data Model Encoding

**Use when:**
- Adding data model context to chat prompts
- Encoding class requirements in structured output
- Documenting entity relationships compactly
- Generating Gemini prompts for data model analysis

**Skip when:**
- Full data model documentation needed (use prose)
- Target audience is non-technical

## Related Documentation

- [SYMBOLS.md](SYMBOLS.md) - Complete symbol reference tables
- [PATTERNS.md](PATTERNS.md) - Lambda calculus tool patterns

## Source

Based on [michaelwhitford/nucleus](https://github.com/michaelwhitford/nucleus).
