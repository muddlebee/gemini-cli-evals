# Prompt Engineering Reference

## System Prompt Architecture

The final system prompt is assembled in two stages:

```
getCoreSystemPrompt(options)        ← snippets.ts / snippets.legacy.ts
  └─ renderPreamble
  └─ renderCoreMandates             ← Security, Context Efficiency, Engineering Standards
  └─ renderSubAgents
  └─ renderAgentSkills
  └─ renderHookContext
  └─ renderPrimaryWorkflows  OR  renderPlanningWorkflow
  └─ renderTaskTracker (optional)
  └─ renderOperationalGuidelines    ← Tone, Security Rules, Tool Usage, Interaction
  └─ renderInteractiveYoloMode
  └─ renderSandbox
  └─ renderGitRepo

renderFinalShell(basePrompt, userMemory)
  └─ renderUserMemory               ← <global_context>, <extension_context>, <project_context>
```

`promptProvider.ts` drives this via `PromptProvider.getCoreSystemPrompt()`. Each section is gated by `withSection(key, factory, guard)` which checks `isSectionEnabled(key)` (env var `GEMINI_DISABLE_SECTIONS`).

---

## Section Keys & Render Functions

| Section key (in `withSection`) | Render function | Notes |
|-------------------------------|-----------------|-------|
| `preamble` | `renderPreamble` | "You are Gemini CLI..." |
| `coreMandates` | `renderCoreMandates` | Security, efficiency, engineering standards |
| `agentContexts` | `renderSubAgents` | Sub-agent list |
| `agentSkills` | `renderAgentSkills` | Skill list |
| `hookContext` | `renderHookContext` | `<hook_context>` tag guidance |
| `primaryWorkflows` | `renderPrimaryWorkflows` | Research→Strategy→Execution lifecycle |
| `planningWorkflow` | `renderPlanningWorkflow` | Plan mode; mutually exclusive with primaryWorkflows |
| `operationalGuidelines` | `renderOperationalGuidelines` | Tone, tool usage, security rules |
| `interactiveYoloMode` | `renderInteractiveYoloMode` | YOLO mode instructions |
| `sandbox` | `renderSandbox` | macOS seatbelt / generic sandbox |
| `git` | `renderGitRepo` | Git repo instructions |

---

## Locating the Right Section to Edit

1. Identify the behavior you want to change (e.g., "agent should not commit without being asked").
2. Search for the relevant text in `snippets.ts`:
   ```bash
   grep -n "commit" packages/core/src/prompts/snippets.ts
   ```
3. Find the render function that owns it.
4. Check if the behavior is in a **leaf helper** (small private function at the bottom of `snippets.ts`) — these are easier to change in isolation.

---

## Leaf Helpers (Private Functions)

These are the small composable pieces inside `snippets.ts`. Prefer editing these over large render functions:

| Helper | Controls |
|--------|----------|
| `mandateConflictResolution` | Hierarchical memory priority rule |
| `mandateConfirm` | Ambiguity confirmation behavior |
| `mandateExplainBeforeActing` | Per-tool narration requirement |
| `mandateTopicUpdateModel` | Topic: header protocol |
| `mandateSkillGuidance` | Skill activation behavior |
| `mandateContinueWork` | Non-interactive / headless mode |
| `workflowStepResearch` | Research step wording |
| `workflowStepStrategy` | Strategy step wording |
| `toolUsageInteractive` | Background/interactive command guidance |
| `toolUsageRememberingFacts` | Memory tool usage rules |
| `gitRepoKeepUserInformed` | Git interactive clarification |

---

## Options Structs — Key Fields

### `CoreMandatesOptions`
```typescript
{
  interactive: boolean;
  hasSkills: boolean;
  hasHierarchicalMemory: boolean;   // Injects mandateConflictResolution when true
  contextFilenames?: string[];       // Shown in "Contextual Precedence" mandate
  topicUpdateNarration: boolean;    // Switches between Topic Model and Explain Before Acting
}
```

### `PrimaryWorkflowsOptions`
```typescript
{
  interactive: boolean;
  enableCodebaseInvestigator: boolean;
  enableWriteTodosTool: boolean;
  enableEnterPlanModeTool: boolean;
  enableGrep: boolean;
  enableGlob: boolean;
  approvedPlan?: { path: string };
  taskTracker?: boolean;
  topicUpdateNarration: boolean;
}
```

### `OperationalGuidelinesOptions`
```typescript
{
  interactive: boolean;
  interactiveShellEnabled: boolean;
  topicUpdateNarration: boolean;
  memoryManagerEnabled: boolean;
}
```

---

## Making a Prompt Change — Checklist

- [ ] Identify the failing behavior and write an eval that reproduces it (fails first).
- [ ] Locate the render function or leaf helper responsible.
- [ ] Prefer **positive** instructions ("do X") over negative ("do not X").
- [ ] Keep changes minimal and surgical — avoid touching unrelated sections.
- [ ] Check `snippets.legacy.ts` — if the section exists there too, apply a parallel change.
- [ ] Update snapshot tests: `npx vitest run --update-snapshots packages/core/src/core/prompts.test.ts`
- [ ] Run the eval 3 times locally to check for flakiness before submitting.
- [ ] Document the intent in the PR — "why" not "what".

---

## Legacy vs. Modern Snippets

`promptProvider.ts` selects between `snippets.ts` and `snippets.legacy.ts` based on model:

```typescript
const isModernModel = supportsModernFeatures(desiredModel);
const activeSnippets = isModernModel ? snippets : legacySnippets;
```

`snippets.legacy.ts` is used for older Gemini models that don't support modern features. When changing prompt behavior, check if the legacy file has a corresponding section and update it too.

---

## Hierarchical Memory Tags

These are **generated by `renderUserMemory`** in `snippets.ts` — not a Gemini API feature. The model is taught to respect them via `mandateConflictResolution`.

```
<loaded_context>
  <global_context>   ← ~/.gemini/ files (lowest priority)
  <extension_context> ← Extension-contributed memory
  <project_context>  ← Workspace GEMINI.md / .gemini/ (highest priority)
</loaded_context>
```

Priority rule injected into Core Mandates when `hasHierarchicalMemory: true`:
> `<project_context>` (highest) > `<extension_context>` > `<global_context>` (lowest)

---

## Env Vars for Prompt Debugging

| Variable | Effect |
|----------|--------|
| `GEMINI_SYSTEM_MD=1` | Load system prompt from `~/.gemini/system.md` instead of composing it |
| `GEMINI_WRITE_SYSTEM_MD=1` | Write the composed prompt to `~/.gemini/system.md` for inspection |
| `GEMINI_DISABLE_SECTIONS=key1,key2` | Disable specific sections (uses section keys from table above) |
| `GEMINI_DEBUG_LOG_FILE=debug.log` | Enable verbose agent logs |

To inspect the exact prompt being sent:
```bash
GEMINI_WRITE_SYSTEM_MD=1 npm run test:always_passing_evals
cat ~/.gemini/system.md
```
