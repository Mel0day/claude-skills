Set up iCloud sync and MCP Memory Server for cross-machine and cross-session Claude Code sync.

## Steps

Run the following setup in order. For each step, check the current state before making changes.

### 1. Detect username and iCloud path

Determine the current user's home directory and verify iCloud Drive is available at `~/Library/Mobile Documents/com~apple~CloudDocs/`.

If iCloud is not available, stop and tell the user to enable iCloud Drive first.

### 2. Create iCloud sync directory

Create the directory `~/Library/Mobile Documents/com~apple~CloudDocs/claude-sync/` if it doesn't exist.

### 3. Sync projects (Auto Memory)

Check if `~/.claude/projects` is already a symlink pointing to the iCloud sync directory.

- If NOT a symlink: copy `~/.claude/projects` to iCloud (`~/Library/Mobile Documents/com~apple~CloudDocs/claude-sync/projects`), then replace with a symlink.
- If already a symlink to the correct location: skip, report already configured.

### 4. Sync CLAUDE.md

Check if `~/.claude/CLAUDE.md` is already a symlink pointing to the iCloud sync directory.

- If NOT a symlink and file exists locally: copy it to iCloud first, then replace with symlink.
- If NOT a symlink and file doesn't exist: create a default CLAUDE.md in iCloud with the user's basic info, then symlink.
- If already a symlink to the correct location: skip, report already configured.

### 5. Configure MCP Memory Server

Read `~/.claude.json` and check if a `memory` entry already exists in `mcpServers`.

- If NOT present: add the following to `mcpServers` in `~/.claude.json`:

```json
"memory": {
  "command": "npx",
  "args": [
    "-y",
    "@modelcontextprotocol/server-memory",
    "--storage-path",
    "<HOME>/Library/Mobile Documents/com~apple~CloudDocs/claude-sync/mcp-memory.json"
  ],
  "env": {},
  "type": "stdio"
}
```

Replace `<HOME>` with the actual home directory path.

- If already present: skip, report already configured.

### 6. Report results

Print a summary of what was done and what was skipped. Remind the user that on other machines they only need to run `/sync-setup` to apply the same configuration.
