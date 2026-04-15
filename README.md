# react-clean-skills

Claude Code plugin marketplace for building React + Next.js apps with hybrid Clean Architecture, Mantine UI, and TanStack Query.

## Plugins

| Plugin              | Purpose                                                                                                 |
| ------------------- | ------------------------------------------------------------------------------------------------------- |
| `architector`       | Design & implement features across Clean Architecture layers (Domain → Use-Cases → Adapters → UI).      |
| `component-creator` | Create React components with `composeHooks` Smart/Dumb separation, Mantine UI, i18n, and file layout.   |

Both are model-invoked: Claude picks them automatically when a task matches their descriptions.

## Install

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
    "architector@react-clean-skills": true,
    "component-creator@react-clean-skills": true
  }
}
```

When a teammate trusts the repo folder, Claude Code prompts them to install the marketplace and plugins automatically.

### Option B — interactive, per-user

```shell
/plugin marketplace add clicktronix/react-clean-skills
/plugin install architector@react-clean-skills
/plugin install component-creator@react-clean-skills
```

### Option C — CLI, per-user

```bash
claude plugin marketplace add clicktronix/react-clean-skills
claude plugin install architector@react-clean-skills --scope user
claude plugin install component-creator@react-clean-skills --scope user
```

## Using the skills

After install, run `/reload-plugins`. Invoke via:

```shell
/react-clean-skills:architector
/react-clean-skills:component-creator
```

Or just describe a task — Claude picks the right skill based on its frontmatter `description`.

## Assumptions

These skills expect a project shaped like:

- **Framework**: Next.js (App Router), React 19, TypeScript
- **UI**: Mantine UI, CSS Modules
- **Validation**: Valibot
- **State**: TanStack Query (server), Zustand (page UI), React Context (global)
- **Architecture**: hybrid Clean Architecture with `domain/`, `use-cases/`, `adapters/inbound/next/`, `adapters/outbound/`, `ui/`

A reference template applying these conventions: [clicktronix/fullstack-ai-template](https://github.com/clicktronix/fullstack-ai-template).

## Versioning

Semver. Breaking changes to skill instructions bump major; additions bump minor; edits bump patch.

Run `claude plugin update` to pull the latest after a release.

## Contributing

Edit `plugins/<name>/skills/<name>/SKILL.md`. Bump the plugin `version` in `plugins/<name>/.claude-plugin/plugin.json` and the matching entry in `.claude-plugin/marketplace.json`. Open a PR.

## License

MIT — see [LICENSE](./LICENSE).
