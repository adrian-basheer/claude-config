# Claude Config

Personal Claude Code configuration — global slash commands, scaffold templates, and reusable patterns.

## Structure

```
claude-config/
├── commands/                        # Global slash commands (symlinked to ~/.claude/commands/)
│   ├── scaffold.md                  # /scaffold — generate a new project with placeholder values
│   ├── azure-setup.md               # /azure-setup — interactive guide through Azure resource creation
│   ├── azure-connect.md             # /azure-connect — replace placeholders with real Azure values
│   └── security-check.md            # /security-check — OWASP Top 10 security audit
├── scaffold/                        # Scaffold reference materials
│   ├── SCAFFOLD-SPEC.md             # Full spec for code generation
│   ├── project-config.template.json # Config template with all fields
│   ├── project-config.mixshare.json # Filled-in example (Mixshare project)
│   └── runsheets/                   # Step-by-step Azure resource setup guides
│       ├── 00-overview.md
│       ├── 01-entra-external-id-setup.md
│       ├── 02-app-registrations.md
│       ├── 03-storage-account.md
│       ├── 04-key-vault.md
│       ├── 05-app-services.md
│       ├── 06-microsoft-graph-api.md
│       ├── 07-azure-devops-pipeline.md
│       ├── 08-api-management.md
│       └── 09-cosmos-db.md
└── README.md
```

## Workflow

The three commands form a pipeline for creating new projects:

```
/scaffold          →  Buildable code with TODO_* placeholders
                       Includes runsheets/ folder for reference

/azure-setup       →  Interactive guide through Azure Portal + az CLI
                       Produces completed project-config.json

/azure-connect     →  Replaces all placeholders with real values
                       Project is ready to deploy
```

Each command is independent — you can skip `/azure-setup` if you fill in `project-config.json` manually, or re-run `/azure-connect` when moving to a different Azure subscription.

## Setup (new machine)

### 1. Clone this repo

```bash
cd C:\Users\YourUser\source\repos
git clone https://github.com/adrian-basheer/claude-config.git
```

### 2. Create a junction to ~/.claude/commands/

A junction links `~/.claude/commands/` to the repo's `commands/` folder so Claude Code picks them up globally. No admin privileges needed.

```powershell
# Remove existing commands dir if present
if (Test-Path "$HOME\.claude\commands") { Remove-Item "$HOME\.claude\commands" -Recurse }

# Create junction (adjust the target path to where you cloned the repo)
cmd /c mklink /J "$HOME\.claude\commands" "C:\Users\YourUser\source\repos\claude-config\commands"
```

### 3. Verify

Open Claude Code in any directory and type `/` — you should see `scaffold`, `azure-setup`, and `azure-connect` in the command list.

## Usage — scaffolding a new project

The slash commands are **global** — available regardless of which folder you run Claude from. To scaffold a new project:

```bash
# Navigate to where you want the project created
cd C:\Users\YourUser\source\repos

# Start Claude Code
claude

# Type /scaffold — follow the prompts (project name, etc.)
# The project folder is created in the current directory

# Later, when ready to set up Azure resources:
# /azure-setup — interactive walkthrough, builds project-config.json

# Finally, wire up real values:
# /azure-connect — replaces TODO_* placeholders with config values
```

## Adding new commands

Create a new `.md` file in `commands/`. The filename (without extension) becomes the slash command name. For example, `commands/deploy.md` becomes `/deploy`.

The symlink means changes here are immediately available — no need to copy or re-link.

## What stays project-specific

This repo holds **global** config only. Project-specific items stay in each project's repo:

| Item | Location | Scope |
|------|----------|-------|
| `CLAUDE.md` | `<project>/CLAUDE.md` | Project instructions |
| `.claude/docs/` | `<project>/.claude/docs/` | Project reference docs |
| `.claude/settings.local.json` | `<project>/.claude/settings.local.json` | Project permissions |
| Memory files | `~/.claude/projects/...` | Auto-managed per project |
