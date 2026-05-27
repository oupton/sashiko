# Design: `git_grep` Output Formatting, Proximity Sorting, and Count-Only Optimizations

This document outlines the design and implementation plan for improving the tokenomics and search effectiveness of `git_grep`.

## Overview
`git_grep` is the most frequently called tool in `sashiko`, accounting for ~70% of all tool output volume. Currently, its output is highly redundant and sorted alphabetically by filename, which causes severe token waste and causes matches in late-alphabet subsystems (like `io_uring/`, `fs/`, `net/`) to be discarded under truncation.

To address these issues, we propose three highly effective optimizations:
1. **Redundant Prefix Strip & Heading Grouping**: Format matches grouped under a single file header, stripping the repeated 40-char revision hash and filename prefix from every line.
2. **Semantic Proximity Sorting**: Sort matches by proximity to files modified by the patch under review so that relevant context (caller/definition sites) is never truncated.
3. **Count-Only Mode**: Add a `count_only` boolean parameter to return only the file paths and match counts, allowing ultra-cheap broad searches across the entire repository with zero risk of missing any matches.

---

## 1. Output Formatting & Prefix Stripping

### Current Format
Every single matched line and context line from `git grep` contains the revision and filename repeated:
```
<revision>:<filename>:<line_num>:<content>
<revision>:<filename>-<line_num>-<content>
```
*Example*:
```
fcbc51e54d2aa9d402206601f4894251049e5d77:drivers/gpu/drm/tegra/output.c-167-\tin (ddc) {
fcbc51e54d2aa9d402206601f4894251049e5d77:drivers/gpu/drm/tegra/output.c:168:\t\toutput->ddc = of_find_i2c_adapter_by_node(ddc);
```

### Optimized Format
Parse the output lines in Rust and reconstruct them grouped by file:
```
[file: <filename>]
  167-  if (ddc) {
  168:    output->ddc = of_find_i2c_adapter_by_node(ddc);
```

### Implementation in `src/worker/tools.rs`
We will add a parsing/reformatting helper function:
```rust
fn format_git_grep_output(stdout: &str, revision: &str) -> String {
    // Parse lines matching "<revision>:<path>[:|-]<line>[:|-]<content>"
    // Strip <revision>:
    // Group by <path> and format compactly.
}
```

---

## 2. Semantic Proximity Sorting

### Goal
Ensure that if a search returns hundreds of matches, the matches in files or directories modified by the patch are sorted to the very top of the output so they are never cut off by the 10,000-token truncator.

### Passing Active Patch Files to `ToolBox`
We will add an optional `active_patch_files` field to `ToolBox`:
```rust
pub struct ToolBox {
    worktree_path: PathBuf,
    prompts_path: Option<PathBuf>,
    active_patch_files: Vec<String>,
}
```
And in `src/bin/review.rs`, we will populate this list using the filenames from the patches of the patchset being reviewed:
```rust
let mut patch_files = Vec::new();
for p in &patches_to_review {
    // Gather modified files (available from patch diffs/structures)
}
let mut tools = ToolBox::new(worktree.path.clone(), prompts_tool_path);
tools.set_active_patch_files(patch_files);
```

### Proximity Ranking Algorithm
When `git_grep` executes and gets reformatted matches grouped by file:
1. Match each file's path against the `active_patch_files`.
2. Assign a priority score to each file:
   * **Score 1 (Highest)**: The file path is exactly in `active_patch_files`.
   * **Score 2**: The file is in the same directory as one of the `active_patch_files` (shares directory prefix).
   * **Score 3**: The file path starts with `include/` (headers).
   * **Score 4 (Lowest)**: All other files.
3. Sort the grouped file blocks by their priority score.
4. Render the sorted blocks to the final string and truncate.

---

## 3. Count-Only Mode

### Goal
Allow the AI to perform broad searches without retrieving verbose line contents, reducing token consumption for initial discovery to near zero.

### Implementation in `src/worker/tools.rs`
Add a `count_only` parameter to `git_grep`. If true, execute `git grep` with the `-c` (or `--count`) flag.

---

## 4. Step-by-Step Implementation Plan

*   [ ] **Step 1**: Implement `format_git_grep_output` logic in `src/worker/tools.rs` and add comprehensive unit tests verifying the prefix stripping and grouping.
*   [ ] **Step 2**: Implement the `count_only` mode in `git_grep` tool execution and add a unit test verifying that it returns only file paths and counts.
*   [ ] **Step 3**: Add `active_patch_files` field and setter to `ToolBox`, and implement the proximity ranking algorithm to sort matches before truncation. Add unit tests verifying the priority sorting.
*   [ ] **Step 4**: Integrate and populate `active_patch_files` from `review.rs` during the active review process.
*   [ ] **Step 5**: Execute `make check-pr` to verify all tests pass.
