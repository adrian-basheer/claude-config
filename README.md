# Claude Config

Personal Claude Code configuration — global commands, skills, scaffold templates, and reusable patterns.

## Structure

```
claude-config/
├── commands/                              # Global slash commands
│   └── security-check.md                  # /security-check — OWASP Top 10 security audit
├── skills/                                # Global skills (richer than commands)
│   └── new-project/                       # /new-project — full project creation workflow
│       ├── SKILL.md                       # Orchestrator: scaffold → azure setup → connect
│       ├── SCAFFOLD-SPEC.md               # Full spec for code generation
│       ├── project-config.template.json   # Config template with all fields
│       ├── project-config.mixshare.json   # Filled-in example (Mixshare project)
│       └── runsheets/                     # Step-by-step Azure resource setup guides
│           ├── 00-overview.md
│           ├── 01-entra-external-id-setup.md
│           ├── 02-app-registrations.md
│           ├── 03-storage-account.md
│           ├── 04-key-vault.md
│           ├── 05-app-services.md
│           ├── 06-microsoft-graph-api.md
│           ├── 07-azure-devops-pipeline.md
│           ├── 08-api-management.md
│           └── 09-cosmos-db.md
└── README.md
```

## Workflow

The `/new-project` skill guides you through three phases:

```
Phase 1: Scaffold    →  Buildable code with TODO_* placeholders
                        Includes runsheets/ folder for reference

Phase 2: Azure Setup →  Interactive guide through Azure Portal + az CLI
                        Fills in project-config.json with real values

Phase 3: Connect     →  Replaces all placeholders with real values
                        Project is ready to deploy
```

You can stop after any phase and resume later — progress is saved in `project-config.json`.

## Setup (new machine)

### 1. Clone this repo

```bash
git clone https://github.com/adrian-basheer/claude-config.git
```

### 2. Create junctions

Two junctions link the repo's folders to `~/.claude/` so Claude Code picks them up globally.

**Windows (PowerShell — no admin required):**

```powershell
# Adjust $RepoPath to where you cloned the repo
$RepoPath = "C:\Users\YourUser\source\repos\claude-config"

# Remove existing dirs if present
if (Test-Path "$HOME\.claude\commands") { Remove-Item "$HOME\.claude\commands" -Recurse }
if (Test-Path "$HOME\.claude\skills")   { Remove-Item "$HOME\.claude\skills" -Recurse }

# Create junctions
cmd /c mklink /J "$HOME\.claude\commands" "$RepoPath\commands"
cmd /c mklink /J "$HOME\.claude\skills"   "$RepoPath\skills"
```

**macOS / Linux:**

```bash
# Adjust REPO_PATH to where you cloned the repo
REPO_PATH="$HOME/source/repos/claude-config"

# Remove existing dirs if present
rm -rf ~/.claude/commands ~/.claude/skills

# Create symlinks
ln -s "$REPO_PATH/commands" ~/.claude/commands
ln -s "$REPO_PATH/skills"   ~/.claude/skills
```

### 3. Verify

Open Claude Code in any directory and type `/` — you should see `new-project` and `security-check` in the list.

## Usage

```bash
# Navigate to where you want the project created
cd ~/source/repos

# Start Claude Code
claude

# Type /new-project — follow the guided phases
# Phase 1: generates the codebase with placeholder values
# Phase 2: walks through Azure resource creation
# Phase 3: wires up real values and verifies builds

# Run a security audit on any project:
# /security-check
```

## Adding new commands or skills

**Commands** — create a `.md` file in `commands/`. The filename becomes the slash command name (e.g., `commands/deploy.md` → `/deploy`).

**Skills** — create a directory in `skills/` with a `SKILL.md` file. The directory name becomes the skill name. Skills support YAML frontmatter, supporting files, and `${CLAUDE_SKILL_DIR}` for portable path references. See [Claude Code skills docs](https://docs.anthropic.com/en/docs/claude-code/skills) for details.

Changes are immediately available through the junctions — no need to copy or re-link.

## What stays project-specific

This repo holds **global** config only. Project-specific items stay in each project's repo:

| Item | Location | Scope |
|------|----------|-------|
| `CLAUDE.md` | `<project>/CLAUDE.md` | Project instructions |
| `.claude/docs/` | `<project>/.claude/docs/` | Project reference docs |
| `.claude/settings.local.json` | `<project>/.claude/settings.local.json` | Project permissions |
| Memory files | `~/.claude/projects/...` | Auto-managed per project |
