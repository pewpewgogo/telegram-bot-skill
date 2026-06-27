# telegram-bot-skills

A comprehensive, cross-referenced **agent skill collection for building Telegram bots
with grammY** (TypeScript/JavaScript) — modeled on the structure of
[samber/cc-skills-golang](https://github.com/samber/cc-skills-golang).

Each `skills/<name>/SKILL.md` is a self-contained skill an agent loads when triggered.
An orchestrator (`telegram-how-to`) routes any bot task to the right skill(s), and
[`RULES.md`](./RULES.md) is the consolidated cross-cutting rule set.

## Two tiers

- **General grammY core** — framework knowledge that works for any bot. Use these by default.
- **agnt-gm platform tier** — correct but *platform-specific* skills for the
  [agnt-gm.ai](https://agnt-gm.ai) bot-building pipeline (CLI, BotSpec test harness,
  container deploy contract). Load only when working on that platform.

## Skills

### General grammY core

| Skill | What it does |
|---|---|
| [telegram-how-to](./skills/telegram-how-to/SKILL.md) | **Orchestrator.** Routes any task to the right skill(s); disambiguation tables; `/telegram-how-to configure` writes project rules |
| [telegram-bot-basics](./skills/telegram-bot-basics/SKILL.md) | Entry point, command/`hears`/filter-query routing, `ctx`, middleware & `Composer`, `bot.catch`, project structure |
| [telegram-bot-ui](./skills/telegram-bot-ui/SKILL.md) | `InlineKeyboard`/`Keyboard`, callback routing & `answerCallbackQuery`, `@grammyjs/menu`, pagination, confirm dialogs |
| [telegram-bot-sessions](./skills/telegram-bot-sessions/SKILL.md) | `ctx.session`, session key/scope, storage adapters (memory/Redis/file/free), `lazySession`, migrations |
| [telegram-bot-conversations](./skills/telegram-bot-conversations/SKILL.md) | Multi-step dialogs with `@grammyjs/conversations` — `wait`/`waitFor`/forms, the replay model, `conversation.external()` |
| [telegram-bot-messages](./skills/telegram-bot-messages/SKILL.md) | HTML/MarkdownV2 + `@grammyjs/parse-mode`, entities, edit/delete, media (`InputFile`, `file_id`), downloads |
| [telegram-bot-scaling](./skills/telegram-bot-scaling/SKILL.md) | `@grammyjs/runner`, `sequentialize`, 429/flood with `apiThrottler`/`auto-retry`, `@grammyjs/ratelimiter` |
| [telegram-bot-deploy](./skills/telegram-bot-deploy/SKILL.md) | Polling vs webhook, `webhookCallback` adapters, `setWebhook` + secret token, serverless/edge, graceful shutdown |
| [telegram-bot-payments](./skills/telegram-bot-payments/SKILL.md) | `sendInvoice`, Telegram Stars (`XTR`), `pre_checkout_query`, `successful_payment`, refunds |
| [telegram-bot-mini-apps](./skills/telegram-bot-mini-apps/SKILL.md) | Web App buttons, `web_app_data`, **initData HMAC validation**, inline mode |
| [telegram-bot-testing](./skills/telegram-bot-testing/SKILL.md) | Generic grammY testing — transformer capture + synthetic `Update` + `bot.handleUpdate` (vitest/jest) |
| [telegram-bot-security](./skills/telegram-bot-security/SKILL.md) | Token hygiene, webhook secret, input validation, authz by `ctx.from.id`, abuse/flood limits |

### agnt-gm platform tier

| Skill | What it does |
|---|---|
| [agnt-cli-builder](./skills/agnt-cli-builder/SKILL.md) | Claim → ship → earn on agnt-gm: `agnt ready`, inspect the DAG, claim a task, ship the PR, track TON rewards |
| [telegram-test-specs](./skills/telegram-test-specs/SKILL.md) | BotSpec dialog specs, `SendShorthand`, `ExpectedCall`, the harness CLI, command-coverage gate |
| [telegram-test-advanced](./skills/telegram-test-advanced/SKILL.md) | Beyond BotSpec — mocks, API failures (429, blocked user), payments, raw `handleUpdate` tests |
| [agntdev-deploy](./skills/agntdev-deploy/SKILL.md) | The agnt-gm deploy contract — `dist/index.js`, `.npmrc`, `REDIS_URL`, platform container, crash-loop debug |

## Structure

```
RULES.md                       # consolidated cross-cutting rule set
skills/
  telegram-how-to/             # orchestrator: intent → skill set, disambiguation, configure
    references/{by-category,disambiguation,project-config}.md
  telegram-bot-basics/         # general grammY core ↓
  telegram-bot-ui/
  telegram-bot-sessions/
  telegram-bot-conversations/
  telegram-bot-messages/
  telegram-bot-scaling/
  telegram-bot-deploy/
  telegram-bot-payments/
  telegram-bot-mini-apps/
  telegram-bot-testing/
  telegram-bot-security/
  agnt-cli-builder/            # agnt-gm platform tier ↓
  telegram-test-specs/
  telegram-test-advanced/
  agntdev-deploy/
```

## How skills work

Each skill is a markdown file the agent reads when triggered:

- `SKILL.md` — instructions with frontmatter (`name`, `description` incl. a `Triggers:` line, `compatibility`, `license`)
- `references/` — supporting docs loaded on demand

`node scripts/validate-skills.mjs` checks frontmatter and that every `references/` link resolves.

## License

MIT — see [LICENSE](./LICENSE)
