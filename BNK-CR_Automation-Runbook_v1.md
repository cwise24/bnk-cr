# BNK-CR Automation Runbook — v1

How to add a new training page to the portal and let Claude Code review,
integrate, and PR it automatically.

**You write the HTML. The agent handles style review, the back button, the
portal card, the branch, the commit, the push, and the PR.** A human still
approves every merge.

---

## One-time setup

### 1. Install Claude Code CLI

```bash
npm install -g @anthropic-ai/claude-code
claude --version
```

Or see <https://code.claude.com> for platform installers.

### 2. Clone the repo

```bash
cd "/Volumes/syn_shr/claude_output/CLAUDE OUTPUTS"
git clone git@github.com:cwise24/bnk-cr.git
cd bnk-cr
```

### 3. Create `.env.local`

In the repo root. One line, `repo` scope on the token:

```bash
echo "GITHUB_TOKEN=github_pat_xxxxxxxxxxxx" > .env.local
```

Already gitignored — it will never be committed. Never paste it into chat.

### 4. Confirm you have the style guide

```bash
ls BNK-CR_Style-Guide_v1.md
```

This file is **gitignored on purpose** — it is agent instruction, not shipped
content, so it does not travel with `git clone`. Every machine that runs the
agent needs its own copy. If it is missing, the agent halts by design. Get it
from whoever set up the repo, or regenerate it from `index.html`.

### 5. Verify push access

```bash
ssh -T git@github.com          # expect: "Hi cwise24! You've successfully authenticated"
git fetch origin && git branch -a | grep claude-dev
```

---

## Adding a new page — the normal loop

### Step 1 — Drop the file in

Save your new page in the repo root, named `Project_Content-Type_vN.html`:

```bash
cd "/Volumes/syn_shr/claude_output/CLAUDE OUTPUTS/bnk-cr"
cp ~/Desktop/MyNewModule_Visualization_v1.html .
```

You do **not** need to add the back button, match the palette, or edit
`index.html` yourself — the agent does all of that. Getting close to the schema
just means fewer flagged items.

**The one thing the agent will not let slide:** every page needs a
back-to-portal button. If yours doesn't have one, the agent injects it. If it
can't inject cleanly — no `<style>` block, no `<body>` to anchor to — the run
halts with no PR. The pattern it enforces:

```html
</style>
</head>
<body>
<nav class="back-nav">
  <a href="index.html" class="back-link">&#8592; Training Portal</a>
</nav>
```

plus matching `.back-nav`, `.back-link`, and `.back-link:hover` CSS. Full block
in §4 of the style guide. Note that `.back-nav` is a **CSS class, not a file** —
it lives inline in each page.

### Step 2 — Start from claude-dev

```bash
git fetch origin
git checkout claude-dev
git pull origin claude-dev
```

### Step 3 — Run the agent

```bash
claude
```

Then in the session:

```
/review-content MyNewModule_Visualization_v1.html
```

Or let it pick up the next unprocessed file on its own:

```
/review-content
```

Fully non-interactive (for scripts and cron):

```bash
claude -p "/review-content MyNewModule_Visualization_v1.html" --permission-mode acceptEdits
```

### Step 4 — Approve tool access

On first run Claude asks to use `Read`, `Edit`, `Write`, `Bash`, `Grep`, and
`Glob`. Approve all six. Choose "always allow" for this project to skip the
prompt on later runs.

### Step 5 — Review the PR

The agent prints a PR link. Open it and check:

- **Auto-fixes applied** — mechanical schema corrections, each citing a
  style-guide section
- **Flagged for review** — judgment calls the agent deliberately did not touch
- **`index.html` diff** — one new card, correct module number and theme class

Merge into `claude-dev` when it looks right. The agent never merges.

---

## What the agent does on each run

1. Reads `BNK-CR_Style-Guide_v1.md` — halts if missing
2. Syncs `claude-dev`
3. Picks the oldest unprocessed HTML file
4. **Gates on the back-to-portal button** — injects it if missing, rewrites
   absolute hrefs to relative, translates legacy tokens; halts if it cannot
   inject cleanly
5. Reviews the page against the rest of the §9 checklist
6. Auto-fixes mechanical issues: `:root` tokens, font links, grid overlay,
   CDN and library versions
7. Flags judgment calls without changing them
8. Adds a card to `index.html` with the next module number and themed class
9. Branches `content-review-<UTC-timestamp>`, commits, pushes
10. Opens a PR into `claude-dev` via the GitHub API
11. Logs to `.processed-files.log` and reports the PR link

**One file per run.** Run it again for the next page.

---

## Running it on a schedule

Hourly scan via cron:

```bash
crontab -e
```

```cron
0 * * * * cd "/Volumes/syn_shr/claude_output/CLAUDE OUTPUTS/bnk-cr" && \
  /usr/local/bin/claude -p "/review-content" --permission-mode acceptEdits \
  >> ~/bnk-cr-review.log 2>&1
```

Adjust the `claude` path to match `which claude`. Because the agent skips files
already logged as `success`, an empty run is a no-op.

---

## State and re-runs

`.processed-files.log` in the repo root, one line per attempt:

```
MyNewModule_Visualization_v1.html | 2026-07-31T18:04:12Z | success | https://github.com/cwise24/bnk-cr/pull/12
```

**Force a re-run** of a file — remove its line from the log, or:

```bash
rm MyNewModule_Visualization_v1.html.processed
```

**Skip a file** — mark it processed without running the agent:

```bash
touch MyNewModule_Visualization_v1.html.processed
```

---

## Troubleshooting

| Symptom | Cause / fix |
|---|---|
| `command not found: claude` | CLI not installed or not on `PATH`. `npm install -g @anthropic-ai/claude-code` |
| Agent halts: "style guide missing" | `BNK-CR_Style-Guide_v1.md` absent. It is gitignored and does not clone — copy it in manually. |
| `/review-content` not recognized | `.claude/commands/review-content.md` missing, or you launched `claude` from outside the repo root |
| `401` / `Bad credentials` from the API | Token expired or lacks `repo` scope. Regenerate at github.com → Settings → Developer settings |
| `Permission denied (publickey)` on push | SSH key not loaded. `ssh-add ~/.ssh/id_ed25519`, verify with `ssh -T git@github.com` |
| PR created against `main` | Wrong base. The agent targets `claude-dev` — confirm the branch exists on origin |
| Same file PR'd twice | Duplicate or missing `.processed-files.log` entry. Dedupe the log |
| Agent flags 5+ items and stops | The page diverges too far from the schema. Bring it closer by hand, then re-run |
| Agent asks about uncommitted changes | You have unrelated work in the tree. Commit or stash it first |

---

## Guardrails

The agent is constrained to:

- **Never delete files** — anywhere, for any reason
- **Never merge a PR**, and never push directly to `claude-dev` or `main`
- **Never modify** the four legacy pages listed in style guide §8
- **Never commit** `.env.local` or `BNK-CR_Style-Guide_v1.md`
- **Never print or log** the GitHub token

Every change reaches `claude-dev` through a pull request you approve.

---

## Files involved

| Path | Committed? | Purpose |
|---|---|---|
| `BNK-CR_Style-Guide_v1.md` | No (gitignored) | Color / font / style schema. Agent reads it every run. |
| `BNK-CR_Automation-Runbook_v1.md` | Yes | This file. |
| `.claude/agents/html-content-reviewer.md` | No (gitignored) | Agent definition. |
| `.claude/commands/review-content.md` | No (gitignored) | `/review-content` slash command. |
| `.env.local` | No (gitignored) | `GITHUB_TOKEN`. |
| `.processed-files.log` | Yes | Run history / dedupe state. |
| `index.html` | Yes | Portal. Agent adds one card per approved page. |

### Three files do not travel with `git clone`

`.gitignore` excludes `.env.local`, `BNK-CR_Style-Guide_v1.md`, and `.claude/`.
A fresh clone therefore has **no agent, no slash command, and no style guide** —
`/review-content` will not exist and the agent would halt anyway.

Every machine that runs the automation needs these copied in by hand:

```
.env.local
BNK-CR_Style-Guide_v1.md
.claude/agents/html-content-reviewer.md
.claude/commands/review-content.md
```

Get them from a machine that already works, or from whoever set up the repo.
Verify before your first run:

```bash
ls .env.local BNK-CR_Style-Guide_v1.md \
   .claude/agents/html-content-reviewer.md \
   .claude/commands/review-content.md
```

If you would rather the agent and command ship with the repo, drop the
`.claude` line from `.gitignore` — they contain no secrets. The style guide and
`.env.local` should stay ignored.

---

**Version:** v1 · **Last updated:** 2026-07-31
