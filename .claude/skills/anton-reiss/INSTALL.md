# Setup

## 1. Install the skill

**Claude Code, this repo only:**
```
cp -r anton-reiss <your-repo>/.claude/skills/
```

**Claude Code, everywhere:**
```
cp -r anton-reiss ~/.claude/skills/
```

Restart Claude Code. Type `/` and confirm `anton-reiss` appears.

**Claude.ai (planning sessions):** upload the `.skill` file and save it, or paste `SKILL.md` at the top of a new conversation. The references won't auto-load in chat, so paste `review-rubric.md` when you want a review there.

## 2. Scaffold the repo

```
mkdir -p ledger docs/adr backlog
touch ledger/STATE.md ledger/REVIEWS.md ledger/LESSONS.md PROJECT.md
```

Fill `PROJECT.md` from `assets/templates.md`. Leave the rest empty. Anton writes them.

## 3. Install the board

```
npm i -g backlog.md
backlog init "<project>"
```

Choose the MCP or CLI integration when it asks, so Anton can read and move tickets. Tasks land in the repo as markdown, so they diff in your PRs.

Optional: `ringleader/project-journal` gives you `/log decision "..."` for capturing things mid-session without breaking flow.

## 4. Point CLAUDE.md at the ledger

Add to `CLAUDE.md`:

```
At session start, read ledger/STATE.md, the open ticket, and `git log --oneline -20`
before doing anything else. ADRs in docs/adr/ are binding. If you contradict one,
the ADR wins.
```

Keep it short. Every line here costs tokens on every turn.

## 5. First session

Open a session and say: **"New project. Here's the PRD."** Anton goes to PLAN mode.

## Using it

- `review this` + paste a diff → REVIEW
- `next ticket` → TICKET
- `here's my design for <id>` → DESIGN
- `wrap up` → HANDOFF
- `exit mentor mode` → normal Claude, for when you actually need code for something outside the project

## Tuning

Anton is set harsh. He nitpicks style as well as substance. If it becomes noise rather than signal after a few weeks, edit the `## Voice` section in `SKILL.md` rather than arguing with him in-session. Arguing in-session teaches him to fold.

If he ever writes code for you: that's a bug in the skill, not a favour. Tell him, and tighten the iron law list in `SKILL.md`.
