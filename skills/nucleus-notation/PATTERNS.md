# Nucleus Lambda Calculus Patterns

Patterns for using lambda calculus as a tool meta-language within Nucleus.

## Lambda Calculus as Tool Language

Nucleus treats tool usage as composable lambda expressions. This enables:

- **Abstraction**: Define reusable patterns
- **Composition**: Combine tools without fractal complexity
- **Clarity**: Explicit input-output relationships

## Core Lambda Pattern

```
lambda(input). transformation(input)
```

The lambda abstracts over the input, making the transformation explicit.

## Heredoc Pattern

Escaping strings in shell commands creates fractal complexity. The heredoc pattern eliminates this:

```bash
lambda(content). read -r -d '' VAR <<'EoC' || true
content
EoC
```

### Why This Works

| Property | Effect |
|----------|--------|
| `read -r` | Raw mode prevents backslash interpretation |
| `-d ''` | Null delimiter reads until heredoc end |
| `<<'EoC'` | Single quotes prevent variable expansion |
| `|| true` | Prevents exit on EOF |

### Example Usage

**Without heredoc (fractal escaping):**
```bash
echo "He said \"Hello, \$USER\" and then \\n continued"
```

**With heredoc (no escaping):**
```bash
read -r -d '' MSG <<'EoC' || true
He said "Hello, $USER" and then \n continued
EoC
echo "$MSG"
```

## Composable Tool Patterns

### Sequential Composition

```
lambda(x). g(f(x))
```

First apply `f`, then apply `g` to the result.

**Example - Fetch then Process:**
```
lambda(url). process(fetch(url))
```

### Parallel Composition

```
lambda(x). merge(f(x), g(x))
```

Apply both `f` and `g` independently, merge results.

**Example - Multi-source Validation:**
```
lambda(input). validate(lint(input), test(input))
```

### Conditional Composition

```
lambda(x). if predicate(x) then f(x) else g(x)
```

Choose transformation based on input properties.

**Example - Format-specific Processing:**
```
lambda(file). if is-json(file) then json-process(file) else text-process(file)
```

## Procedure Encoding Patterns

### TDD Workflow Pattern

```
lambda(spec).
  red: write-failing-test(spec)
  green: implement-minimal(spec)
  refactor: clean(implementation)
```

Encoded as:
```
[mu tau] | [Delta lambda] | RGR
Human comp AI
```

### REPL Exploration Pattern

```
lambda(question).
  read: understand(question)
  eval: execute(hypothesis)
  print: analyze(result)
  loop: refine(question)
```

Encoded as:
```
[phi fractal] | [lambda eps/phi] | REPL
Human pipe AI
```

### OODA Decision Pattern

```
lambda(situation).
  observe: gather(information)
  orient: analyze(context)
  decide: select(action)
  act: execute(decision)
```

Encoded as:
```
[phi e] | [Delta lambda inf/0] | OODA
Human tensor AI
```

## Tool Abstraction Examples

### Generic Validation Tool

```
validate = lambda(input, rules).
  results = map(lambda(rule). rule(input), rules)
  all(results)
```

### Generic Transformation Pipeline

```
pipeline = lambda(input, transforms).
  reduce(lambda(acc, f). f(acc), input, transforms)
```

### Generic Parallel Execution

```
parallel = lambda(input, tools).
  merge(map(lambda(tool). tool(input), tools))
```

## Integration with Project Skills

### Invoking Skills as Lambdas

Skills can be viewed as lambda abstractions:

```
skill:tdd = lambda(feature).
  [mu tau] | [Delta lambda] | RGR
  Human comp AI

skill:debug = lambda(issue).
  [phi fractal] | [Delta lambda inf/0] | OODA
  Human pipe AI

skill:explore = lambda(question).
  [phi e] | [lambda eps/phi] | REPL
  Human pipe AI
```

### Composition of Skills

```
implement = lambda(feature).
  spec = skill:explore(feature)
  code = skill:tdd(spec)
  verified = skill:debug(code)
  verified
```

## Anti-Patterns

### Avoid Nested Escaping

**Bad:**
```bash
cmd="echo \"$(echo 'nested \"quotes\"')\""
eval "$cmd"
```

**Good:**
```bash
read -r -d '' INNER <<'EoC' || true
nested "quotes"
EoC
echo "$INNER"
```

### Avoid Deep Lambda Nesting

**Bad:**
```
lambda(x). lambda(y). lambda(z). f(x, y, z)
```

**Good:**
```
lambda(x, y, z). f(x, y, z)
```

### Avoid Implicit State

**Bad:**
```
lambda(x). global_state += f(x); global_state
```

**Good:**
```
lambda(x, state). (f(x), updated_state)
```

## Data Model Encoding Patterns

### Complete Entity Encoding

**Simple entity:**
```
#Customer[id:mu:id, name:tau:s!, email:s!, phone:s?]
```

**Entity with inheritance:**
```
@BaseEntity[id:mu:id, createdAt:dt!, updatedAt:dt!]
#Customer⊂BaseEntity[name:tau:s!, email:s!, phone:s?, tier:e!<bronze|silver|gold>]
```

**Entity with constraints:**
```
#Order⊂BaseEntity[
  status:e!<draft|submitted|approved|shipped>,
  total:$!<>=0>,
  items:[]o,
  customer:o!
]
```

### Field Encoding Examples

| Business Meaning | Nucleus Encoding |
|------------------|------------------|
| Primary key | `id:mu:id` |
| Required name | `name:tau:s!` |
| Optional description | `description:s?` |
| Currency field | `amount:$!` |
| Non-negative amount | `amount:$!<>=0>` |
| Status enum | `status:e!<active\|inactive>` |
| Self-referencing FK | `parentId:phi:o?` |
| Collection of items | `items:[]o` |
| Percentage | `rate:%` |
| Timestamp | `createdAt:dt!` |

### Relationship Encoding Examples

**One-to-many:**
```
Customer --*→ Order (via orders)
```

**One-to-one:**
```
Order --1→ ShippingAddress (via shippingAddress)
```

**Composition (parent owns child lifecycle):**
```
Order ⊂→ LineItem
```

**Aggregation (shared reference):**
```
Employee ○→ Department
```

### Combined Procedure + Data Model Format

When adding context to chat prompts, combine procedure and data model encodings:

```
PROCEDURE: [[ProcessOrder]]
NUCLEUS: φ valid → λ items → [[ValidateItem]] ⊗ total:=Σ

DATA CONTEXT:
DM:[
  #Order⊂BaseEntity[status:e!, items:[]o, total:$!<>=0>]
  #LineItem[qty:n!<>0>, price:$!, product:o!]
]
REL:[Order⊂→LineItem]
```

### Chat Context Integration Pattern

For chat system prompts, data model nucleus encodings are added as stable context:

```
## Data Model (Nucleus Encodings)
DM:[
  @BaseEntity[id:mu:id, createdAt:dt!]
  #Customer⊂BaseEntity[name:tau:s!, email:s!]
  #Order⊂BaseEntity[status:e!, total:$!]
]
REL:[Customer--*→Order]
```

This format is:
- Compact (token-efficient)
- Cacheable (stable across requests)
- Queryable (extract class/field info)

### Generation Pattern for Gemini

When prompting Gemini to generate data model nucleus:

**Class nucleus prompt:**
```
Analyze this class and generate nucleus encoding:
Class: Order
Inherits: BaseEntity
Fields: id, status, total, items, createdAt

Generate: #Order⊂BaseEntity[id:mu:id, status:e!, total:$!, items:[]o, createdAt:dt!]
```

**Field nucleus prompt:**
```
Analyze this field and generate nucleus encoding:
Field: totalAmount
Type: decimal
Constraints: Must be >= 0

Generate: totalAmount:$!<>=0>
```

## Cross-Reference

- Main skill documentation: [SKILL.md](SKILL.md)
- Symbol reference: [SYMBOLS.md](SYMBOLS.md)
