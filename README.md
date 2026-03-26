# Airspace Claude Plugins

Claude Code plugins for Airspace development workflows.

## Plugins

### airspace-tools

JIRA-aware code review that grounds feedback in ticket requirements and acceptance criteria.

**Command:** `/airspace-tools:airspace-code-review`

**Features:**
- Fetches JIRA ticket from PR body, title, or branch name (AT/RR/INT/VAN prefixes)
- Reviews against JIRA acceptance criteria, CLAUDE.md rules, and code quality
- Re-review support: detects prior reviews and tracks which issues were fixed
- Works on PRs or local branches (no PR required)

**Usage:**
```
/airspace-tools:airspace-code-review                              # review current branch PR, terminal output
/airspace-tools:airspace-code-review --pr=123                     # specific PR
/airspace-tools:airspace-code-review --markdown                   # write to review-PR-<number>.md
/airspace-tools:airspace-code-review --comment                    # post inline comments on GitHub PR
/airspace-tools:airspace-code-review Focus on the SQL migration   # add context for reviewers
```

**Prerequisites:**
- `gh` CLI installed and authenticated
- Atlassian MCP server configured (for JIRA integration)

## Installation

### 1. Register the marketplace

Add the following entry to `~/.claude/plugins/known_marketplaces.json` inside the top-level object:

```json
"airspace-plugins": {
  "source": {
    "source": "github",
    "repo": "jeremyberglund-airspace/airspace-claude-plugins"
  },
  "installLocation": "<HOME_DIR>/.claude/plugins/marketplaces/airspace-plugins",
  "lastUpdated": "2026-01-01T00:00:00.000Z"
}
```

Replace `<HOME_DIR>` with your actual home directory path (e.g. `/Users/yourname`).

If the file doesn't exist yet, create it:
```json
{
  "airspace-plugins": {
    "source": {
      "source": "github",
      "repo": "jeremyberglund-airspace/airspace-claude-plugins"
    },
    "installLocation": "<HOME_DIR>/.claude/plugins/marketplaces/airspace-plugins",
    "lastUpdated": "2026-01-01T00:00:00.000Z"
  }
}
```

### 2. Install the plugin

Open Claude Code and run:

```
/plugin install airspace-tools@airspace-plugins
```

### 3. Verify

Restart Claude Code. The command `/airspace-tools:airspace-code-review` should appear in the slash command list.

## Updating

After changes are pushed to this repo:

1. Sync your local marketplace clone (required due to a [Claude Code bug](https://github.com/anthropics/claude-code/issues/10182) — it doesn't git pull before checking for updates):
   ```bash
   cd ~/.claude/plugins/marketplaces/airspace-plugins && git pull origin main
   ```
2. Open Claude Code and run `/plugin`, then select "Update Now" for airspace-plugins

## Development

### Local development workflow

Clone the repo:
```bash
git clone git@github.com:jeremyberglund-airspace/airspace-claude-plugins.git ~/Code/airspace-claude-plugins
```

Test changes without reinstalling by using the `--plugin-dir` flag:
```bash
claude --plugin-dir ~/Code/airspace-claude-plugins/plugins/airspace-tools
```

Once changes are ready, commit, push, and update via `/plugin`.

### Adding a new plugin

1. Create a directory under `plugins/`:
   ```
   plugins/my-new-plugin/
   ├── .claude-plugin/
   │   └── plugin.json
   └── commands/
       └── my-command.md
   ```

2. Add a `plugin.json`:
   ```json
   {
     "name": "my-new-plugin",
     "description": "What it does",
     "version": "1.0.0"
   }
   ```

3. Register it in `.claude-plugin/marketplace.json`:
   ```json
   {
     "name": "my-new-plugin",
     "description": "What it does",
     "source": "./plugins/my-new-plugin",
     "category": "productivity"
   }
   ```

4. Push and install: `/plugin install my-new-plugin@airspace-plugins`

### Repo structure

```
.claude-plugin/
└── marketplace.json              # marketplace manifest — register plugins here
plugins/
└── airspace-tools/               # each plugin gets its own directory
    ├── .claude-plugin/
    │   └── plugin.json
    └── commands/
        └── airspace-code-review.md
```
