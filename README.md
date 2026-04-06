# shared-Claude-Todo

A shared task queue for [Claude Todo](https://github.com/Ayush-Puri/Claude-Todo). Push task files here and they'll be automatically picked up and executed by Claude Code on the host machine.

## How It Works

```
You push a task file → GitHub repo → Host pulls every 2 min → Claude Todo picks it up → Executes in Claude Code → Results logged
```

1. **You** create a JSON task file and push it (or open a PR)
2. The **host machine** pulls new files every 2 minutes
3. **Claude Todo** detects new tasks and queues them
4. Tasks from the **same batch file** run in a **single Claude Code session** (shared context)
5. **Max 3 sessions** run concurrently — new batches wait until all 3 finish
6. Results are logged to Obsidian + Google Drive, and task status is pushed back to this repo

## Task File Format

Place JSON files in the `queue/` directory. One file = one batch = one Claude Code session.

### File naming
```
queue/{priority}_{description}.json
```
- Priority: `01` (highest) to `99` (lowest)
- Example: `queue/01_fix-auth-bug.json`, `queue/50_write-tests.json`

### JSON schema
```json
{
  "batch": "fix-auth-bug",
  "description": "Fix the authentication token refresh bug in the API",
  "author": "github-username",
  "priority": 1,
  "model": "haiku",
  "project": "~/projects/my-api",
  "tasks": [
    {
      "title": "Diagnose the auth refresh issue",
      "prompt": "Go to ~/projects/my-api. Read src/auth/refresh.js. Identify why tokens aren't refreshing after expiry. Check the error logs in logs/auth.log.",
      "verification": "Run: grep -c 'refresh' src/auth/refresh.js",
      "expectedResult": "File exists and contains refresh logic"
    },
    {
      "title": "Fix the token refresh",
      "prompt": "Fix the token refresh bug identified in the previous step. Ensure tokens refresh 5 minutes before expiry.",
      "verification": "Run: cd ~/projects/my-api && npm test -- --grep 'refresh'",
      "expectedResult": "All refresh-related tests pass"
    },
    {
      "title": "Create PR with the fix",
      "prompt": "Create a branch 'fix/token-refresh', commit the changes, push, and create a PR with a description of what was fixed.",
      "verification": "Run: gh pr list --repo owner/my-api --state open --json title",
      "expectedResult": "PR exists with token refresh fix"
    }
  ]
}
```

### Fields

| Field | Required | Description |
|-------|----------|-------------|
| `batch` | Yes | Unique name for this batch (used as session identifier) |
| `description` | Yes | Human-readable description of the batch |
| `author` | Yes | GitHub username of the person who created the batch |
| `priority` | No | 1-99, lower = higher priority (default: 50) |
| `model` | No | Claude model: `haiku`, `sonnet`, `opus` (default: `haiku`) |
| `project` | No | Working directory for all tasks in this batch |
| `tasks` | Yes | Array of tasks (executed sequentially in one session) |
| `tasks[].title` | Yes | Short title for the task |
| `tasks[].prompt` | Yes | Full instructions for Claude |
| `tasks[].verification` | No | Command to verify completion |
| `tasks[].expectedResult` | No | What success looks like |

## Execution Rules

1. **One batch = one session**: All tasks in a batch file share the same Claude Code session (full context preserved between tasks)
2. **Max 3 concurrent sessions**: If 3 batches are running, new ones wait in queue
3. **Priority ordering**: Lower priority number runs first (`01` before `50`)
4. **Completion**: After a batch completes, its file moves from `queue/` to `done/` (or `failed/`)
5. **Status updates**: The host pushes status back to this repo every sync cycle

## Directory Structure

```
shared-Claude-Todo/
├── queue/           ← Drop task files here (pending execution)
│   ├── 01_urgent-fix.json
│   └── 50_write-docs.json
├── running/         ← Currently executing (moved here by dispatcher)
├── done/            ← Completed successfully (moved here after execution)
├── failed/          ← Failed after retry (moved here, check iterations for trace)
├── responses/       ← Packaged output from completed tasks
│   ├── 01_urgent-fix-output.md
│   └── 50_write-docs-output.md
└── README.md
```

### Response Files

After each batch completes, the dispatcher packages all Claude outputs into a markdown file at `responses/{batch-name}-output.md`. This includes:

- **Full text output** from each task
- **Tools used** (files read/written, commands run, web searches)
- **Files created** (paths of any files Claude wrote)
- **Errors** (if any)

Response files accumulate — they are never deleted. This gives you a complete history of every execution's output, reviewable directly in GitHub.

## Quick Start

### For collaborators
```bash
# Clone the repo
git clone https://github.com/Ayush-Puri/shared-Claude-Todo.git
cd shared-Claude-Todo

# Create a task
cat > queue/50_my-task.json << 'EOF'
{
  "batch": "my-first-task",
  "description": "Test the shared queue",
  "author": "your-github-username",
  "priority": 50,
  "model": "haiku",
  "tasks": [
    {
      "title": "Say hello",
      "prompt": "Print 'Hello from the shared queue!' and write it to ~/claude-auto/logs/shared-test.md",
      "verification": "Check that ~/claude-auto/logs/shared-test.md exists",
      "expectedResult": "File exists with hello message"
    }
  ]
}
EOF

# Push it
git add . && git commit -m "Add test task" && git push
```

### For the host (already set up if Claude Todo is installed)
The sync daemon pulls every 2 minutes automatically. No action needed.

## Security

- **No direct machine access** — collaborators only push JSON files to GitHub
- **PR review** — enable branch protection to require PR approval before merge
- **Audit trail** — full git history of who added what tasks and when
- **Revoke access** — remove GitHub collaborator to cut off access
- **Sandboxed execution** — tasks run via Claude Code with `--dangerously-skip-permissions` (host's responsibility to trust collaborators)
- **Rate limited** — max 3 concurrent sessions prevents resource exhaustion

## Recommended: Enable Branch Protection

For production use, enable branch protection on `main`:
1. Go to Settings → Branches → Add rule for `main`
2. Require pull request reviews (1 reviewer)
3. This ensures all tasks are reviewed before execution

## License

MIT
