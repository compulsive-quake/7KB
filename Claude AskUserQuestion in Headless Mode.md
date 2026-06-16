# Claude AskUserQuestion in Headless Mode

Learned while fixing ModForge's "TOOL ERROR — Answer questions?" in Claude mode (2026-06-15).

## The trap

ModForge runs Claude headless: `claude --print --input-format stream-json --output-format stream-json` (see `claude_invocation` in `src-tauri/src/claude_stream.rs`). In that mode there is **no built-in UI for `AskUserQuestion`**, so the CLI handles the tool itself:

1. Emits a normal `assistant` → `tool_use` block for `AskUserQuestion` with the questions/options in `input` (`{ questions: [{ question, header, multiSelect, options:[{label,description}] }] }`, plus a `caller` field).
2. **Immediately auto-fails it** by emitting a `user` → `tool_result` with `{ "content": "Answer questions?", "is_error": true }` (and `tool_use_result: "Error: Answer questions?"`).
3. Claude then treats the question as dismissed, ends the turn, and a `system/post_turn_summary` line reports `status_category:"blocked"` / `needs_action:"…"`.

Crucially, the CLI does **not** use the control protocol for this — it never sends a `control_request` awaiting a `control_response`. So you cannot "answer the original tool call"; the turn already moved past it.

## How ModForge handles it

Mirror the existing permission-denied flow (don't invent a new transport):

- Detect the `AskUserQuestion` `tool_use` block in the UI and render an answer widget (`QuestionPrompt` in `src/components/StreamView.tsx`). Only the most recent AskUserQuestion stays interactive (`lastQuestionToolUseId`).
- Hide the `"Answer questions?"` error `tool_result` — it's pure headless noise (`isAskQuestionErrorResult`).
- On submit, send the picks back as an ordinary follow-up user message via `claudeStreamSend` (`onQuestionAnswer` in `src/components/LeafPaneView.tsx`). The turn was left `blocked` awaiting input, so a plain message resumes it.

This is the same shape as the `PermissionPrompt` / `onPermissionDecision` path — a true control-protocol responder is unnecessary because the native `--print` CLI doesn't delegate either questions or permissions over stdin; it auto-resolves and ModForge reacts to the resulting block.

## Related

- [[Claude CLI Session Resume Gotchas]] — other stream-json mode quirks in the same files.
