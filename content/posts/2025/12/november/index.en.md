---
title: "November: Hardware, Libraries, and Controllers"
date: 2025-12-06T12:00:00+03:00
slug: "november"
draft: false
tags: ["kubernetes", "golang", "open-source", "homelab", "monthly-recap"]
categories: ["tech"]
---

Welcome to my new tradition: monthly recaps!

This month was mostly about tackling personal backlog and closing out accumulated issues.

## ML310 Migration

Continuing my migration from MicroServer to HP ML310e.

Ran into a wall of proprietary nonsense like "you need OUR specific fan for $150". Turns out it's easy to fool — just short pins 4-5-6 to ground and all errors disappear.

Caught some Illegal OpCode errors with the HBA controller and NIC, but disabling Optional ROM in BIOS fixed it. Not a universal solution, but I boot from USB anyway, not disks or network, so it works for me.

Another pain point: dedicating 2 power outlets to this thing felt wasteful. But apparently C13 → 2× C14 splitters exist.

All errors cleared, iLO stopped flashing yellow at me. Still need to flash the DAC and set up 10G networking between router and storage.

Can't say I'm thrilled with the migration — ML310 is significantly larger, louder, and the only real benefit is more PCI slots. I got into this thinking it would be cheaper than replacing the PSU in the MicroServer, but I'm starting to have doubts.

## Custom external-dns Webhook

First I found an existing [external-dns-unifi-webhook](https://github.com/kashalls/external-dns-unifi-webhook). Contributed about a dozen PRs with improvements, but eventually realized that rewriting someone else's project through PRs is too tedious.

Thus were born [external-dns-unifios-webhook](https://github.com/lexfrei/external-dns-unifios-webhook) and the underlying library [go-unifi](https://github.com/lexfrei/go-unifi).

Simple and effective. The library part is interesting — UniFi's API is completely undocumented. I reverse-engineered the parts I needed, but it would be fun to reverse-engineer everything. Haven't figured out how to non-destructively reverse 300 endpoints (multiply by HTTP methods) without running out of LLM context though.

## Transmission API in Go

Pure NIH syndrome produced [go-transmission](https://github.com/lexfrei/go-transmission) and [transmission-bot](https://github.com/lexfrei/transmission-bot).

The library is generated from the official RPC spec and covers 100% of the API. The Telegram bot makes it easy to add torrents, check download status, and manage transfers.

Surprised there's no proper code generation from RPC spec for Go. But hey, what's an LLM if not a code generator?

## Gateway API over Cloudflare Tunnels

Probably the most interesting and substantial work this month.

Started as a joke, but quickly turned into something genuinely useful. The problem is simple: Cloudflare Tunnel is great for exposing services, but configuration is either through their dashboard or a ConfigMap with a custom format. I wanted to use Gateway API — the standard way to describe routing in Kubernetes.

Enter [cloudflare-tunnel-gateway-controller](https://github.com/lexfrei/cloudflare-tunnel-gateway-controller). The controller watches Gateway API resources (HTTPRoute, GRPCRoute) and automatically configures the tunnel. Hot reload is supported — route updates don't require cloudflared restarts.

What's supported:

- GatewayClass and Gateway for tunnel definition
- HTTPRoute and GRPCRoute for routing
- ReferenceGrant for cross-namespace routes
- Leader election for HA

Bonus: there's a VPN sidecar option, already tested with AmneziaWG. Useful if you're dealing with DPI-based censorship.

Now I have a unified approach to routing: Cilium for internal traffic, Cloudflare for external — all through Gateway API.

## Image Index in Helm

Proposed teaching Helm to use Image Index: [helm/community#424](https://github.com/helm/community/pull/424). Also submitted an implementation: [helm/helm#31583](https://github.com/helm/helm/pull/31583).

I was surprised to discover Helm lacks many publishing capabilities. For example, it doesn't support Image Index, which means you can't publish images + chart + SBOM + signatures in a single manifest.

The feature itself is straightforward, but the team decided to put me through their proposal process.

In my view, this is a critical feature that would enable atomic release of all artifacts associated with a release.

## Raspberry Pi SSD Upgrade

Hit an interesting problem: Samsung SSD firmware updates only work through their official utility. Which doesn't exist for ARM.

Had to extract the firmware and key from the updater. [Details here](https://gist.github.com/lexfrei/3e8dc9e97608502cf226371f6badb1e2).

## PRs in Brief

- Fixed OCI chart handling for Argo CD in Renovate: [renovatebot/renovate#39149](https://github.com/renovatebot/renovate/pull/39149)
- Added HTTPRoute support to Grafana Operator: [grafana/grafana-operator#2293](https://github.com/grafana/grafana-operator/pull/2293) (PR under different name, but [my original](https://github.com/grafana/grafana-operator/pull/2283) was merged with another — I'm listed as an accomplice)
- Ventured into Haskell, had to touch rules_haskell for Bazel: [tweag/rules_haskell#2359](https://github.com/tweag/rules_haskell/pull/2359)
- Proposed HTTPRoute for Argo Workflows: [argoproj/argo-helm#3567](https://github.com/argoproj/argo-helm/pull/3567) — radio silence so far
- Proposed a bug fix for k3s-ansible: [k3s-io/k3s-ansible#474](https://github.com/k3s-io/k3s-ansible/pull/474)

## Issues I Ran Into

- Longhorn [doesn't actually control](https://github.com/longhorn/longhorn/issues/12294) replica health
- Cilium [can't properly start](https://github.com/cilium/cilium/issues/43130) Gateway API reconciliation loop
- AmneziaWG [has DNS issues](https://github.com/amnezia-vpn/amnezia-client/issues/2022) on macOS
- External-dns [leaks memory](https://github.com/kubernetes-sigs/external-dns/issues/5965), SOMETIMES
- Linux kernel [loses interfaces](https://bugs.launchpad.net/ubuntu/+source/linux-raspi/+bug/2133877) and kills networking, SOMETIMES

## Useful Discoveries

- **[ttl.sh](https://ttl.sh)** — OCI registry with TTL, perfect for temporary images like PR builds
- **[MkDocs](https://www.mkdocs.org/)** — simpler than Hugo for documentation. Works great with GitHub Actions. Already published two docs sites: [unifi-edns.lex.la](https://unifi-edns.lex.la/) and [cf.k8s.lex.la](https://cf.k8s.lex.la/latest/)
