# react-clean-skills

Portable Claude Code and Codex plugin marketplace for applying the `fullstack-ai-template` architecture profile: strict Clean Architecture boundaries plus `composeHooks` Smart/Dumb React components.

## Plugin

| Plugin                | Skills                               | Purpose                                                                                             |
| --------------------- | ------------------------------------ | --------------------------------------------------------------------------------------------------- |
| `react-clean-skills`  | `architector`, `component-creator`   | Design feature slices and create React components using the strict Fullstack AI Template profile. |

Both skills are model-invoked: Claude Code and Codex can select them automatically when a task matches the skill frontmatter `description`.

## Claude Code Install

### Option A — committed in your repo (recommended for teams)

Add to `.claude/settings.json`:

```json
{
  "extraKnownMarketplaces": {
    "react-clean-skills": {
      "source": { "source": "github", "repo": "clicktronix/react-clean-skills" }
    }
  },
  "enabledPlugins": {
    "react-clean-skills@react-clean-skills": true
  }
}
```

When a teammate trusts the repo folder, Claude Code prompts them to install the marketplace and plugins automatically.

### Option B — interactive, per-user

```shell
/plugin marketplace add clicktronix/react-clean-skills
/plugin install react-clean-skills@react-clean-skills
```

### Option C — CLI, per-user

```bash
claude plugin marketplace add clicktronix/react-clean-skills
claude plugin install react-clean-skills@react-clean-skills --scope user
```

### Claude Code Usage

After install, run `/reload-plugins`. Invoke via:

```shell
/react-clean-skills:architector
/react-clean-skills:component-creator
```

Or just describe a task — Claude picks the right skill based on its frontmatter `description`.

## Codex Install

This repository also contains a Codex marketplace at `.agents/plugins/marketplace.json` and a Codex plugin manifest at `plugins/react-clean-skills/.codex-plugin/plugin.json`.

When this marketplace is available to Codex, install the `react-clean-skills` plugin from `/plugins`. The installed plugin exposes:

```text
$architector
$component-creator
```

You can also let Codex select the matching skill from the task description, or use `@react-clean-skills` when choosing the plugin in the composer.

## Architecture Profile

The skills are portable, but they are not generic prompts. They default to the rules from `clicktronix/fullstack-ai-template` and adapt only where the target repository has explicit equivalent conventions.

Default profile:

- **Framework**: Next.js App Router, React, TypeScript
- **Architecture**: `domain -> use-cases -> adapters -> ui/server-state -> ui/app` boundaries from the template docs
- **Validation**: Valibot schemas with inferred types
- **Backend role**: Supabase outbound adapters, replaceable only by a repository/gateway equivalent
- **Server state**: TanStack Query in `src/ui/server-state/<feature>/`
- **UI**: Mantine + CSS Modules
- **i18n**: `messages.json` + `TranslationText`
- **Component pattern**: `composeHooks(View)(useProps)` for any component with logic

A reference template applying these conventions with Mantine + Supabase + Valibot: [clicktronix/fullstack-ai-template](https://github.com/clicktronix/fullstack-ai-template).

## Versioning

Semver. Breaking changes to skill instructions bump major; additions bump minor; edits bump patch.

Run `claude plugin update` to pull the latest after a release.

## Contributing

Edit `plugins/react-clean-skills/skills/<name>/SKILL.md`. Bump the plugin `version` in:

- `plugins/react-clean-skills/.claude-plugin/plugin.json`
- `plugins/react-clean-skills/.codex-plugin/plugin.json`
- `.claude-plugin/marketplace.json`
- `.agents/plugins/marketplace.json`

Open a PR.

## License

MIT — see [LICENSE](./LICENSE).
