# skills

A Claude Code plugin marketplace. Add it once, install whatever you need from it.

```
/plugin marketplace add https://github.com/Uzasch/skills.git
```

Use the full `https://` URL, not the `Uzasch/skills` shorthand — the shorthand resolves to
`git@github.com:` and fails with `Permission denied (publickey)` on any machine without a GitHub
SSH key, even though this repo is public.

## Plugins

| Plugin | What it does |
| :--- | :--- |
| [`implement-codex-review-loop`](plugins/implement-codex-review-loop) | Implement a PRD or set of issues, then harden it in a Claude↔Codex review loop until Codex approves. |

```
/plugin install implement-codex-review-loop@skills
```

## Adding a plugin to this marketplace

1. Create `plugins/<name>/` with a `.claude-plugin/plugin.json` and a `skills/` directory.
2. Add an entry to `.claude-plugin/marketplace.json` with `"source": "./plugins/<name>"`.
3. Run `claude plugin validate . --strict`.

Plugins here may declare each other in `dependencies` and they resolve without extra configuration,
since dependencies resolve within the declaring plugin's own marketplace. Depending on a plugin in
a *different* marketplace requires `allowCrossMarketplaceDependenciesOn` here — which is why
`codex-orchestrator` is listed as an entry in this marketplace, sourced unmodified from
[upstream](https://github.com/alexzh3/codex-orchestrator), rather than being vendored or
cross-referenced.
