# pyze-clojure

Clojure and ClojureScript development skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). Provides coding standards, REPL workflows, and companion patterns for [pyze-workflow](https://github.com/pyze/pyze-workflow)'s general principles.

## Installation

Install alongside pyze-workflow for the full experience:

```bash
claude plugin add pyze/pyze-workflow
claude plugin add pyze/pyze-clojure
```

## Skills

### Coding Standards & Patterns

| Skill | Description |
|-------|-------------|
| clojure-coding-standards | Functional programming principles, code organization, collection patterns, idioms |
| error-handling-clojure | Truss assertions, Telemere structured logging, ex-info, Integrant config patterns |
| decomplection-clojure | Clojure decomplection: composition, protocols, state management approval workflow |
| testing-patterns-clojure | clojure.test anti-patterns: `#'` var access, `with-redefs`, config divergence |
| caching-and-purity-clojure | Impure vs pure function patterns, memoize correctness |
| bdd-scenarios-clojure | Given/When/Then with `deftest`, `testing` blocks, Clojure predicates |
| tool-selection-clojure | clojure-lsp operations, REPL queries, namespace navigation |
| rewrite-clj-transforms | Structural code modification via bb + rewrite-clj for s-expression-aware transforms |

### REPL & Development Workflow

| Skill | Description |
|-------|-------------|
| clojure-mcp-repl | Execute Clojure/ClojureScript at the REPL via clojure-mcp MCP server |
| repl-driven-development | TDD and REPL-driven workflow: write tests first, build features incrementally |
| repl-semantic-search | REPL introspection as semantic search: resolver relationships, namespace structure |

### Frameworks & Libraries

| Skill | Description |
|-------|-------------|
| integrant-lifecycle | System lifecycle with Integrant dependency injection |
| pathom3-eql | EQL resolvers with Pathom 3 for data resolution |
| pyramid-state-management | Normalized state management with Pyramid |
| reitit-routing | Server and client routing with Reitit |
| replicant-ui | UI components with Replicant vDOM and hiccup syntax |
| continuous-eql | Reactive pipelines with Missionary signal graph architecture |

### ClojureScript

| Skill | Description |
|-------|-------------|
| clojurescript-shadow-cljs | Shadow-CLJS compilation, hot reload, and build configuration |
| clojurescript-cross-platform-code | Cross-platform code with reader conditionals for JVM TDD |

### Integrations

| Skill | Description |
|-------|-------------|
| gemini-sdk-clojure | Google Gemini Java SDK interop: function declarations, schemas, varargs |
| wemble-gemini | Wemble's Gemini integration: chat-turn, tool data maps, context caching |
| nucleus-notation | Mathematical symbol encoding for compressed AI prompts |

## Commands

| Command | Description |
|---------|-------------|
| /code-cleanup | Clojure-specific static analysis with clj-kondo and Clojure skill registry |

When both pyze-workflow and pyze-clojure are installed, run both for full coverage:
- `/pyze-workflow:code-cleanup` — principle-driven analysis + Chiasmus graph analysis
- `/pyze-clojure:code-cleanup` — clj-kondo + Clojure-specific skill registry

## Hooks

- **nREPL hint** — when a `.nrepl-port` file exists, suggests using REPL over grep for structural queries
- **Mutable state warning** — warns when `atom`/`volatile!`/`ref`/`agent` is introduced without an approval comment

## License

MIT
