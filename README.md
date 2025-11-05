# Personal Blog

Hugo-based bilingual blog deployed to GitHub Pages at [blog.lex.la](https://blog.lex.la).

## Tech Stack

### Static Site Generator: Hugo

**Why Hugo:**
- Go-based (familiar technology)
- Go templates (already know the syntax)
- Fast build times (~50ms for 70 pages)
- Content-first approach (pure Markdown)
- Excellent multilingual support out of the box
- Large ecosystem of themes and plugins

**Version:** 0.152.2 extended (locked in GitHub Actions)

### Theme: PaperMod

**Why PaperMod:**
- Clean, minimal design
- Built-in dark mode
- Mobile-friendly responsive layout
- Good i18n support
- Active maintenance
- Easy to customize via layouts overrides

**Installation:** Git submodule in `themes/PaperMod`

### Comments: Giscus

**Why Giscus:**
- GitHub Discussions-based (no external service)
- Respects user privacy
- Supports multiple languages dynamically
- Free and open source
- Easy moderation through GitHub

**Integration:** Custom partial in `layouts/partials/giscus.html` with dynamic language support (`{{ .Site.Language.Lang }}`)

### Analytics: Cloudflare Web Analytics

**Why Cloudflare:**
- Privacy-friendly (no cookies, GDPR compliant)
- Lightweight (< 5KB script)
- Real-time insights
- Free tier is sufficient
- No impact on SEO

**Integration:** `layouts/partials/extend_head.html`

### Hosting: GitHub Pages

**Why GitHub Pages:**
- Free hosting for static sites
- Custom domain support with SSL
- Direct integration with GitHub Actions
- Fast CDN
- Simple deployment workflow

**Custom domain:** `blog.lex.la` (configured via `static/CNAME`)

## Project Structure

```
.
├── content/
│   └── posts/
│       ├── *.en.md          # English posts
│       └── *.ru.md          # Russian posts
├── layouts/
│   ├── alias.html           # Root redirect with browser language detection
│   └── partials/
│       ├── comments.html    # Giscus comments integration
│       ├── footer.html      # Custom footer (removes attribution)
│       ├── giscus.html      # Giscus config with dynamic language
│       └── extend_head.html # Cloudflare Analytics script
├── static/
│   └── CNAME                # Custom domain for GitHub Pages
├── themes/
│   └── PaperMod/            # Theme as git submodule
├── .github/
│   └── workflows/
│       └── hugo.yml         # Automated deployment
├── hugo.toml                # Main configuration
├── CLAUDE.md                # Instructions for Claude Code
└── README.md                # This file
```

## Multilingual Setup

### Configuration

**Strategy:** Suffix-based file naming (`.en.md`, `.ru.md`)

**Key config in `hugo.toml`:**
```toml
defaultContentLanguage = 'en'
defaultContentLanguageInSubdir = true  # Forces /en/ and /ru/ subdirectories
```

### Language Detection

**Root redirect (`layouts/alias.html`):**
- Detects browser language via `navigator.language`
- Russian browsers (`ru_*`) → `/ru/`
- All others → `/en/`
- Fallback via `<noscript>` meta refresh

**Why custom `alias.html`:**
- Hugo's built-in alias mechanism only does simple redirects
- Need JavaScript for browser language detection
- Static hosting (GitHub Pages) has no server-side redirect capability
- This approach keeps `/en/` and `/ru/` as full pages (no redirect loop)

### URL Structure

```
/           → Redirects based on browser language
/en/        → English homepage
/en/posts/  → English posts
/ru/        → Russian homepage
/ru/posts/  → Russian posts
```

## Development

### Prerequisites

```bash
brew install hugo
```

### Local Development

```bash
# Start development server with drafts
hugo server --buildDrafts

# Access at http://localhost:1313/
```

### Creating Posts

```bash
# Create bilingual posts (use same filename stem)
hugo new content posts/my-post.en.md
hugo new content posts/my-post.ru.md
```

**Front matter template:**
```yaml
---
title: "Post Title"
date: 2024-11-04T23:00:00+03:00
draft: false
tags: ["kubernetes", "helm", "open-source"]
categories: ["tech"]
---
```

### Building

```bash
# Production build
hugo --gc --minify

# Output: ./public/
```

## Deployment

### Automated via GitHub Actions

**Trigger:** Push to `master` branch

**Workflow (`.github/workflows/hugo.yml`):**
1. Checkout repo with submodules (theme)
2. Setup Hugo 0.152.2 extended
3. Build site (`hugo --gc --minify`)
4. Upload artifact to GitHub Pages
5. Deploy to production

**Deploy time:** ~50 seconds

### Manual Deployment

Not recommended, but possible:
```bash
# Build locally
hugo --gc --minify

# Push public/ to gh-pages branch (not used, we use Actions)
```

## Customizations

### Theme Overrides

All customizations are in `layouts/` without modifying theme files:

1. **`layouts/alias.html`** - Root redirect with language detection
2. **`layouts/partials/footer.html`** - Removes "Powered by Hugo & PaperMod"
3. **`layouts/partials/giscus.html`** - Giscus comments config
4. **`layouts/partials/comments.html`** - Integrates Giscus into posts
5. **`layouts/partials/extend_head.html`** - Cloudflare Analytics

### Theme Updates

```bash
cd themes/PaperMod
git pull origin master
cd ../..
git add themes/PaperMod
git commit -m "chore: update PaperMod theme"
```

## Key Decisions

### Why suffix-based files instead of directory structure?

**Chosen:** `posts/october.en.md` + `posts/october.ru.md`

**Rejected:** `posts/en/october.md` + `posts/ru/october.md`

**Reason:** Easier to manage translations side-by-side, simpler file organization

### Why custom alias.html instead of static/index.html?

**Reason:** Hugo overwrites files in `static/` when there's a conflict with generated content. Using `layouts/alias.html` is the official Hugo way to customize alias redirects.

### Why browser language detection instead of subdomain/path selection?

**Reason:** Better UX for first-time visitors. Users can still directly access `/en/` or `/ru/` if needed. Subsequent visits use direct links.

### Why remove "Powered by Hugo & PaperMod"?

**Reason:** MIT license allows removal, cleaner footer. Still documented in README.

## Common Issues

### Infinite redirect loop

**Symptom:** `/en/` redirects to `/en/` forever

**Cause:** Redirect script applies to all pages, not just root

**Fix:** Use `layouts/alias.html` with pathname check:
```javascript
if (window.location.pathname !== '/') {
    window.location.replace('{{ .Permalink }}');
    return;
}
```

### Giscus 404 on login

**Symptom:** GitHub OAuth returns 404

**Cause:** Usually caused by redirect loop breaking OAuth callback

**Fix:** Resolve redirect loop first

### Theme not found

**Symptom:** `Error: module "PaperMod" not found`

**Cause:** Submodule not initialized

**Fix:**
```bash
git submodule update --init --recursive
```

## License

**Blog content:** © 2025 Aleksei Sviridkin, All Rights Reserved

**Theme (PaperMod):** MIT License

**Hugo:** Apache License 2.0
