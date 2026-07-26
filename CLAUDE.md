# my-personal-site

Personal site (angelospanag.me), built with [Hugo](https://gohugo.io/) and the
[Congo](https://github.com/jpanther/congo) theme.

## Tooling

Tool versions (Hugo, Go) are pinned in `mise.toml`. Use `mise run <task>` rather than calling
`hugo` directly — see `mise.toml` for the task list (`dev`, `build`, `deps`, `clean`).

Congo is installed as a [Hugo Module](https://gohugo.io/hugo-modules/) (`go.mod`/`go.sum`), not a
git submodule — there is no `themes/` checkout to keep in sync. `mise run deps` updates it.

There is no Go application code in this project (no `.go` files) — `go.mod` exists solely so Hugo
can resolve the theme module. There is deliberately no golangci-lint setup here (unlike other Go
projects) since there's nothing for it to lint; add it back if real Go code (e.g. a custom build
step) is ever introduced.

## Structure

- `content/_index.md` — homepage, using Congo's `profile` homepage layout (see
  `config/_default/params.toml` → `[homepage]`). Author identity (name, avatar, bio, social links)
  lives in `config/_default/languages.en.toml` under `[params.author]`, not in this file — this
  file only holds the markdown body shown below the profile card.
- `content/posts/` — blog posts.
- `static/images/` — images referenced by posts, referenced as `/images/...`.
- `assets/img/avatar.jpeg` — homepage avatar, resolved via `params.author.image` in
  `languages.en.toml`.
- `config/_default/` — split config (`hugo.toml`, `params.toml`, `languages.en.toml`,
  `menus.en.toml`, `markup.toml`).

## Post front matter conventions

```yaml
---
title: Post Title
date: "YYYY-MM-DD" # must be zero-padded, Hugo rejects e.g. 2023-4-3
tags: ["tag-one", "tag-two"]
draft: false
showTableOfContents: false # posts already have a hand-written TOC as the first list — avoid a duplicate auto-generated one
summary: "One-line description used in list pages / meta"
---
```

## Search

Congo ships its own Fuse.js-based search, enabled via `enableSearch = true` in `params.toml`.
