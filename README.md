# ios-claude-tools

A Claude Code [plugin marketplace](https://code.claude.com/docs/en/plugin-marketplaces) for iOS / Swift developers.

## What's in here

| Plugin | What it gives you |
|---|---|
| **ios-dev** | iOS / Swift developer commands. Currently `/ios-dev:review` — a multi-agent code review of a PR or local branch diff. More commands will be added to this plugin over time. |
| **swiftui-pro** ¹ | Deep SwiftUI review skill. `/ios-dev:review` picks it up automatically when present. |
| **swift-testing-pro** ¹ | Deep Swift Testing review skill. |
| **swift-concurrency-pro** ¹ | Deep Swift concurrency review skill. |

¹ Sourced from [Paul Hudson](https://github.com/twostraws)'s upstream repos (MIT). They are **optional** — `/ios-dev:review` runs with seven core agents on its own, and adds a deeper pass for any of these that happens to be installed.

> Plugin commands are always namespaced as `/<plugin>:<command>`, so the review command is invoked as **`/ios-dev:review`** (never a bare `/review`, which is a built-in command).

## Install (per developer)

```
/plugin marketplace add wolfspy/ios-claude-tools
/plugin install ios-dev@ios-tools
```

Optionally add the deep-dive skills:

```
/plugin install swiftui-pro@ios-tools
/plugin install swift-testing-pro@ios-tools
/plugin install swift-concurrency-pro@ios-tools
```

## Install (whole team, per project)

Commit this to a project repo's `.claude/settings.json` so everyone who clones and
trusts the repo is prompted to install the same tooling:

```json
{
  "extraKnownMarketplaces": {
    "ios-tools": {
      "source": { "source": "github", "repo": "wolfspy/ios-claude-tools" }
    }
  },
  "enabledPlugins": {
    "ios-dev@ios-tools": true,
    "swiftui-pro@ios-tools": true,
    "swift-testing-pro@ios-tools": true,
    "swift-concurrency-pro@ios-tools": true
  }
}
```

## Usage

```
/ios-dev:review              # reviews the PR for the current branch
/ios-dev:review 1234         # reviews PR #1234
/ios-dev:review feature/foo  # reviews the PR for branch feature/foo
```

If the current branch has no PR yet (or the repo has no remote), `/ios-dev:review`
falls back to reviewing the local branch diff against the base branch, including any
uncommitted changes.

## Versioning

`plugins/ios-dev/.claude-plugin/plugin.json` carries a semver `version`. Bump it on
each release. Pushing commits without bumping the version does **not** trigger an
update for installed users.

Consumers update with:

```
/plugin marketplace update ios-tools
```

This refreshes the `ios-tools` marketplace and pulls the latest versions of every
plugin installed from it (`ios-dev` plus any of the Pro skills) in one command —
there is no per-plugin `/plugin update` command. Run `/reload-plugins` afterwards to
activate the changes without restarting. Users can also enable auto-update for the
marketplace (`/plugin` → **Marketplaces** → `ios-tools` → **Enable auto-update**) so
new versions are pulled automatically at startup.

## Adding more commands

New commands live in the same plugin — drop another `.md` file into
`plugins/ios-dev/commands/` and it becomes `/ios-dev:<command>`. Unrelated tools can
instead be added as a separate plugin under `plugins/` with its own entry in
`.claude-plugin/marketplace.json`.
