# Full 13-Phase Development Cycle

Use this when the [compressed cycle](./SKILL.md) is insufficient — production features, multiple components, UI work, or unclear requirements.

## Phase Reference

| Phase | Name | Goal | Key Activities |
|-------|------|------|----------------|
| **1** | Specify | Define requirements | Write specs, create examples |
| **2** | Research | Gather technical context | Test libraries at REPL, discover constraints |
| **3** | Explore | Understand problem space | Load namespaces, examine data |
| **4** | Validate | Test assumptions | Edge cases, error handling |
| **5** | Design | Create solution plan | Choose data structures, design signatures |
| **6** | Develop | Build incrementally | Code at REPL, test immediately |
| **7** | JVM Unit Tests | Validate in production | Run tests for namespace |
| **8** | Browser Validation | Test in real browser | Verify DOM, events, async |
| **9** | Critique | Review implementation | Check spec alignment |
| **10** | Build | Compose components | Higher-level functions |
| **11** | Edit | Refine and polish | Remove complexity, improve naming |
| **12** | Verify | Ensure compliance | Write tests, run suite |
| **13** | Code Quality | Catch syntax issues | Run clj-kondo, fix warnings |

See [PHASE_GUIDE.md](./PHASE_GUIDE.md) for detailed checklists per phase.
