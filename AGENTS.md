# AGENTS.md

## Trunc-Based Agent Development Workflow

This document describes the required conventions for agents (of any kind: human or automated) contributing to this repository.
Agents must strictly follow this workflow to ensure maximal parallelism, clarity, and project hygiene.

### 1. Selecting Work
- Agents should select any open TODO item anywhere in the repo (README, code, docs, issues, etc).
- Evaluate if the TODO item is sufficiently granular. If the task is too broad, break it into smaller TODOs, and select one atomic item to work on.

### 2. Mark the Work as In-Progress
- Immediately replace the chosen TODO item with

   * `IN-PROGRESS (YYYY-MM-DDTHH:MM:SSZ): <original text>`
   * using the current UTC date and time (ISO 8601 format, 'Z' for UTC).
     +- Commit and push this change right away. This signals to all agents which TODOs are being taken, minimizing risk of parallel duplication.
   *

### 3. Perform the Work
- Implement the task as described.
- If the area of work turns out to be too broad, add new TODO items for any subproblems discovered (do not mark them as in-progress unless you are actively working on them now).
- If you notice issues outside your focus area, add TODO items where appropriate. Always document the reasoning.

### 4. Complete or Update the TODOs
- When a task is finished, remove the now-completed IN-PROGRESS/TODO line(s).
- If you broke the work into parts, remove only what you've finished.
- Commit and push all changes, ensuring your working directory is clean (no uncommitted files left).

### 5. Handling Merge Conflicts
- On push/pull conflict:

   * * Investigate the file's commit history from the merge base.
   * * If the conflict is trivial, resolve it directly and **add a code comment explaining your resolution** at the merge site.
   * * If the conflict is due to overlapping/uncoordinated TODO work, either:
   * * Add more refined TODOs for what remains.
   * * Or, if your work is now obsolete, explain why in a comment and abandon/remove incomplete changes.

- Always commit and push after a merge conflict, ensuring all resolutions are justified in comments.

### 6. Principles
- Never leave uncommitted changes behind.
- All your actions must be transparent: visible in git history, via TODO/IN-PROGRESS lines, and clarified with commit messages or code comments as required.
- If your work requires meta-process improvements, add a TODO describing specifically what needs clarifying or refining in this workflow.

### Example
1. Find:

   * * `- [ ] Implement websocket server`
           +2. Replace with:
   * * `- [ ] IN-PROGRESS (2025-06-19T08:28:57Z): Implement websocket server`
           +3. Commit message:
   * * `mark websocket server task as IN-PROGRESS (2025-06-19T08:28:57Z)`
           +4. If you discover you need a new validator, add:
   * * `- [ ] TODO: Write validator for websocket packet payload`
           +5. After work is done, remove/chop up finished IN-PROGRESS/TODO items. Commit and push.
   *

### Agent Loop

If you are an automated agent:
On every "work" command, read this file and execute these instructions step by step. At end, confirm there are zero unstaged changes.