# Commit Message Convention

## Format

```
<type>(<scope>): <subject>
```

`scope` is optional.

---

## Types

| Type | When to use |
|------|-------------|
| `feat` | New feature |
| `fix` | Bug fix |
| `docs` | Documentation changes only |
| `refactor` | Code restructuring with no behavior change |
| `test` | Adding or updating tests |
| `chore` | Build, config, or dependency changes |

---

## Scopes

| Scope | Target |
|-------|--------|
| `crewai` | `crewai_prototype/` |
| `autogen` | `autogen_prototype/` |
| `langgraph` | `langgraph_prototype/` |
| `ui` | `research_system_ui/` |
| `docker` | Docker / deployment |
| `docs` | Documentation |

---

## Examples

```
feat(crewai): add Phase 3 stderr context isolation
fix(crewai): resolve RunCommandTool array input rejection
docs: translate sub-directory READMEs to English
chore(docker): fix VITE_API_BASE_URL for inter-container networking
refactor(ui): extract SSE hook into useEventStream
test(crewai): add L3 Phase 3 repair loop integration test
```
