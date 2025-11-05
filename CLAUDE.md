# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Overview

This is a Hugo-based multilingual blog (English/Russian) deployed to GitHub Pages at `blog.lex.la`. Uses PaperMod theme with custom modifications for comments, analytics, and language detection.

## Development Commands

### Local Development
```bash
# Start local server with drafts
hugo server --buildDrafts

# Build site
hugo --gc --minify

# Create new post (use language suffix)
hugo new content posts/my-post.en.md
hugo new content posts/my-post.ru.md
```

### Testing
```bash
# Check for broken links
hugo --gc --minify && echo "Build successful"
```

## Architecture

### Multilingual Setup
- **Structure**: Suffix-based (`.en.md`, `.ru.md`)
- **URLs**: `/en/` for English, `/ru/` for Russian
- **Root redirect**: `layouts/index.html` detects browser language and redirects
- **Config**: `defaultContentLanguageInSubdir = true` forces both languages into subdirectories

### Custom Overrides
Located in `layouts/partials/`:
- `footer.html` - Removes "Powered by Hugo & PaperMod" attribution
- `comments.html` - Integrates Giscus comments
- `giscus.html` - Dynamic language support via `{{ .Site.Language.Lang }}`
- `extend_head.html` - Cloudflare Web Analytics integration

### Post Front Matter
```yaml
---
title: "Post Title"
date: 2024-11-04T23:00:00+03:00
draft: false
tags: ["kubernetes", "helm"]
categories: ["tech"]
---
```

### Deployment
- **Trigger**: Push to `master` branch
- **Process**: GitHub Actions builds with Hugo 0.152.2 extended, deploys to GitHub Pages
- **Domain**: Custom domain configured via `static/CNAME` and GitHub Pages API
- **Theme**: PaperMod as git submodule (`themes/PaperMod`)

### Key Configuration Points
- **baseURL**: `https://blog.lex.la/` (must match GitHub Pages custom domain)
- **Comments**: Giscus using GitHub Discussions (repo: lexfrei/blog)
- **Analytics**: Cloudflare Web Analytics token in `extend_head.html`
- **Social links**: Configured in `hugo.toml` under `params.socialIcons`

## Translation Guidelines

When translating content between Russian and English, follow the **Tolkien translation philosophy**:

- **Preserve style and voice** over literal word-for-word translation
- **Adapt context-specific references**: Local memes, cultural references, and idioms should be adapted for the target audience, not directly translated
- **Examples of adaptation**:
  - Russian slang → English equivalent expressions
  - Culture-specific humor → Universally understandable alternatives
  - Local references → International context

### Content Moderation

**Critical**: Any content that could damage professional reputation must be removed immediately:
- Discriminatory jokes or references
- Potentially offensive comparisons
- Controversial political statements in technical context

**When removing such content**:
1. Remove it first (do not wait for approval)
2. **Explicitly inform the user** about what was removed and why
3. Suggest alternative phrasings if appropriate

The blog represents professional identity in the global tech community. Err on the side of caution.

## Important Notes

- Both language versions of a post must use the same filename stem (e.g., `october.en.md` and `october.ru.md`)
- The root `/` page redirects based on `navigator.language` (Russian → `/ru/`, others → `/en/`)
- Theme customizations in `layouts/` override theme defaults without modifying theme files
- Submodules must be checked out recursively for theme to work
