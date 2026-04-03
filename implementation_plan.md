# Analysis and Plan to Reduce Token Overhead

The user reported that simply saying "hi" results in 18,000 input tokens. This overhead significantly increases latency and cost for routine interactions.

## Analysis of the 18k Token Overhead

Based on the codebase analysis, the token count is composed of:

1.  **System Prompt (~8,000 tokens)**: `src/agents/system-prompt.ts` generates a ~33KB string containing safety rules, tool style guides, CLI references, and workspace instructions.
2.  **Tool Definitions (~8,000 - 10,000 tokens)**: OpenClaw registers ~30 core tools. Each tool comes with a JSON schema for parameters. Providers like Anthropic and OpenAI include these schemas in the input token count.
3.  **Injected Context**: `soul.md` and other workspace files are automatically added to the prompt, which can grow based on workspace content.

## Proposed Improvements

### 1. Modular System Prompt (Immediate Impact)
Introduce a `minimal` mode or a set of toggles for `buildAgentSystemPrompt` to exclude sections like "Safety", "OpenClaw CLI Quick Reference", and "Tool Call Style" for routine messages.

### 2. Intelligent Tool Pruning
Implement a "Level 1" toolset for initial greetings and "Level 2" for task execution.
- **Level 1 (Core)**: `read`, `ls`, `session_status`, `message`.
- **Level 2 (Full)**: All tools (loaded only when the agent decides it needs them or after the first turn).

### 3. Prompt Caching Integration
Leverage provider-specific caching (Anthropic's Prompt Caching or Gemini's Context Caching) to ensure the 18k base is only charged once or at a fraction of the cost.

### 4. Schema Optimization
Create a "minified" version of tool schemas for the LLM that removes redundant `description` fields or simplifies complex `anyOf` types while maintaining functional correctness.

## User Review Required

> [!IMPORTANT]
> Reducing the system prompt or tool definitions can sometimes lead to decreased agent performance or "forgetfulness" regarding certain capabilities. I recommend starting with **Modular System Prompt** and **Tool Pruning** for the first turn.

## Proposed Changes

### [agents]

#### [MODIFY] [system-prompt.ts](file:///d:/Work/Source/openclaw/src/agents/system-prompt.ts)
- Add more granular toggles for sections.
- Optimize the "CLI Quick Reference" to be more concise.

#### [MODIFY] [pi-tools.ts](file:///d:/Work/Source/openclaw/src/agents/pi-tools.ts)
- Implement a `pruneToolsForFirstTurn` helper that filters out complex tools (like `browser`, `canvas`, `apply_patch`) when the conversation history is empty or the user intent is simple.

#### [MODIFY] [pi-embedded-runner/payloads.ts](file:///d:/Work/Source/openclaw/src/agents/pi-embedded-runner/run/payloads.ts)
- Detect "first turn" and apply optimizations.

## Open Questions

1. **Should pruning be automatic?** Alternatively, should we provide a config option like `tools.autoPrune` (default=true)?
2. **First-turn vs. All-turns**: Should we only prune on the very first message ("hi"), or maintain a smaller toolset until a task is explicitly started?

## Verification Plan

### Automated Tests
- `pnpm test src/agents/system-prompt.test.ts`: Verify minimal modes work.
- `pnpm test src/agents/pi-tools.test.ts`: Verify tool filtering logic.

### Manual Verification
- Run `openclaw gateway run` and send "hi".
- Observe the token usage in the logs (should drop significantly from 18k).
