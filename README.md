# 🌐 Angelos Panagiotopoulos personal website

🔗 Live Site: [https://www.angelospanag.me/](https://www.angelospanag.me/)

Hello, I am Angelos, a Contractor Software Engineer.

## Where can I find your CV?

It is always available on my website, along with my contact details: https://angelospanag.me/about

## Favourite languages/tools for software development?

Mainly Go and Python for backend development and React/TypeScript for frontend development. This blog is [using Next.js](https://github.com/angelospanag/my-personal-site) with a [starter blog template](https://github.com/timlrx/tailwind-nextjs-starter-blog) and [Vercel](https://vercel.com/) for automating its deployment.

## Other interests?

I love listening to heavy metal and playing electric guitar, reading science-fiction and fantasy books, PC gaming and especially RPGs, exercising/running.

## Prerequisites

- [Node 24](https://nodejs.org/en/download)
- [pnpm](https://pnpm.io/next/installation)

**MacOS (using `brew`)**

```bash
brew install node@24 pnpm
```

**Ubuntu/Debian**

```bash
# Download and install nvm:
curl -o- https://raw.githubusercontent.com/nvm-sh/nvm/v0.40.3/install.sh | bash

# in lieu of restarting the shell
\. "$HOME/.nvm/nvm.sh"

# Download and install Node.js:
nvm install 24

# Verify the Node.js version:
node -v # Should print "v24.11.1".

# Download and install pnpm:
corepack enable pnpm

# Verify pnpm version:
pnpm -v
```

### Run development server

From the root of the project run:

```bash
pnpm run dev
```

The development server will be available on http://localhost:3000
