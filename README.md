# pyze-clojure

Clojure and ClojureScript development skills for [Claude Code](https://docs.anthropic.com/en/docs/claude-code). A language companion to [pyze-workflow](https://github.com/pyze/pyze-workflow).

## Why install this

- **Claude learns your stack.** Skills cover [Integrant](https://github.com/weavejester/integrant), [Pathom 3](https://pathom3.wsscode.com/), [Replicant](https://github.com/cjohansen/replicant), [Shadow-CLJS](https://shadow-cljs.github.io/docs/UsersGuide.html), [Pyramid](https://github.com/lilactown/pyramid), and [Reitit](https://github.com/metosin/reitit) — so it generates idiomatic code instead of Java-flavored Clojure.
- **REPL-first workflows.** Claude tests assumptions at the [nREPL](https://nrepl.org/) before editing files, uses REPL introspection for semantic search, and builds features incrementally.
- **Clojure-specific quality gates.** Hooks warn on unapproved mutable state (`atom`/`volatile!`/`ref`/`agent`) and suggest REPL over grep for structural queries.
- **Structural code transforms.** [rewrite-clj](https://github.com/clj-commons/rewrite-clj) via [Babashka](https://babashka.org/) for s-expression-aware modifications — safer than regex for namespace manipulation, keyword renames, and defn transforms.

## Installation

Install alongside pyze-workflow for the full experience:

```bash
claude plugin add pyze/pyze-workflow
claude plugin add pyze/pyze-clojure
```

See the [Claude Code plugin documentation](https://docs.anthropic.com/en/docs/claude-code/plugins) for details on plugin management.

## Relationship to pyze-workflow

[pyze-workflow](https://github.com/pyze/pyze-workflow) provides language-agnostic principles (decomplection, testing patterns, error handling, caching/purity). pyze-clojure grounds those principles with Clojure-specific code examples and adds skills for Clojure frameworks and tooling.

You can use pyze-clojure standalone, but pairing it with pyze-workflow gives you PDCA cycle management, risk assessment, and quality gate hooks.

## Skills

### Coding Standards & Patterns

| Skill | Companion to | Description |
|-------|-------------|-------------|
| clojure-coding-standards | — | Functional programming principles, code organization, collection patterns, idioms |
| error-handling-clojure | [error-handling-patterns](https://github.com/pyze/pyze-workflow) | [Truss](https://github.com/taoensso/truss) assertions, [Telemere](https://github.com/taoensso/telemere) structured logging, `ex-info`, Integrant config patterns |
| decomplection-clojure | [decomplection-first-design](https://github.com/pyze/pyze-workflow) | Composition, protocols, state management approval workflow |
| testing-patterns-clojure | [testing-patterns](https://github.com/pyze/pyze-workflow) | [clojure.test](https://clojure.github.io/clojure/clojure.test-api.html) anti-patterns: `#'` var access, `with-redefs`, config divergence |
| caching-and-purity-clojure | [caching-and-purity](https://github.com/pyze/pyze-workflow) | Impure vs pure function patterns, [memoize](https://clojuredocs.org/clojure.core/memoize) correctness |
| bdd-scenarios-clojure | [bdd-scenarios](https://github.com/pyze/pyze-workflow) | [Given/When/Then](https://martinfowler.com/bliki/GivenWhenThen.html) with `deftest`, `testing` blocks, Clojure predicates |
| tool-selection-clojure | [tool-selection](https://github.com/pyze/pyze-workflow) | [clojure-lsp](https://clojure-lsp.io/) operations, REPL queries, namespace navigation |
| rewrite-clj-transforms | — | Structural code modification via [Babashka](https://babashka.org/) + [rewrite-clj](https://github.com/clj-commons/rewrite-clj) for s-expression-aware transforms |

### REPL & Development Workflow

| Skill | Description |
|-------|-------------|
| clojure-mcp-repl | Execute Clojure/ClojureScript at the REPL via [clojure-mcp](https://github.com/clojure-mcp/clojure-mcp) MCP server |
| repl-driven-development | TDD and REPL-driven workflow: write tests first, build features incrementally |
| repl-semantic-search | REPL introspection as semantic search: resolver relationships, namespace structure |

### Frameworks & Libraries

| Skill | Description |
|-------|-------------|
| integrant-lifecycle | System lifecycle with [Integrant](https://github.com/weavejester/integrant) dependency injection |
| pathom3-eql | [EQL](https://edn-query-language.org/) resolvers with [Pathom 3](https://pathom3.wsscode.com/) for data resolution |
| pyramid-state-management | Normalized state management with [Pyramid](https://github.com/lilactown/pyramid) |
| reitit-routing | Server and client routing with [Reitit](https://github.com/metosin/reitit) |
| replicant-ui | UI components with [Replicant](https://github.com/cjohansen/replicant) vDOM and hiccup syntax |
| continuous-eql | Reactive pipelines with [Missionary](https://github.com/leonoel/missionary) signal graph architecture |

### ClojureScript

| Skill | Description |
|-------|-------------|
| clojurescript-shadow-cljs | [Shadow-CLJS](https://shadow-cljs.github.io/docs/UsersGuide.html) compilation, hot reload, and build configuration |
| clojurescript-cross-platform-code | Cross-platform code with [reader conditionals](https://clojure.org/guides/reader_conditionals) for JVM TDD |

### Integrations

| Skill | Description |
|-------|-------------|
| gemini-sdk-clojure | [Google Gemini Java SDK](https://ai.google.dev/gemini-api/docs) interop: function declarations, schemas, varargs |
| wemble-gemini | Wemble's Gemini integration: chat-turn, tool data maps, context caching |
| nucleus-notation | Mathematical symbol encoding for compressed AI prompts |

## Commands

| Command | Description |
|---------|-------------|
| /code-cleanup | Clojure-specific static analysis with [clj-kondo](https://github.com/clj-kondo/clj-kondo) and Clojure skill registry |

When both pyze-workflow and pyze-clojure are installed, run both for full coverage:
- `/pyze-workflow:code-cleanup` — principle-driven analysis + Chiasmus graph analysis
- `/pyze-clojure:code-cleanup` — clj-kondo + Clojure-specific skill registry

## Hooks

- **nREPL hint** — when a `.nrepl-port` file exists, suggests using REPL over grep for structural queries
- **Mutable state warning** — warns when `atom`/`volatile!`/`ref`/`agent` is introduced without an approval comment

## Further Reading

- [Clojure Reference](https://clojure.org/reference/reader) — language reference
- [ClojureScript](https://clojurescript.org/) — ClojureScript documentation
- [Simple Made Easy](https://www.infoq.com/presentations/Simple-Made-Easy/) — Rich Hickey on decomplection
- [Effective Programs](https://www.youtube.com/watch?v=2V1FtfBDsLU) — Rich Hickey on spec and data
- [Claude Code Plugins](https://docs.anthropic.com/en/docs/claude-code/plugins) — how to install and manage plugins

## License

MIT
