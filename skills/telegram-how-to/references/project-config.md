# Configure mode — force-trigger Telegram bot skills in a project

Adds a `## Required Telegram bot skills` block to the project's agent config so bot conventions always load.

## When to use

- A bot repo must always enforce `makeBot()`, test coverage, or deploy contract.
- The team shares skills via `.agents/skills/` or `agnt-skills` and wants consistent routing.
- Agents keep missing tests or deploy rules without explicit triggers.

## Step 1 — Detect the project config file

Check in this precedence order:

```
1. AGENTS.md          (bot repos — agntdev bot-starter ships this)
2. CLAUDE.md          (Claude Code)
3. .cursor/rules      (Cursor)
4. .github/copilot-instructions.md  (GitHub Copilot)
```

Use `Glob` at the project root. If multiple exist, update all that apply. If none exist, ask which file to create.

## Step 2 — Idempotency check

Grep for an existing block:

```bash
grep -n "## Required Telegram bot skills" AGENTS.md CLAUDE.md 2>/dev/null
```

If present, confirm whether to replace the skill list or skip.

## Step 3 — Confirm the skill set

**Recommended default for agntdev bot repos:**

```
telegram-how-to
telegram-bot-basics
telegram-test-specs
```

**Add based on codebase:**

| Signal | Add skill |
| --- | --- |
| `ctx.session` / flow steps | `telegram-bot-sessions` |
| `inline_keyboard` / `callback_data` / toolkit UI imports | `telegram-bot-ui` |
| `handleUpdate` tests / mocks / `vi.mock` | `telegram-test-advanced` |
| `Dockerfile`, deploy errors, `REDIS_URL` | `telegram-bot-deploy` |
| `agnt` CLI / connect code / bounty context | `agnt-cli-builder` |

Remind the user: each always-loaded skill adds description tokens to every session (~80–150 tokens per skill).

## Step 4 — Write the block

### Template (agntdev bot repo)

```markdown
## Required Telegram bot skills

Load these skills at the start of every bot-related task in this repo.

- `telegram-how-to` — route to the right domain skill(s)
- `telegram-bot-basics` — makeBot(), routing, toolkit patterns
- `telegram-test-specs` — all specs must pass before publish

### Non-negotiables

- `makeBot()` returns a **new** bot per call (harness isolation)
- `await ctx.answerCallbackQuery()` in every callback handler
- Never commit `BOT_TOKEN`; use `process.env.BOT_TOKEN`
- Entry point: `dist/index.js` (see telegram-bot-deploy)
```

Adjust the skill list and non-negotiables to match Step 3.

### Insertion point

- Empty file → block at top.
- Existing content → append after last section with a blank line.
- Existing `## Required Telegram bot skills` → replace only the list inside.

## Step 5 — Confirm to the user

Summarize:

- Which file(s) updated
- Which skills are always-loaded
- Approximate token cost
- Point to `telegram-how-to` for intent routing on specific tasks

## Skill path resolution

Agents resolve skills from (first match wins):

1. `<repo>/.agents/skills/<name>/SKILL.md`
2. `<repo>/.grok/skills/<name>/SKILL.md`
3. User-global `~/.grok/skills/` or `~/.agents/skills/`
4. Installed bundle (e.g. `agnt-skills/skills/`)

Use short names (`telegram-bot-basics`) in the block when skills are copied into `.agents/skills/`.