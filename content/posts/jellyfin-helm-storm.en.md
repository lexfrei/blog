---
title: "10 PRs or One Giant: Choosing an Approach for Refactoring Jellyfin Helm Chart"
date: 2025-11-05T18:30:00+03:00
draft: false
tags: ["helm", "kubernetes", "jellyfin", "open-source", "methodology"]
categories: ["tech"]
---

People often ask me "how do I make my first PR?", the answer is simple — just do it. But I want to write a series of notes about open source culture and how to make life convenient for everyone. Today we'll look at a case I encountered yesterday, namely — multiple changes to one semi-abandoned repo.

> **TL;DR**
>
> **When preparing a large refactor in an open source project:**
>
> 1. Break into small logical PRs — reduces cognitive load on reviewers
> 2. Propose integration branch for coordination — allows incremental review and batch changes
> 3. If in doubt, don't hesitate to reach out to the team early — openness and communication matter
> 4. Be ready for iterations — open source is about people
>
> This reduces load on reviewers and increases chances of successful merge.

## The Situation

Wanted to try Jellyfin for my home media server. Went to [jellyfin/jellyfin-helm](https://github.com/jellyfin/jellyfin-helm), looked at the chart, and realized—in its current state, it's not suitable for use. Plus my personal crusade against Ingress—need to add at least Gateway API support.

The Jellyfin team added the Helm chart by community request, but they simply don't have the resources and Helm expertise for active maintenance. This is a normal situation—you can't be an expert in everything.

Opened [issue #94](https://github.com/jellyfin/jellyfin-helm/issues/94) proposing comprehensive maintenance and listing what I wanted to improve.

## Five Courses of Action

### 1. Walk Away

Not an option. I want to use this, and the chart doesn't work for my needs in its current state.

### 2. Fork and Maintain Separately

Easy win—do whatever you want, publish your own chart. But even though I [publish my own charts](https://artifacthub.io/packages/search?user=lexfrei&sort=relevance&page=1), people first look at official ones, then at "community" ones. So the problems remain for the community.

### 3. One Giant PR

15 files changed, 500+ lines of code, breaking changes mixed with features.

**When this is appropriate:** Only when changes are so large that there's no point trying to push them through bit by bit. Plus if the team is technically strong and ready to review large changes. If there are automated tests covering critical parts. If changes are conceptually connected and hard to separate.

**Doesn't work for this repo:** They don't have resources to even review a [PR from May](https://github.com/jellyfin/jellyfin-helm/pull/67) (by the way, this is an antipattern, we'll discuss later). Team simply won't handle a giant diff. Reviewer looks at 500 lines and thinks: "What's even happening? Why this line? Is this related to that feature or this one?"

**Result:** Either "LGTM" without looking, or sits for six months without attention, or lengthy comment debates.

### 4. Ask to Become a Maintainer

Could immediately ask to join the team and make changes directly. But this requires trust that doesn't exist yet. Plus you still need a review process—can't just commit to master.

**Verdict:** Already proposed in the issue, but will bring it up again after proving myself.

By the way, this is useful and helps build reputation. Chart maintenance doesn't take too much effort if you set up CI, linting, etc.—just react to GitHub notifications and you're done. For charts like these, it's usually once every six months. I already maintain several community charts besides my own.

### 5. Many Small PRs

**This is what I chose.**

## Why 10 Separate PRs

Main reason: **reducing cognitive load on reviewers**.

### One giant PR:

```diff
# deployment.yaml
+ {{- if .Values.gateway.enabled }}
+ kind: HTTPRoute
...
+ {{- if .Values.persistence.cache.enabled }}
+ - name: cache
...
+ {{- if .Values.extraInitContainers }}
- {{- if .Values.initContainers }}
...
+ startupProbe:
+   httpGet:
...
```

Reviewer looks and thinks: "What's this block? Gateway API? Persistence? Init containers? Startup probe? How are they related? Can I remove this without breaking that?"

### Multiple separate PRs:

- **PR #86**: Gateway API support — all code about HTTPRoute, nothing else
- **PR #87**: IPv6 configuration — only IPv6, completely obvious
- **PR #88**: Cache volume — only about cache, clear boundary
- **PR #92**: Startup probe — one code block, one feature
- **PR #93**: Fix initContainers — clear problem, clear solution

Each PR contains one complete feature or one bugfix.

Reviewer opens PR #86, sees only Gateway API code. Everything clear: "OK, this adds HTTPRoute. Logic is correct, approve."

**No need to guess which piece of code does what.**

### Important: Don't Fragment Too Much

Don't make PRs for 2-3 tiny changes like:

- PR #1: "Added one variable"
- PR #2: "Added second variable"
- PR #3: "Used these variables"

That's overkill. One PR should be **one complete feature** or **one bugfix**.

You can make one huge PR like "I rewrote the world, here's what happened: 1, 2, 3, 4, 5... 128, 129", but then write a **detailed change list** in the description and be ready for lengthy review.

## Integration Branch: How It Works

But we're not going to release 10 chart versions just because I decided to tidy it up, right? This is where GitFlow approach comes to the rescue — integration branch.

Proposed this workflow to the Jellyfin team:

1. Maintainer creates `v3.0-integration` branch from master
2. I rebase all 10 PRs onto this branch
3. Team reviews **each PR separately**
4. All PRs merge into `v3.0-integration`, **not into master**
5. After merging all PRs, make **common batch changes** in integration branch:
   - Write tests (helm-unittest)
   - Prepare values.schema.json validation
   - Rewrite README accounting for all new features
   - Form unified changelog for v3.0.0
   - Add configuration examples
6. When everything's ready and tested, `v3.0-integration` merges into master as **v3.0.0**

### Why Is This Needed?

**Isolation:** Don't break master during development. Users stay on stable version, we experiment in integration branch.

**Incremental review:** Each PR is small and clear. Reviewer can review one per day without overload.

**Holistic testing:** After merging all PRs, can test `v3.0-integration` as a whole. If something doesn't work together—easy to find and fix.

**Rollback:** If a specific PR causes problems, easy to exclude or rework it.

**Transparency:** Whole team and community see progress. Can comment on each feature separately.

**Common changes:** Can make batch actions that would be awkward in separate PRs. Rewriting documentation, preparing changelog, writing tests—all this is logical when you see the whole picture, after merging all features.

### This Is Standard in Large Projects

Linux kernel, Kubernetes, and other large projects use this approach:

- `linux-next` in kernel
- `staging` branches in Kubernetes
- `integration` or `develop` branches in GitFlow

But for some reason it's rare in Helm charts. Too bad—the approach works great.

## Communication with the Team: Discuss Upfront

When I realized I'd already opened 6 PRs and wasn't planning to stop, I reached out to the team via Discord (this contact method is listed on the project website) and created [issue #94](https://github.com/jellyfin/jellyfin-helm/issues/94) to track the changes:

- What exactly I want to change
- Why it's needed
- How I propose to do it (integration branch)
- What breaking changes there will be

**Why this matters:**

**Shows respect:** Team understands you didn't just "dump some code" but thought through the approach.

**Get feedback:** Maybe the team will say "don't do that" or "let's do it this way instead". Better to learn BEFORE writing code.

**Avoid surprises:** Maintainers don't like when 10 PRs suddenly land without warning.

### How to Find the Team

**Standard entry point is an Issue** in the repository, if the repo is active and the team is responsive on GitHub.

But if you're in a situation like mine (repo semi-abandoned, PRs sitting for months without review), you can try:

**1. Go through the main project repository:**

- Jellyfin Helm chart is a separate repo
- Main project — [jellyfin/jellyfin](https://github.com/jellyfin/jellyfin)
- There might be more activity and links to communication channels

**2. Look for contact methods outside GitHub:**

**Where to look:**

- Official project website (usually has Community/Contact section)
- Main repository README
- `CONTRIBUTING.md` file
- Organization profile on GitHub
- Previous issues/PRs (see where important discussions happen)

**Popular channels:**

- **Discord** — most modern open source projects
- **Matrix** — FOSS-friendly Discord alternative
- **Slack** — corporate and enterprise projects
- **IRC** — older projects (Linux kernel, many GNU utilities)
- **Mailing lists** — academic and large projects (LKML, Apache)

**In my case:**

- Jellyfin website lists Discord
- I wrote there, explained the situation
- Got quick response from maintainer Cody and approach approval

**Important:** Even if the team isn't very active in chat, the mere fact that you **attempted to reach out** shows good intentions. The team will appreciate the proactivity.

## Lessons Learned

### 1. Small PRs Reduce Cognitive Load

Reviewer shouldn't hold entire change context in their head. One PR = one feature = clear boundary.

### 2. Integration Branch Provides Safety

Can experiment without fear of breaking production. Users stay on stable version.

### 3. Openness and Communication Matter

I didn't just throw PRs at them—I opened an issue, explained the plan, proposed a workflow. Team appreciated and supported it.

### 4. Open Source Is About People

Jellyfin team honestly said: "We don't have resources for Helm chart, but we welcome help." That's normal. Projects live thanks to community.

## What's Next

Phase 1: integration branch, review, merge 10 PRs.

Phase 2 (after Phase 1): tests (helm-unittest), CI workflows, values.schema.json, documentation, configuration examples.

If everything goes well—possibly official maintainership.

---

**P.S.** If you're preparing a large refactor in an open source project:

1. Break into small logical PRs
2. Propose integration branch for coordination
3. Explain the plan to the team upfront
4. Be ready for iterations

This reduces load on reviewers and increases chances of successful merge.
