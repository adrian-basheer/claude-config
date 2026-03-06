# Interactive Azure setup guide

Walk through Azure resource creation step-by-step, collaboratively building a `project-config.json` with real values.

## What this command does

Guides the user through each runsheet interactively — explaining what to do in Azure Portal, running `az` CLI commands where possible, asking for values to paste back, and progressively filling in the config JSON. At the end, produces a completed `project-config.json` ready for `/azure-connect`.

## Steps

1. **Find the project:**
   - Look for `project-config.json` in the current directory
   - If not found, ask the user for the project path
   - Read the current config to see what's already filled in vs still placeholder

2. **Read the runsheets** from the project's `runsheets/` folder (or from `C:\Users\adria\source\repos\claude-config\scaffold\runsheets\`)
   - Start with `00-overview.md` to show the dependency graph and order
   - Determine which steps are already done (values already in config)

3. **For each runsheet, in order:**

   a. **Show a summary** of what this step creates and how long it typically takes

   b. **For manual steps** (Azure Portal actions):
      - Explain clearly what to do, with specific navigation paths
      - Wait for the user to confirm completion
      - Ask them to paste back the generated values (GUIDs, URLs, etc.)

   c. **For automatable steps** (az CLI commands):
      - Show the command and explain what it does
      - Ask for confirmation before running
      - Capture output values automatically

   d. **Update `project-config.json`** with each value as it's collected
      - Save after each runsheet so progress isn't lost
      - Show what was filled in

   e. **Validate** — check that values look correct (GUIDs are valid format, URLs are well-formed, etc.)

4. **Track progress:**
   - After each runsheet, show a summary: "Completed 3/9 — next: Key Vault"
   - If the user needs to stop, the config JSON has all progress saved
   - On resume, skip already-completed steps

5. **After all runsheets:**
   - Display the completed `project-config.json`
   - Tell the user to run `/azure-connect` to wire everything up

## Important notes

- Never run destructive Azure commands without explicit confirmation
- If a resource already exists, detect it and skip creation
- If `az` is not logged in, help the user log in first
- Some steps require Azure Portal — cannot be fully automated (Entra tenant creation, admin consent, user flow setup)
- The user may not complete all steps in one session — that's fine, progress is saved in the config JSON
