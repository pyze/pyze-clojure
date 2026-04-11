# Pyze Clojure Plugin

Claude Code plugin providing Clojure/ClojureScript development skills, coding standards, and REPL workflows.

## Project Structure
- `skills/` - Skill definitions (SKILL.md files with frontmatter)
- `commands/` - Slash commands (markdown with YAML frontmatter)
- `hooks/` - Hook configurations (hooks.json)
- `.claude-plugin/plugin.json` - Plugin manifest

## Companion Plugin
This plugin provides Clojure-specific skills. For general workflow infrastructure (PDCA cycle, risk assessment, etc.), install pyze-workflow.

Several skills in this plugin are companions to general principles in pyze-workflow:
- decomplection-clojure → companion to pyze-workflow's decomplection-first-design
- testing-patterns-clojure → companion to pyze-workflow's testing-patterns
- error-handling-clojure → companion to pyze-workflow's error-handling-patterns
- caching-and-purity-clojure → companion to pyze-workflow's caching-and-purity
- bdd-scenarios-clojure → companion to pyze-workflow's bdd-scenarios
- tool-selection-clojure → companion to pyze-workflow's tool-selection

## Testing Changes
- Start a new Claude Code session to test skill/command/hook changes
- Use `/reload-plugins` to reload without restarting

## Contributing
- Semver: major (breaking renames/removals), minor (new skills/hooks), patch (content fixes)
