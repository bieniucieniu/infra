# Git

Every commit uses [Conventional Commits](https://www.conventionalcommits.org/en/v1.0.0/). A branch Claude creates uses [Conventional Branch](https://conventionalbranch.org/) with the `claude/` type. Cursor uses `cursor/`. Any other agent uses `ai/`.

The rest of the project instructions are in [AGENTS.md](AGENTS.md).

## Commits

`<type>[optional scope][!]: <description>`

- Types: `feat`, `fix`, `docs`, `style`, `refactor`, `perf`, `test`, `build`, `ci`, `chore`, `revert`
- Scope is the app or area (`cookbook`, `traefik`, `solid-kit`)
- Description is imperative, lowercase, and has no trailing period
- A breaking change adds `!` after the type or scope and a `BREAKING CHANGE:` footer
- One subject line. Add a body only when the why is not obvious

```
feat(cookbook): set the authkit domain
chore(solid-kit): update image
```

## Branches

`<type>/<description>`

```
claude/traefik-gateway-policy
cursor/cookbook-authkit-domain
ai/image-update-automation
```

Description is lowercase `a-z`, digits, and hyphens. Dots are for versions. No underscores, spaces, doubled hyphens, or a hyphen at either end.

`main`, `master`, and `develop` stay as they are. Create a branch only when the user asks. A commit on the current branch still uses Conventional Commits.
