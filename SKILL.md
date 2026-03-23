---
name: gemini-cli-evals
description: Write, debug, fix, and iterate on behavioral evals and system prompt changes for the Gemini CLI project. Use when working with files in the evals/ directory, packages/core/src/prompts/, writing new eval test cases, investigating failing evals, promoting evals from USUALLY_PASSES to ALWAYS_PASSES, or making prompt/tool changes that need behavioral validation.
---

# Gemini CLI Evals & Prompt Engineering

## Key Paths

| Purpose | Path |
|---------|------|
| Eval test files | `evals/*.eval.ts` |
| Eval harness | `evals/test-helper.ts` |
| Eval docs | `evals/README.md` |
| Eval logs | `evals/logs/` |
| System prompt snippets | `packages/core/src/prompts/snippets.ts` |
| Prompt provider | `packages/core/src/prompts/promptProvider.ts` |
| Legacy snippets | `packages/core/src/prompts/snippets.legacy.ts` |

## Core Concepts

**Behavioral evals** verify that the model *chooses* the correct action — not just that the system works. They are the feedback loop for prompt and tool changes.

**Two policies:**
- `ALWAYS_PASSES` — runs in every CI, must pass 100%. Only promote via `/promote-behavioral-eval`.
- `USUALLY_PASSES` — skipped unless `RUN_EVALS=1`. All new evals **must** start here.

## Workflow: Writing a New Eval

1. **Identify the behavior** — what specific model decision are you testing? Tie it to a real user scenario.
2. **Fail first** — confirm the behavior fails *before* your prompt/tool change. An eval that passes without a change is likely testing something already free.
3. **Create the eval file** — add to `evals/` or extend an existing `*.eval.ts` file.
4. **Start as `USUALLY_PASSES`** — never create a new eval as `ALWAYS_PASSES`.
5. **Build and run** — see [Running Evals](#running-evals).
6. **Iterate** — if flaky, tighten the prompt or assertion; see [eval-patterns.md](references/eval-patterns.md).

### Minimal eval template

```typescript
import { describe, expect } from 'vitest';
import { evalTest } from './test-helper.js';
import { assertModelHasOutput } from '../integration-tests/test-helper.js';

describe('feature_name', () => {
  evalTest('USUALLY_PASSES', {
    name: 'descriptive name of the behavior being tested',
    prompt: `Clear, unambiguous prompt that triggers the behavior.`,
    assert: async (rig, result) => {
      assertModelHasOutput(result);
      // Check tool calls
      const toolLogs = rig.readToolLogs();
      expect(toolLogs.some(t => t.name === 'tool_name')).toBe(true);
      // Check output content
      expect(result).toMatch(/expected pattern/i);
    },
  });
});
```

For file-based scenarios, assertion patterns, and advanced setups see [eval-patterns.md](references/eval-patterns.md).

## Workflow: Making a Prompt Change

1. **Locate the section** — `snippets.ts` is composed of named render functions. Find the relevant one (e.g., `renderCoreMandates`, `renderOperationalGuidelines`).
2. **Check if a section is toggleable** — `promptProvider.ts` uses `isSectionEnabled(key)` and `withSection()`. Section keys match the render function names (e.g., `'coreMandates'`, `'primaryWorkflows'`).
3. **Prefer positive instructions** — "do X" over "do not do X". Only use negative instructions when positive ones fail.
4. **Write the eval first** — confirm it fails, then apply the prompt change.
5. **Check legacy parity** — if changing `snippets.ts`, check if `snippets.legacy.ts` needs a parallel change (used for older models).
6. **Run the snapshot test** — `packages/core/src/core/prompts.test.ts` has snapshot tests; update them with `vitest --update-snapshots`.

For prompt section reference and change patterns see [prompt-engineering.md](references/prompt-engineering.md).

## Running Evals

```bash
# Required before every run after code changes
npm run build && npm run bundle

# CI-safe (ALWAYS_PASSES only)
npm run test:always_passing_evals

# Full suite (includes USUALLY_PASSES)
npm run test:all_evals

# Single file
npx vitest run --config evals/vitest.config.ts evals/my_feature.eval.ts
```

Logs land in `evals/logs/` — `<test-name>.log` (tool call JSON), `<test-name>.stderr.log`, and `<test-name>.jsonl` (activity log, deleted on success).

## Workflow: Fixing a Failing Eval

```bash
gemini /fix-behavioral-eval
# or with a specific GH Actions run URL:
gemini /fix-behavioral-eval https://github.com/google-gemini/gemini-cli/actions/runs/<id>
```

Manual investigation steps:
1. Read `evals/logs/<test-name>.log` — inspect tool calls and their order.
2. Read `evals/logs/<test-name>.stderr.log` — check for model errors or refusals.
3. Enable verbose logs: `GEMINI_DEBUG_LOG_FILE=debug.log npm run test:all_evals`.
4. Identify whether the failure is in the **prompt** (model chose wrong action) or the **assertion** (assertion is too strict/fragile).
5. Apply minimal targeted fix; avoid changing the eval itself unless the assertion is wrong.

## Workflow: Promoting an Eval

```bash
gemini /promote-behavioral-eval
```

Requirements before promotion:
- 10+ nightly runs completed.
- 100% pass rate across all supported models in the last 7 runs.
- Never promote manually by editing the policy string.

## EvalCase API Reference

```typescript
interface EvalCase {
  name: string;                          // Test name, used for log filenames
  prompt: string;                        // Sent to the agent as the user message
  params?: {
    settings?: Record<string, unknown>;  // Rig settings (NOT tools.core — forbidden)
  };
  timeout?: number;                      // ms, default 300000 (5 min)
  files?: Record<string, string>;        // { 'path/in/testdir': 'content' }
                                         // Triggers git init + initial commit
  approvalMode?: 'default' | 'auto_edit' | 'yolo' | 'plan';  // default: 'yolo'
  assert: (rig: TestRig, result: string) => Promise<void>;
}
```

**`rig` key methods:**
- `rig.readToolLogs()` — array of `{ name, params, result }` tool call records
- `rig._lastRunStdout` — raw stdout from the agent run
- `rig._lastRunStderr` — raw stderr
- `rig.testDir` — path to the temp workspace directory
- `rig.run({ args, approvalMode, timeout, env })` — run the agent again (for multi-turn evals)
