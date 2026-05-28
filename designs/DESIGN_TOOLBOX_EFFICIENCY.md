# Design: Sashiko Toolbox Efficiency & Standardized Responses

## Overview
This document details the design for optimizing the sashiko LLM toolbox. It addresses two primary goals:
1.  **Standardized and Robust Truncation Responses:** Establishing a consistent, structured JSON envelope for all content-returning tools to make truncation explicitly visible and pagination easily actionable for the LLM.
2.  **Parallel Tool Execution:** Upgrading the worker loop in `src/worker/prompts.rs` to execute independent batched tool calls concurrently, dramatically reducing review stage latencies.

---

## 1. Standardized Tool Response Envelope

To ensure that the LLM can use a single, consistent set of reasoning rules for handling pagination, truncation, and paging across *all* tools, we will implement a standardized response envelope.

### Standard Schema Definition (JSON)
Every text or list-returning tool will conform to the following structure:

```json
{
  "content": "[The main output text, list, or serialized data]",
  "truncated": true,
  "metadata": {
    "total_items": 1850,      // Total lines in file, total matches, total commits, or total files
    "returned_items": 800,     // How many items are returned in "content"
    "start_index": 1,         // 1-based start line/index
    "end_index": 800          // 1-based end line/index
  },
  "next_page_hint": "Only lines 1-800 are shown due to token limits. To read the remaining lines, call this tool again with start_line=801.",
  
  // Legacy Backward-Compatibility Fallbacks
  "output": "[diff/log text...]",  // Kept for git_log backwards compatibility
  "files": "[file list...]"         // Kept for git_find_files backwards compatibility
}
```

### Refactoring Mapping for Tools

We will refactor the tools inside `src/worker/tools.rs` to return standard responses:

1.  **`git_read_files` / `read_single_file`**:
    *   Add `"truncated"` and `"metadata"` to each file result.
    *   Populate `next_page_hint` if the file was truncated inside `read_single_file` or raw mode.
2.  **`git_blame`**:
    *   Wrap blame output into the standard envelope with proper metadata.
3.  **`git_diff`**:
    *   Wrap diff output. Set `"metadata"` containing total lines of diff vs returned lines.
4.  **`git_show`**:
    *   Update both commit and file-show logic to return the standard envelope.
5.  **`git_log`**:
    *   Change primary field to `"content"`.
    *   Populate legacy `"output"` with the same value to maintain backward compatibility.
    *   Set `metadata` with `limit` and `total_items` if determinable.
6.  **`git_find_files`**:
    *   Change primary field to `"content"`.
    *   Populate legacy `"files"` with the same value.
    *   Add `metadata` with `total_items` (total found).

---

## 2. Parallel Tool Execution

Sashiko's prompt system instructs the LLM to batch independent tool calls into a single response to save round-trips. Currently, the worker loop in `src/worker/prompts.rs` executes these batched calls sequentially using an awaited `for` loop:

```rust
// CURRENT SEQUENTIAL FLOW (Slow)
for call in tool_calls {
    let result = self.tools.call(&name, args).await?;
    ...
}
```

Since all tool calls inside `ToolBox` are read-only and take `&self`, we can execute them concurrently without race conditions.

### Concurrency Design
We will refactor the tool loop in `run_ai_stage_raw` to:
1.  **Synchronously Check Duplicates & Log History:** Before spawning futures, synchronously perform duplicate checks on `self.action_history` to prevent multiple duplicate calls in the same batch.
2.  **Spawn Concurrent Futures:** Package each tool call into an async future.
3.  **Await All Concurrently:** Use `futures::future::join_all` to run all tool calls concurrently in parallel tokio tasks.
4.  **Collect and Order Responses:** Reassemble the results into standard `AiMessage` structures in their original call order to preserve the conversation flow.

```
                [ LLM Batched Tool Calls ]
                            |
            +---------------+---------------+
            |               |               |
       [Tool Call 1]   [Tool Call 2]   [Tool Call 3]
            |               |               |
            +---------------+---------------+  <-- Executed Concurrently in Tokio
                            |
                  [ Collect Results ]
                            |
                  [ Next LLM Conversation Turn ]
```

---

## 3. Regression Risks & Verification Plan

### Regression Risks
1.  **Broken Unit Tests:** Test assertions in `src/worker/tools_test.rs` and `src/worker/tools.rs` that rely on the exact pathing structure (like `result["output"]` or `result["start_line"]` at root) will break.
    *   *Mitigation:* We will modify both test suites synchronously to align assertions with the standardized JSON fields.
2.  **AI Model Confusion:** If prompt instructions explicitly reference legacy fields like `files` or `output`, the LLM might fail to parse the results.
    *   *Mitigation:* We will implement **backward-compatibility fallbacks** so that tools return both legacy keys (e.g., `"output"`, `"files"`) and the new `"content"` key during the transition period.

### Verification Plan
1.  **Lint Check:** Run `make lint` to ensure clean formatting and clippy alignment.
2.  **Unit Tests:** Run `make test` to ensure all updated tests pass successfully.
3.  **Integration Tests:** Run `make integration-test` to verify full sashiko review cycle works end-to-end with the new concurrent execution and standardized envelopes.
