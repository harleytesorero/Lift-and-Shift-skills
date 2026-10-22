# Lift-and-Shift-skills

Collection of skills used to support Lift and Shift project activities and automation.

| Folder | Type | What it does |
|--------|------|--------------|
| `columbia-cleanup` | Standalone skill | Finds and removes/renames client-specific references (Columbia, CIG, AMBest, NAICO) with a human approval gate before any change |
| `guidewire-version-upgrade` | Standalone skill | Upgrades a Guidewire PolicyCenter LOB from one platform version to another using a three-way delta approach |
| `pc-lob-removal` | Project toolkit (agents + skills + context) | Agent knowledge base for PolicyCenter LOB removal. See `pc-lob-removal/README.md` |

## Prerequisites

- [Claude Code](https://claude.com/claude-code) installed
- This repo cloned locally:
  ```
  git clone https://github.com/harleytesorero/Lift-and-Shift-skills.git
  ```

## Installing and using `columbia-cleanup` and `guidewire-version-upgrade`

These are standalone skills. Install them once on your computer and they are available in every project.

1. Copy the skill folders into your personal Claude skills folder:

   | OS | Destination |
   |----|-------------|
   | Windows | `C:\Users\<your-username>\.claude\skills\` |
   | macOS / Linux | `~/.claude/skills/` |

   Windows (PowerShell), from the cloned repo:
   ```powershell
   New-Item -ItemType Directory -Force "$env:USERPROFILE\.claude\skills"
   Copy-Item -Recurse .\columbia-cleanup "$env:USERPROFILE\.claude\skills\"
   Copy-Item -Recurse .\guidewire-version-upgrade "$env:USERPROFILE\.claude\skills\"
   ```

   The result should look like this (each skill's `SKILL.md` directly inside its own folder):
   ```
   .claude/skills/columbia-cleanup/SKILL.md
   .claude/skills/guidewire-version-upgrade/SKILL.md
   ```

2. Restart Claude Code if it was already running.
3. Open Claude Code in the project you want to work on (`cd` into the project, then run `claude`).
4. Invoke a skill by typing its name as a slash command:
   ```
   /columbia-cleanup
   /guidewire-version-upgrade
   ```
   You can also ask in plain words (for example "run the Columbia cleanup") and Claude will pick up the skill.

> To share a skill with only one project instead of your whole computer, put it in `<project-root>/.claude/skills/` instead.

## Installing and using `pc-lob-removal`

`pc-lob-removal` is a full toolkit (agents, context, rules, skills, tools), so it is installed **per project**, not globally.

1. In the **root directory of the project** you are working on, create a `.claude` folder if it does not already exist.
2. Copy the **contents** of `pc-lob-removal` into that `.claude` folder (do not copy the `pc-lob-removal` folder itself). The project should end up like this:
   ```
   <project-root>/
   └── .claude/
       ├── agents/
       ├── context/
       ├── rules/
       ├── scripts/
       ├── skills/
       ├── test_plan/
       ├── tools/
       └── README.md
   ```
   Windows (PowerShell), from the cloned repo:
   ```powershell
   New-Item -ItemType Directory -Force "<project-root>\.claude"
   Copy-Item -Recurse .\pc-lob-removal\* "<project-root>\.claude\"
   ```
3. Open a terminal in the **project root directory** and start Claude Code there:
   ```
   cd <project-root>
   claude
   ```
   Claude Code only picks up `.claude` from the directory you launch it in (or its parents), so always start it from the project root.
4. Invoke the skills as slash commands, for example:
   ```
   /pc-current-state
   /pc-plan-lob-removal
   /pcf-find-usages
   /entity-find-usages
   /gosu-find-usages
   /grill-me
   ```
   The agents (`gw-pcf-agent`, `gw-gosu-agent`, `gw-entity-agent`, `gw-typelist-agent`, `gw-build-agent`, `gw-planner-agent`) are available to Claude automatically once the files are in `.claude/agents/`.

See `pc-lob-removal/README.md` for how the agents, context, rules and skills are organized.

## Updating

When this repo changes, pull the latest and copy the folders over your installed copies again:
```
git pull
```
