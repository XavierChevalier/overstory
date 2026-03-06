# Cursor CLI Runtime Adapter Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Add a `CursorRuntime` adapter so Overstory can spawn and manage agents using the Cursor CLI (`agent` binary).

**Architecture:** The adapter follows the established `AgentRuntime` interface pattern — a single self-contained class in `src/runtimes/cursor.ts` registered in the runtime registry. Cursor is a TUI runtime (like Claude Code, Copilot, Gemini) with interactive sessions in tmux panes. Instructions are deployed to `.cursor/rules/overstory.md` (Cursor's native rules system). No guard/hook deployment in v1 (same as Copilot and Gemini — Cursor has a permission system via `.cursor/cli.json` but no per-tool interception hooks).

**Tech Stack:** TypeScript (Bun), `bun:test` for tests, real filesystem via temp directories (no mocks).

---

## Key Design Decisions

### CLI Binary

The Cursor CLI binary is `agent` (installed via `curl https://cursor.com/install -fsS | bash`). All commands use this binary name.

### Instruction Path

`.cursor/rules/overstory.md` — Cursor reads `.cursor/rules/` files as context rules. This avoids conflicts with `CLAUDE.md` (Claude Code), `AGENTS.md` (Codex/OpenCode), and `GEMINI.md` (Gemini). The Cursor CLI also reads `CLAUDE.md` and `AGENTS.md` from the project root, but using the native rules directory is cleaner.

### Permission Mode Mapping

- `bypass` → `--yolo` flag (force-allows all commands unless explicitly denied)
- `ask` → no flag (default interactive approval mode)

### System Prompt Handling

`appendSystemPrompt` and `appendSystemPromptFile` are **ignored** — the `agent` CLI has no `--append-system-prompt` equivalent. Role definitions are deployed to `.cursor/rules/overstory.md` via `deployConfig()`.

### TUI Readiness Detection

Cursor CLI uses an interactive TUI similar to Claude Code. Two signals required (AND logic):
- **Prompt indicator:** `❯` (U+276F) or `>` at line start
- **Status bar indicator:** `shift+tab` (mode rotation hint) or `agent` keyword in pane

No trust dialog phase — use `--trust` flag for worktrees to skip trust prompts.

### Transcript Parsing

Cursor's `--output-format stream-json` produces NDJSON events. The `system/init` event carries `model`; `assistant` events carry text; `result` event carries `duration_ms`. **No token usage** is available in the NDJSON output, so `parseTranscript` parses what's available (model from init event, zero tokens). Token usage may become available in future Cursor CLI versions.

### Beacon Verification

Not defined (defaults to `true`) — like Claude Code and Copilot, the TUI may swallow Enter during late initialization.

### Guard Deployment

None in v1 — same as Copilot and Gemini. Cursor's permission system (`~/.cursor/cli-config.json` or `.cursor/cli.json`) exists but no per-tool interception hooks. The `_hooks` parameter in `deployConfig` is unused.

---

## Task 1: Write failing tests for CursorRuntime identity

**Files:**
- Create: `src/runtimes/cursor.test.ts`

**Step 1: Write the failing test**

```typescript
import { describe, expect, test } from "bun:test";
import { CursorRuntime } from "./cursor.ts";

describe("CursorRuntime", () => {
	const runtime = new CursorRuntime();

	describe("id and instructionPath", () => {
		test("id is 'cursor'", () => {
			expect(runtime.id).toBe("cursor");
		});

		test("instructionPath is .cursor/rules/overstory.md", () => {
			expect(runtime.instructionPath).toBe(".cursor/rules/overstory.md");
		});
	});
});
```

**Step 2: Run test to verify it fails**

Run: `bun test src/runtimes/cursor.test.ts`
Expected: FAIL — `Cannot find module "./cursor.ts"`

**Step 3: Write minimal implementation**

Create `src/runtimes/cursor.ts` with:

```typescript
import type { ResolvedModel } from "../types.ts";
import type {
	AgentRuntime,
	HooksDef,
	OverlayContent,
	ReadyState,
	SpawnOpts,
	TranscriptSummary,
} from "./types.ts";

export class CursorRuntime implements AgentRuntime {
	readonly id = "cursor";
	readonly instructionPath = ".cursor/rules/overstory.md";

	buildSpawnCommand(_opts: SpawnOpts): string {
		throw new Error("Not implemented");
	}

	buildPrintCommand(_prompt: string, _model?: string): string[] {
		throw new Error("Not implemented");
	}

	async deployConfig(
		_worktreePath: string,
		_overlay: OverlayContent | undefined,
		_hooks: HooksDef,
	): Promise<void> {
		throw new Error("Not implemented");
	}

	detectReady(_paneContent: string): ReadyState {
		throw new Error("Not implemented");
	}

	async parseTranscript(_path: string): Promise<TranscriptSummary | null> {
		throw new Error("Not implemented");
	}

	getTranscriptDir(_projectRoot: string): string | null {
		return null;
	}

	buildEnv(_model: ResolvedModel): Record<string, string> {
		throw new Error("Not implemented");
	}
}
```

**Step 4: Run test to verify it passes**

Run: `bun test src/runtimes/cursor.test.ts`
Expected: PASS — 2 tests pass

**Step 5: Commit**

```bash
git add src/runtimes/cursor.ts src/runtimes/cursor.test.ts
git commit -m "feat(cursor): scaffold CursorRuntime with identity tests"
```

---

## Task 2: buildSpawnCommand — tests then implementation

**Files:**
- Modify: `src/runtimes/cursor.test.ts`
- Modify: `src/runtimes/cursor.ts`

**Step 1: Write the failing tests**

Add to `cursor.test.ts` inside the `CursorRuntime` describe block:

```typescript
describe("buildSpawnCommand", () => {
	test("bypass permission mode includes --yolo", () => {
		const opts: SpawnOpts = {
			model: "sonnet",
			permissionMode: "bypass",
			cwd: "/tmp/worktree",
			env: {},
		};
		const cmd = runtime.buildSpawnCommand(opts);
		expect(cmd).toBe("agent --model sonnet --yolo");
	});

	test("ask permission mode omits permission flag", () => {
		const opts: SpawnOpts = {
			model: "opus",
			permissionMode: "ask",
			cwd: "/tmp/worktree",
			env: {},
		};
		const cmd = runtime.buildSpawnCommand(opts);
		expect(cmd).toBe("agent --model opus");
		expect(cmd).not.toContain("--yolo");
		expect(cmd).not.toContain("--force");
	});

	test("appendSystemPrompt is ignored (cursor has no such flag)", () => {
		const opts: SpawnOpts = {
			model: "sonnet",
			permissionMode: "bypass",
			cwd: "/tmp/worktree",
			env: {},
			appendSystemPrompt: "You are a builder agent.",
		};
		const cmd = runtime.buildSpawnCommand(opts);
		expect(cmd).toBe("agent --model sonnet --yolo");
		expect(cmd).not.toContain("append-system-prompt");
		expect(cmd).not.toContain("You are a builder agent");
	});

	test("appendSystemPromptFile is ignored (cursor has no such flag)", () => {
		const opts: SpawnOpts = {
			model: "opus",
			permissionMode: "bypass",
			cwd: "/project",
			env: {},
			appendSystemPromptFile: "/project/.overstory/agent-defs/coordinator.md",
		};
		const cmd = runtime.buildSpawnCommand(opts);
		expect(cmd).toBe("agent --model opus --yolo");
		expect(cmd).not.toContain("cat");
		expect(cmd).not.toContain("coordinator.md");
	});

	test("cwd and env are not embedded in command string", () => {
		const opts: SpawnOpts = {
			model: "sonnet",
			permissionMode: "bypass",
			cwd: "/some/specific/path",
			env: { CURSOR_API_KEY: "test-key-123" },
		};
		const cmd = runtime.buildSpawnCommand(opts);
		expect(cmd).not.toContain("/some/specific/path");
		expect(cmd).not.toContain("test-key-123");
		expect(cmd).not.toContain("CURSOR_API_KEY");
	});

	test("all model names pass through unchanged", () => {
		for (const model of ["sonnet", "opus", "gpt-4o", "gpt-5.2", "openrouter/gpt-5"]) {
			const opts: SpawnOpts = {
				model,
				permissionMode: "bypass",
				cwd: "/tmp",
				env: {},
			};
			const cmd = runtime.buildSpawnCommand(opts);
			expect(cmd).toContain(`--model ${model}`);
		}
	});

	test("produces identical output for same inputs (deterministic)", () => {
		const opts: SpawnOpts = {
			model: "sonnet",
			permissionMode: "bypass",
			cwd: "/tmp/worktree",
			env: {},
		};
		const cmd1 = runtime.buildSpawnCommand(opts);
		const cmd2 = runtime.buildSpawnCommand(opts);
		expect(cmd1).toBe(cmd2);
	});
});
```

Also add `import type { SpawnOpts } from "./types.ts";` to imports.

**Step 2: Run test to verify it fails**

Run: `bun test src/runtimes/cursor.test.ts`
Expected: FAIL — 7 tests fail with "Not implemented"

**Step 3: Write minimal implementation**

Replace `buildSpawnCommand` in `cursor.ts`:

```typescript
buildSpawnCommand(opts: SpawnOpts): string {
	let cmd = `agent --model ${opts.model}`;

	if (opts.permissionMode === "bypass") {
		cmd += " --yolo";
	}

	return cmd;
}
```

**Step 4: Run test to verify it passes**

Run: `bun test src/runtimes/cursor.test.ts`
Expected: PASS — all buildSpawnCommand tests pass

**Step 5: Commit**

```bash
git add src/runtimes/cursor.ts src/runtimes/cursor.test.ts
git commit -m "feat(cursor): implement buildSpawnCommand with yolo bypass"
```

---

## Task 3: buildPrintCommand — tests then implementation

**Files:**
- Modify: `src/runtimes/cursor.test.ts`
- Modify: `src/runtimes/cursor.ts`

**Step 1: Write the failing tests**

Add to `cursor.test.ts`:

```typescript
describe("buildPrintCommand", () => {
	test("basic prompt produces agent -p argv with --yolo", () => {
		const argv = runtime.buildPrintCommand("Summarize this diff");
		expect(argv).toEqual(["agent", "-p", "Summarize this diff", "--yolo"]);
	});

	test("with model override appends --model flag", () => {
		const argv = runtime.buildPrintCommand("Classify this error", "haiku");
		expect(argv).toEqual([
			"agent",
			"-p",
			"Classify this error",
			"--yolo",
			"--model",
			"haiku",
		]);
	});

	test("model undefined omits --model flag", () => {
		const argv = runtime.buildPrintCommand("Hello", undefined);
		expect(argv).not.toContain("--model");
		expect(argv).toContain("--yolo");
	});

	test("--yolo always present regardless of model", () => {
		const withModel = runtime.buildPrintCommand("prompt", "opus");
		const withoutModel = runtime.buildPrintCommand("prompt");
		expect(withModel).toContain("--yolo");
		expect(withoutModel).toContain("--yolo");
	});

	test("prompt with special characters is preserved", () => {
		const prompt = 'Fix the "bug" in file\'s path & run tests';
		const argv = runtime.buildPrintCommand(prompt);
		expect(argv[2]).toBe(prompt);
	});
});
```

**Step 2: Run test to verify it fails**

Run: `bun test src/runtimes/cursor.test.ts`
Expected: FAIL — buildPrintCommand tests fail with "Not implemented"

**Step 3: Write minimal implementation**

Replace `buildPrintCommand` in `cursor.ts`:

```typescript
buildPrintCommand(prompt: string, model?: string): string[] {
	const cmd = ["agent", "-p", prompt, "--yolo"];
	if (model !== undefined) {
		cmd.push("--model", model);
	}
	return cmd;
}
```

**Step 4: Run test to verify it passes**

Run: `bun test src/runtimes/cursor.test.ts`
Expected: PASS — all buildPrintCommand tests pass

**Step 5: Commit**

```bash
git add src/runtimes/cursor.ts src/runtimes/cursor.test.ts
git commit -m "feat(cursor): implement buildPrintCommand with yolo and print flags"
```

---

## Task 4: detectReady — tests then implementation

**Files:**
- Modify: `src/runtimes/cursor.test.ts`
- Modify: `src/runtimes/cursor.ts`

**Step 1: Write the failing tests**

Add to `cursor.test.ts`:

```typescript
describe("detectReady", () => {
	test("returns loading for empty pane", () => {
		expect(runtime.detectReady("")).toEqual({ phase: "loading" });
	});

	test("returns loading for partial content (prompt only, no status bar)", () => {
		expect(runtime.detectReady("Welcome!\n\u276f")).toEqual({ phase: "loading" });
	});

	test("returns loading for partial content (status bar only, no prompt)", () => {
		expect(runtime.detectReady("shift+tab to toggle")).toEqual({ phase: "loading" });
	});

	test("returns ready for ❯ + shift+tab", () => {
		const pane = "Cursor Agent\n\u276f\nshift+tab to switch mode";
		expect(runtime.detectReady(pane)).toEqual({ phase: "ready" });
	});

	test("returns ready for ❯ + agent keyword", () => {
		const pane = "Agent mode\n\u276f\nmodel: sonnet";
		expect(runtime.detectReady(pane)).toEqual({ phase: "ready" });
	});

	test("returns ready for > at line start + shift+tab", () => {
		const pane = "Cursor\n> \nshift+tab";
		expect(runtime.detectReady(pane)).toEqual({ phase: "ready" });
	});

	test("returns ready for ❯ + esc", () => {
		const pane = "Agent\n\u276f\nesc to cancel";
		expect(runtime.detectReady(pane)).toEqual({ phase: "ready" });
	});

	test("case-insensitive match for AGENT", () => {
		const pane = "AGENT MODE\n\u276f ready";
		expect(runtime.detectReady(pane)).toEqual({ phase: "ready" });
	});

	test("returns loading for random pane content", () => {
		expect(runtime.detectReady("Loading...\nPlease wait")).toEqual({ phase: "loading" });
	});

	test("never returns dialog phase", () => {
		const panes = [
			"",
			"Loading...",
			"Agent\n\u276f\nshift+tab",
			"trust this folder",
		];
		for (const pane of panes) {
			const result = runtime.detectReady(pane);
			expect(result.phase).not.toBe("dialog");
		}
	});

	test("Shift+Tab (capital) is matched case-insensitively", () => {
		expect(runtime.detectReady("\u276f\nShift+Tab to toggle")).toEqual({
			phase: "ready",
		});
	});
});
```

**Step 2: Run test to verify it fails**

Run: `bun test src/runtimes/cursor.test.ts`
Expected: FAIL — detectReady tests fail with "Not implemented"

**Step 3: Write minimal implementation**

Replace `detectReady` in `cursor.ts`:

```typescript
detectReady(paneContent: string): ReadyState {
	const lower = paneContent.toLowerCase();

	const hasPrompt =
		paneContent.includes("\u276f") || /^> /m.test(paneContent);

	const hasStatusBar =
		lower.includes("shift+tab") ||
		lower.includes("esc") ||
		lower.includes("agent");

	if (hasPrompt && hasStatusBar) {
		return { phase: "ready" };
	}

	return { phase: "loading" };
}
```

**Step 4: Run test to verify it passes**

Run: `bun test src/runtimes/cursor.test.ts`
Expected: PASS — all detectReady tests pass

**Step 5: Commit**

```bash
git add src/runtimes/cursor.ts src/runtimes/cursor.test.ts
git commit -m "feat(cursor): implement detectReady with prompt+status bar detection"
```

---

## Task 5: deployConfig — tests then implementation

**Files:**
- Modify: `src/runtimes/cursor.test.ts`
- Modify: `src/runtimes/cursor.ts`

**Step 1: Write the failing tests**

Add imports at the top of `cursor.test.ts`:

```typescript
import { afterEach, beforeEach, describe, expect, test } from "bun:test";
import { mkdtemp } from "node:fs/promises";
import { tmpdir } from "node:os";
import { join } from "node:path";
import { cleanupTempDir } from "../test-helpers.ts";
```

Add to `cursor.test.ts` inside the `CursorRuntime` describe block:

```typescript
describe("deployConfig", () => {
	let tempDir: string;

	beforeEach(async () => {
		tempDir = await mkdtemp(join(tmpdir(), "overstory-cursor-test-"));
	});

	afterEach(async () => {
		await cleanupTempDir(tempDir);
	});

	test("writes overlay to .cursor/rules/overstory.md when provided", async () => {
		const worktreePath = join(tempDir, "worktree");

		await runtime.deployConfig(
			worktreePath,
			{ content: "# Cursor Instructions\nYou are a builder." },
			{ agentName: "test-builder", capability: "builder", worktreePath },
		);

		const overlayPath = join(worktreePath, ".cursor", "rules", "overstory.md");
		const content = await Bun.file(overlayPath).text();
		expect(content).toBe("# Cursor Instructions\nYou are a builder.");
	});

	test("creates .cursor/rules directory if it does not exist", async () => {
		const worktreePath = join(tempDir, "new-worktree");

		await runtime.deployConfig(
			worktreePath,
			{ content: "# Instructions" },
			{ agentName: "test", capability: "builder", worktreePath },
		);

		const overlayExists = await Bun.file(
			join(worktreePath, ".cursor", "rules", "overstory.md"),
		).exists();
		expect(overlayExists).toBe(true);
	});

	test("skips overlay write when overlay is undefined", async () => {
		const worktreePath = join(tempDir, "worktree");

		await runtime.deployConfig(worktreePath, undefined, {
			agentName: "coordinator",
			capability: "coordinator",
			worktreePath,
		});

		const overlayPath = join(worktreePath, ".cursor", "rules", "overstory.md");
		const overlayExists = await Bun.file(overlayPath).exists();
		expect(overlayExists).toBe(false);
	});

	test("does not write settings.local.json or guard files", async () => {
		const worktreePath = join(tempDir, "worktree");

		await runtime.deployConfig(
			worktreePath,
			{ content: "# Instructions" },
			{ agentName: "test-builder", capability: "builder", worktreePath },
		);

		const settingsPath = join(worktreePath, ".claude", "settings.local.json");
		expect(await Bun.file(settingsPath).exists()).toBe(false);

		const piGuardPath = join(worktreePath, ".pi", "extensions", "overstory-guard.ts");
		expect(await Bun.file(piGuardPath).exists()).toBe(false);
	});

	test("overwrites existing overstory.md", async () => {
		const worktreePath = tempDir;
		const rulesDir = join(worktreePath, ".cursor", "rules");
		await Bun.write(join(rulesDir, "overstory.md"), "# Old content");

		await runtime.deployConfig(
			worktreePath,
			{ content: "# New content" },
			{ agentName: "test-agent", capability: "builder", worktreePath },
		);

		const content = await Bun.file(join(rulesDir, "overstory.md")).text();
		expect(content).toBe("# New content");
	});
});
```

**Step 2: Run test to verify it fails**

Run: `bun test src/runtimes/cursor.test.ts`
Expected: FAIL — deployConfig tests fail with "Not implemented"

**Step 3: Write minimal implementation**

Add `import { mkdir } from "node:fs/promises";` and `import { dirname, join } from "node:path";` at the top of `cursor.ts`.

Replace `deployConfig` in `cursor.ts`:

```typescript
async deployConfig(
	worktreePath: string,
	overlay: OverlayContent | undefined,
	_hooks: HooksDef,
): Promise<void> {
	if (overlay) {
		const rulesPath = join(worktreePath, this.instructionPath);
		await mkdir(dirname(rulesPath), { recursive: true });
		await Bun.write(rulesPath, overlay.content);
	}
}
```

**Step 4: Run test to verify it passes**

Run: `bun test src/runtimes/cursor.test.ts`
Expected: PASS — all deployConfig tests pass

**Step 5: Commit**

```bash
git add src/runtimes/cursor.ts src/runtimes/cursor.test.ts
git commit -m "feat(cursor): implement deployConfig writing to .cursor/rules/"
```

---

## Task 6: parseTranscript — tests then implementation

**Files:**
- Modify: `src/runtimes/cursor.test.ts`
- Modify: `src/runtimes/cursor.ts`

**Step 1: Write the failing tests**

Add to `cursor.test.ts`:

```typescript
describe("parseTranscript", () => {
	let tempDir: string;

	beforeEach(async () => {
		tempDir = await mkdtemp(join(tmpdir(), "overstory-cursor-transcript-"));
	});

	afterEach(async () => {
		await cleanupTempDir(tempDir);
	});

	test("returns null for non-existent file", async () => {
		const result = await runtime.parseTranscript(join(tempDir, "does-not-exist.jsonl"));
		expect(result).toBeNull();
	});

	test("parses model from system init event", async () => {
		const path = join(tempDir, "session.jsonl");
		const initEvent = JSON.stringify({
			type: "system",
			subtype: "init",
			model: "Claude 4 Sonnet",
			session_id: "abc-123",
		});
		await Bun.write(path, `${initEvent}\n`);

		const result = await runtime.parseTranscript(path);
		expect(result).not.toBeNull();
		expect(result?.model).toBe("Claude 4 Sonnet");
		expect(result?.inputTokens).toBe(0);
		expect(result?.outputTokens).toBe(0);
	});

	test("returns zero tokens (cursor stream-json has no token usage)", async () => {
		const path = join(tempDir, "session.jsonl");
		const events = [
			JSON.stringify({ type: "system", subtype: "init", model: "gpt-5.2", session_id: "s1" }),
			JSON.stringify({ type: "user", message: { role: "user", content: [{ type: "text", text: "hello" }] } }),
			JSON.stringify({ type: "assistant", message: { role: "assistant", content: [{ type: "text", text: "hi" }] } }),
			JSON.stringify({ type: "result", subtype: "success", duration_ms: 1234, result: "hi", is_error: false }),
		].join("\n");
		await Bun.write(path, events);

		const result = await runtime.parseTranscript(path);
		expect(result).not.toBeNull();
		expect(result?.inputTokens).toBe(0);
		expect(result?.outputTokens).toBe(0);
		expect(result?.model).toBe("gpt-5.2");
	});

	test("skips malformed lines and continues parsing", async () => {
		const path = join(tempDir, "session.jsonl");
		const initEvent = JSON.stringify({ type: "system", subtype: "init", model: "sonnet" });
		await Bun.write(path, `not json at all\n${initEvent}\n{broken`);

		const result = await runtime.parseTranscript(path);
		expect(result).not.toBeNull();
		expect(result?.model).toBe("sonnet");
	});

	test("returns zero tokens and empty model for empty file", async () => {
		const path = join(tempDir, "empty.jsonl");
		await Bun.write(path, "");

		const result = await runtime.parseTranscript(path);
		expect(result).not.toBeNull();
		expect(result?.inputTokens).toBe(0);
		expect(result?.outputTokens).toBe(0);
		expect(result?.model).toBe("");
	});

	test("handles trailing newlines", async () => {
		const path = join(tempDir, "session.jsonl");
		const initEvent = JSON.stringify({ type: "system", subtype: "init", model: "opus" });
		await Bun.write(path, `${initEvent}\n\n\n`);

		const result = await runtime.parseTranscript(path);
		expect(result?.model).toBe("opus");
	});
});
```

**Step 2: Run test to verify it fails**

Run: `bun test src/runtimes/cursor.test.ts`
Expected: FAIL — parseTranscript tests fail with "Not implemented"

**Step 3: Write minimal implementation**

Replace `parseTranscript` in `cursor.ts`:

```typescript
async parseTranscript(path: string): Promise<TranscriptSummary | null> {
	const file = Bun.file(path);
	if (!(await file.exists())) {
		return null;
	}

	try {
		const text = await file.text();
		const lines = text.split("\n").filter((l) => l.trim().length > 0);

		let model = "";

		for (const line of lines) {
			let event: Record<string, unknown>;
			try {
				event = JSON.parse(line) as Record<string, unknown>;
			} catch {
				continue;
			}

			// Model from system init event.
			if (
				event.type === "system" &&
				event.subtype === "init" &&
				typeof event.model === "string"
			) {
				model = event.model;
			}
		}

		// Cursor's stream-json format does not include token usage.
		return { inputTokens: 0, outputTokens: 0, model };
	} catch {
		return null;
	}
}
```

**Step 4: Run test to verify it passes**

Run: `bun test src/runtimes/cursor.test.ts`
Expected: PASS — all parseTranscript tests pass

**Step 5: Commit**

```bash
git add src/runtimes/cursor.ts src/runtimes/cursor.test.ts
git commit -m "feat(cursor): implement parseTranscript for stream-json NDJSON"
```

---

## Task 7: buildEnv and getTranscriptDir — tests then implementation

**Files:**
- Modify: `src/runtimes/cursor.test.ts`
- Modify: `src/runtimes/cursor.ts`

**Step 1: Write the failing tests**

Add `import type { ResolvedModel } from "../types.ts";` to imports.

Add to `cursor.test.ts`:

```typescript
describe("buildEnv", () => {
	test("returns empty object when model has no env", () => {
		const model: ResolvedModel = { model: "sonnet" };
		expect(runtime.buildEnv(model)).toEqual({});
	});

	test("returns model.env when present", () => {
		const model: ResolvedModel = {
			model: "gpt-5.2",
			env: { CURSOR_API_KEY: "test-key-123" },
		};
		expect(runtime.buildEnv(model)).toEqual({ CURSOR_API_KEY: "test-key-123" });
	});

	test("returns empty object when model.env is undefined", () => {
		const model: ResolvedModel = { model: "opus", env: undefined };
		expect(runtime.buildEnv(model)).toEqual({});
	});

	test("env is safe to spread into session env", () => {
		const model: ResolvedModel = { model: "sonnet" };
		const env = runtime.buildEnv(model);
		const combined = { ...env, OVERSTORY_AGENT_NAME: "builder-1" };
		expect(combined).toEqual({ OVERSTORY_AGENT_NAME: "builder-1" });
	});
});

describe("getTranscriptDir", () => {
	test("returns null (cursor transcript location not yet known)", () => {
		expect(runtime.getTranscriptDir("/some/project")).toBeNull();
	});
});
```

**Step 2: Run test to verify it fails**

Run: `bun test src/runtimes/cursor.test.ts`
Expected: FAIL — buildEnv tests fail with "Not implemented"

**Step 3: Write minimal implementation**

Replace `buildEnv` in `cursor.ts`:

```typescript
buildEnv(model: ResolvedModel): Record<string, string> {
	return model.env ?? {};
}
```

**Step 4: Run test to verify it passes**

Run: `bun test src/runtimes/cursor.test.ts`
Expected: PASS — all buildEnv and getTranscriptDir tests pass

**Step 5: Commit**

```bash
git add src/runtimes/cursor.ts src/runtimes/cursor.test.ts
git commit -m "feat(cursor): implement buildEnv and getTranscriptDir"
```

---

## Task 8: Register CursorRuntime in registry — tests then implementation

**Files:**
- Modify: `src/runtimes/registry.ts`
- Modify: `src/runtimes/registry.test.ts`
- Modify: `src/runtimes/cursor.test.ts`

**Step 1: Write the failing tests**

Add to `cursor.test.ts` (outside the main describe block):

```typescript
describe("CursorRuntime integration: registry resolves 'cursor'", () => {
	test("getRuntime('cursor') returns CursorRuntime", async () => {
		const { getRuntime } = await import("./registry.ts");
		const rt = getRuntime("cursor");
		expect(rt).toBeInstanceOf(CursorRuntime);
		expect(rt.id).toBe("cursor");
		expect(rt.instructionPath).toBe(".cursor/rules/overstory.md");
	});
});
```

Add to `registry.test.ts`:

```typescript
import { CursorRuntime } from "./cursor.ts";

// Inside the "getRuntime" describe block:
it("returns CursorRuntime when name is 'cursor'", () => {
	const runtime = getRuntime("cursor");
	expect(runtime).toBeInstanceOf(CursorRuntime);
	expect(runtime.id).toBe("cursor");
});

it("uses config.runtime.default 'cursor' when name is omitted", () => {
	const config = { runtime: { default: "cursor" } } as OverstoryConfig;
	const runtime = getRuntime(undefined, config);
	expect(runtime).toBeInstanceOf(CursorRuntime);
	expect(runtime.id).toBe("cursor");
});

it("cursor runtime returns a new instance on each call", () => {
	const a = getRuntime("cursor");
	const b = getRuntime("cursor");
	expect(a).not.toBe(b);
});
```

**Step 2: Run tests to verify they fail**

Run: `bun test src/runtimes/registry.test.ts src/runtimes/cursor.test.ts`
Expected: FAIL — `Unknown runtime: "cursor"`

**Step 3: Write minimal implementation**

Add to `src/runtimes/registry.ts`:

1. Import: `import { CursorRuntime } from "./cursor.ts";`
2. Add map entry: `["cursor", () => new CursorRuntime()]`
3. Add to `getAllRuntimes()`: `new CursorRuntime()`

**Step 4: Run tests to verify they pass**

Run: `bun test src/runtimes/registry.test.ts src/runtimes/cursor.test.ts`
Expected: PASS — all registry tests pass

Note: The "throws with a helpful message for an unknown runtime" test in `registry.test.ts` will need its expected message updated to include "cursor" in the available list:
`'Unknown runtime: "unknown-runtime". Available: claude, codex, pi, copilot, gemini, sapling, opencode, cursor'`

**Step 5: Commit**

```bash
git add src/runtimes/registry.ts src/runtimes/registry.test.ts src/runtimes/cursor.test.ts
git commit -m "feat(cursor): register CursorRuntime in runtime registry"
```

---

## Task 9: Quality gates — typecheck, lint, full test suite

**Files:**
- None modified — verification only

**Step 1: Run typecheck**

Run: `tsc --noEmit`
Expected: PASS — no type errors

**Step 2: Run linter**

Run: `biome check .`
Expected: PASS — no lint errors (fix any if they appear)

**Step 3: Run full test suite**

Run: `bun test`
Expected: PASS — all tests pass, including existing ones

**Step 4: Commit if any fixes were needed**

```bash
git add -A
git commit -m "fix(cursor): address lint/type issues from quality gates"
```

---

## Task 10: Update runtime-adapters.md documentation

**Files:**
- Modify: `docs/runtime-adapters.md`

**Step 1: Add Cursor to the architecture overview diagram**

Add `+--- CursorRuntime  (src/runtimes/cursor.ts)` with `agent --model ... --yolo ...` to the ASCII diagram in Section 1.

**Step 2: Add Cursor to the instructionPath table**

Add row: `| Cursor | .cursor/rules/overstory.md |`

**Step 3: Add a Cursor subsection to Section 3 (Existing Adapters)**

```markdown
### Cursor (`src/runtimes/cursor.ts`)

A TUI runtime for the Cursor CLI (`agent` binary).

**Key characteristics:**
- `id = "cursor"`, `instructionPath = ".cursor/rules/overstory.md"`
- Spawn command: `agent --model <model> [--yolo]`
- `permissionMode: "bypass"` maps to `--yolo`; `"ask"` adds no flag
- `appendSystemPrompt` and `appendSystemPromptFile` are silently ignored —
  the `agent` CLI has no equivalent flag
- No hooks deployment — Cursor has a permission system (`cli.json`) but no
  per-tool interception hooks; `_hooks` param unused
- TUI readiness: prompt indicator (`❯` or `>`) AND status bar
  (`shift+tab`, `esc`, or `agent`) → ready; no trust dialog phase
- Transcript parsing extracts model from `system/init` NDJSON events;
  token usage is not available in Cursor's stream-json format (returns zeros)
- Beacon verification uses the default (omitted → gets resend loop)
```

**Step 4: Add Cursor examples to relevant tables**

Update the buildSpawnCommand examples, buildPrintCommand examples, deployConfig table, detectReady behavior table, requiresBeaconVerification table, and buildEnv section where other runtimes are listed.

**Step 5: Commit**

```bash
git add docs/runtime-adapters.md
git commit -m "docs: add Cursor runtime adapter to runtime-adapters.md"
```

---

## Summary

| Task | What | Test file | Source file |
|------|------|-----------|-------------|
| 1 | Identity (id, instructionPath) | `cursor.test.ts` | `cursor.ts` (scaffold) |
| 2 | `buildSpawnCommand` | `cursor.test.ts` | `cursor.ts` |
| 3 | `buildPrintCommand` | `cursor.test.ts` | `cursor.ts` |
| 4 | `detectReady` | `cursor.test.ts` | `cursor.ts` |
| 5 | `deployConfig` | `cursor.test.ts` | `cursor.ts` |
| 6 | `parseTranscript` | `cursor.test.ts` | `cursor.ts` |
| 7 | `buildEnv` + `getTranscriptDir` | `cursor.test.ts` | `cursor.ts` |
| 8 | Registry registration | `registry.test.ts` + `cursor.test.ts` | `registry.ts` |
| 9 | Quality gates | — | — |
| 10 | Documentation | — | `runtime-adapters.md` |

Total: ~10 tasks, ~40 steps. Each task follows strict red-green-commit TDD.
