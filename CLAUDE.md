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
- **Root redirect**: `layouts/alias.html` detects browser language and redirects
- **Config**: `defaultContentLanguageInSubdir = true` forces both languages into subdirectories

**IMPORTANT**: Use `layouts/alias.html`, NOT `layouts/index.html` for root redirect:
- `layouts/index.html` applies to ALL homepage pages (`/`, `/en/`, `/ru/`), causing infinite redirect loops
- `layouts/alias.html` only applies to alias redirects (like root `/` when `defaultContentLanguageInSubdir = true`)
- This is the official Hugo way to customize alias behavior without breaking language homepages

### Custom Overrides

**Root level:**
- `layouts/alias.html` - Browser language detection for root `/` redirect (JavaScript-based)

**Partials in `layouts/partials/`:**
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

## Common Issues and Solutions

### Infinite Redirect Loop

**Symptom**: Opening `/en/` or `/ru/` causes endless redirects (browser shows "too many redirects" error)

**Root Cause**: Using `layouts/index.html` for language detection. Hugo applies this template to ALL homepage pages (`/`, `/en/`, `/ru/`), causing the redirect script to execute on language-specific pages.

**Solution**: Use `layouts/alias.html` instead:
```html
<script>
    (function() {
        // CRITICAL: Only redirect at root
        if (window.location.pathname !== '/') {
            window.location.replace('{{ .Permalink }}');
            return;
        }
        // Language detection logic here...
    })();
</script>
```

**Why it works**: Hugo only applies `alias.html` to alias redirects (like root `/` when `defaultContentLanguageInSubdir = true`), not to actual content pages.

### Giscus 404 After Login

**Symptom**: Clicking "Sign in with GitHub" in Giscus comments returns 404 error page

**Root Cause**: Usually caused by infinite redirect loop. GitHub OAuth callback URL gets caught in redirect cycle.

**Solution**: Fix the redirect loop first (see above). Once `/en/` and `/ru/` load without redirecting, Giscus OAuth will work.

### Hugo Build Artifacts in Git

**Symptom**: `.hugo_build.lock` file appears in git status

**Solution**: Already added to `.gitignore`. This is Hugo's temporary lock file and should never be committed.

### Theme Not Loading

**Symptom**: Site builds but shows no styling, "theme not found" errors

**Root Cause**: PaperMod theme submodule not initialized

**Solution**:
```bash
git submodule update --init --recursive
```

## Technical Notes

### Build Process

**Always clean build before testing redirect changes**:
```bash
rm -rf public && hugo --gc --minify
```

Hugo may cache old builds in `public/`, leading to false test results.

### File Locations

- **Generated site**: `public/` (gitignored, never commit)
- **Build lock**: `.hugo_build.lock` (gitignored, temporary)
- **Static assets**: `static/` (copied as-is to `public/`)
- **Theme**: `themes/PaperMod/` (git submodule)

### Testing Language Detection

1. Build site: `hugo --gc --minify`
2. Check root redirect: `cat public/index.html` (should contain browser detection JS)
3. Check language pages: `grep -c 'window.location.replace' public/en/index.html` (should return 0)
4. Test in browser with different language settings

### Deployment

GitHub Actions automatically deploys on push to `master`. Check deployment status:
```bash
gh run list --limit 1
```

Typical deploy time: ~50 seconds

## Additional Documentation

For detailed technical information, architecture decisions, and troubleshooting:
- See `README.md` for comprehensive technical documentation
- See `hugo.toml` for all configuration options
- See `.github/workflows/hugo.yml` for deployment workflow

## Important Notes

- Both language versions of a post must use the same filename stem (e.g., `october.en.md` and `october.ru.md`)
- The root `/` page redirects based on `navigator.language` (Russian → `/ru/`, others → `/en/`)
- Theme customizations in `layouts/` override theme defaults without modifying theme files
- Submodules must be checked out recursively for theme to work
- **Never** use `layouts/index.html` for multilingual sites with `defaultContentLanguageInSubdir = true`
