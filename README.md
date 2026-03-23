## Skill: `gemini-cli-evals`

Location: `.cursor/skills/gemini-cli-evals/` (project-scoped, shared with the repo)

### Structure

```
gemini-cli-evals/
├── SKILL.md                          (139 lines — core workflows)
└── references/
    ├── eval-patterns.md              (183 lines — assertion strategies, fixtures, pitfalls)
    └── prompt-engineering.md         (175 lines — prompt architecture, section map, change checklist)
```

### What it covers

**`SKILL.md`** — the entry point, loaded when you're working on evals or prompt changes:
- Key file paths table
- Step-by-step workflows for: writing a new eval, making a prompt change, fixing a failing eval, promoting an eval
- Minimal eval template
- `EvalCase` API reference with all fields and `rig` methods

**`references/eval-patterns.md`** — loaded on demand for deeper eval work:
- Tool call, file content, XML output, and text assertion patterns with code examples
- File fixture patterns including agent definition files and hierarchical memory simulation
- Multi-turn eval pattern, settings/params patterns
- Common pitfalls table and a quick-reference map of all 23 existing eval files

**`references/prompt-engineering.md`** — loaded on demand for prompt changes:
- Full system prompt assembly diagram (section order)
- Section keys → render function mapping table
- All leaf helper functions and what they control
- Options struct fields for `CoreMandatesOptions`, `PrimaryWorkflowsOptions`, `OperationalGuidelinesOptions`
- Prompt change checklist
- Legacy vs. modern snippets explanation
- Hierarchical memory tag internals
- Env vars for prompt debugging (`GEMINI_WRITE_SYSTEM_MD`, `GEMINI_DISABLE_SECTIONS`, etc.)
