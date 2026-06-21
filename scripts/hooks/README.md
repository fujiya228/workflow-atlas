# Git hooks

Versioned hooks for this repo. Enable them once per clone:

```bash
git config core.hooksPath scripts/hooks
```

## `pre-push`

Refuses to push to **main** when files under `skills/` changed but
`.claude-plugin/plugin.json`'s `version` did not. The marketplace only
recognizes a release when that version moves, so this prevents silently
shipping skill changes that installers will never receive.

The hook checks only *that* a bump happened — choosing patch vs minor vs major
is left to you. WIP feature-branch pushes are not gated. Emergency override:
`git push --no-verify`.
