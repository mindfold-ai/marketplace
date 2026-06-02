# Trellis Marketplace

Downloadable templates for Claude Code users.

## Structure

```
marketplace/
├── skills/      # Claude Skills (.claude/skills/)
├── commands/    # Slash Commands (.claude/commands/)
│   ├── trellis-plan.md      # Plan: brainstorm → prd/design/implement → validate
│   └── trellis-execute.md   # Execute: implement → check → update-spec → commit → finish
├── agents/      # Sub-agent definitions (.claude/agents/)
└── specs/       # Spec templates (.trellis/spec/)
```

## Usage

### Install via npx skills

```bash
npx skills add mindfold-ai/Trellis/marketplace
```

### Manual Installation

Copy the desired template to your project's corresponding directory.
