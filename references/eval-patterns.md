# Eval Patterns Reference

## Assertion Strategies

### Tool call assertions (most reliable)
Check that the agent used the right tool, not just that output text looks right.

```typescript
assert: async (rig) => {
  const logs = rig.readToolLogs();
  // Tool was called at all
  expect(logs.some(t => t.name === 'write_file')).toBe(true);
  // Tool called with specific params
  const call = logs.find(t => t.name === 'edit_file');
  expect(call?.params?.path).toMatch(/src\/index\.ts/);
  // Tool call count (e.g. frugality evals)
  const reads = logs.filter(t => t.name === 'read_file');
  expect(reads.length).toBeLessThanOrEqual(3);
},
```

### File content assertions (for code changes)
Read the modified file from `rig.testDir` after the run.

```typescript
import fs from 'node:fs';
import path from 'node:path';

assert: async (rig) => {
  const content = fs.readFileSync(
    path.join(rig.testDir!, 'src/index.ts'), 'utf8'
  );
  expect(content).toContain('export function newFeature');
  expect(content).not.toContain('TODO');
},
```

### Output text assertions (least reliable — use sparingly)
Only assert on output text when the behavior is the *text response itself*.

```typescript
assert: async (rig, result) => {
  assertModelHasOutput(result);         // non-empty, not an error
  expect(result).toMatch(/Cherry/i);    // case-insensitive
  expect(result).not.toMatch(/Apple/i); // negative assertion
},
```

### XML-structured output assertions
Ask the model to emit a specific XML block; parse it for reliable extraction.

```typescript
// In prompt:
// "Provide the answer as: <results><item>...</item></results>"

assert: async (rig, result) => {
  expect(result).toMatch(/<results>[\s\S]*<item>expected value<\/item>/i);
},
```

---

## File Fixture Patterns

### Basic file setup
```typescript
evalTest('USUALLY_PASSES', {
  name: 'agent edits existing TypeScript file',
  files: {
    'src/utils.ts': `export function add(a: number, b: number) {\n  return a + b;\n}\n`,
    'src/utils.test.ts': `import { add } from './utils';\ntest('add', () => expect(add(1,2)).toBe(3));\n`,
    'package.json': JSON.stringify({ name: 'test', scripts: { test: 'vitest' } }, null, 2),
  },
  prompt: 'Add a subtract function to src/utils.ts with a corresponding test.',
  assert: async (rig) => {
    const content = fs.readFileSync(path.join(rig.testDir!, 'src/utils.ts'), 'utf8');
    expect(content).toContain('subtract');
  },
});
```

### Agent definition files (sub-agent evals)
Files under `.gemini/agents/*.md` are auto-acknowledged by the harness.

```typescript
files: {
  '.gemini/agents/my-agent.md': `---\nname: my-agent\ndescription: Does X\n---\n# My Agent\n...`,
},
```

### Simulating hierarchical memory
The harness doesn't load real `~/.gemini/` files. Inject the tags directly in the prompt:

```typescript
prompt: `
<global_context>
Global instruction here.
</global_context>

<project_context>
Project instruction here (wins over global).
</project_context>

What instruction applies?`,
```

---

## Approval Mode Selection

| Mode | When to use |
|------|-------------|
| `'yolo'` (default) | Most evals — agent acts autonomously without confirmation prompts |
| `'plan'` | Testing plan mode behavior specifically |
| `'auto_edit'` | Testing file edit confirmation flows |
| `'default'` | Testing interactive confirmation behavior |

---

## Multi-Turn Evals

Call `rig.run()` again inside `assert` for follow-up turns:

```typescript
assert: async (rig) => {
  // First turn already ran; now do a follow-up
  const result2 = await rig.run({
    args: 'Now add error handling to the function you just wrote.',
    approvalMode: 'yolo',
  });
  expect(result2).toMatch(/try|catch|error/i);
},
```

---

## Settings / Params Patterns

```typescript
params: {
  settings: {
    security: {
      folderTrust: { enabled: true },  // Required when files are provided
    },
  },
},
```

**Forbidden:** `settings.tools.core` — evals must test against the full default tool set.

---

## Common Pitfalls

| Pitfall | Fix |
|---------|-----|
| Eval passes without prompt change | Confirm failure first; assertion may be testing free behavior |
| Flaky on text assertions | Switch to tool call or file content assertions |
| Test hangs | Add `timeout: 60000` (ms); default is 300000 |
| `git` commands fail in eval | Add `files: {}` to trigger git init, or check `folderTrust` setting |
| `node_modules` missing in test dir | Harness auto-symlinks; if failing, check `symlinkNodeModules` in test-helper.ts |
| Unauthorized tool error | The agent tried a tool not in the allowed set; check `approvalMode` |

---

## Existing Eval Files — Quick Reference

| File | What it tests |
|------|---------------|
| `frugalReads.eval.ts` | Agent minimizes unnecessary `read_file` calls |
| `frugalSearch.eval.ts` | Agent minimizes unnecessary search calls |
| `plan_mode.eval.ts` | Agent enters/exits plan mode correctly |
| `hierarchical_memory.eval.ts` | `<global/extension/project_context>` priority |
| `save_memory.eval.ts` | `save_memory` tool usage and persistence |
| `tool_output_masking.eval.ts` | Sensitive data not leaked in tool output |
| `gitRepo.eval.ts` | Git-aware behaviors (commit, status) |
| `subagents.eval.ts` | Sub-agent delegation |
| `concurrency-safety.eval.ts` | No parallel mutations to same files |
| `answer-vs-act.eval.ts` | Agent answers inquiries vs. acts on directives |
| `ask_user.eval.ts` | Agent uses `ask_user` tool appropriately |
| `tracker.eval.ts` | Task tracker tool usage |
| `validation_fidelity.eval.ts` | Agent validates its own changes |
| `interactive-hang.eval.ts` | Agent avoids interactive commands that hang |
