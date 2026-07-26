# 🌐 Angelos Panagiotopoulos personal website

🔗 Live Site: [https://www.angelospanag.me/](https://www.angelospanag.me/)

Hello, I'm Angelos, a Software Engineer specializing in backend and frontend development.

Built with [Hugo](https://gohugo.io/) and [Go](https://go.dev/), using the
[Congo](https://github.com/jpanther/congo) theme installed as a
[Hugo Module](https://gohugo.io/hugo-modules/) (Go-based dependency management, no git submodules).

## Prerequisites

- [mise](https://mise.jdx.dev/) — manages the Hugo and Go versions pinned in `mise.toml`

```bash
mise install
```

## Run development server

```bash
mise run dev
```

The development server will be available on http://localhost:1313

## Build for production

```bash
mise run build
```

The static site is generated into `public/`.

## Update the Congo theme module

```bash
mise run deps
```

## Content

- `content/_index.md` — homepage / about me (Congo "profile" layout)
- `content/posts/` — blog posts
- `static/images/` — images referenced by posts
- `assets/img/avatar.jpeg` — profile picture used on the homepage
- `config/_default/` — site configuration (theme params, menus, author/social links)
