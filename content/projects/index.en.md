---
title: "Projects"
layout: "single"
url: "/en/projects/"
summary: "Open source projects I develop and maintain"
---

Open source projects I develop and maintain.

## Kubernetes Controllers

### cloudflare-tunnel-gateway-controller

Gateway API implementation over Cloudflare Tunnels. Manage tunnels via standard Kubernetes resources (HTTPRoute, GRPCRoute).

- [GitHub](https://github.com/lexfrei/cloudflare-tunnel-gateway-controller)
- [Documentation](https://cf.k8s.lex.la/latest/)

## Helm Charts

### vipalived

VIP for Kubernetes API via VRRP/keepalived. Works without kube-api dependency — solves the chicken-and-egg problem for CNI.

- [ArtifactHub](https://artifacthub.io/packages/helm/vipalived/vipalived)
- [GitHub](https://github.com/lexfrei/helm-charts)

### prometheus-ipmi-exporter

Chart for IPMI Exporter. Primary maintainer in prometheus-community.

- [ArtifactHub](https://artifacthub.io/packages/helm/prometheus-community/prometheus-ipmi-exporter)

### system-upgrade-controller

Chart for Rancher System Upgrade Controller.

- [ArtifactHub](https://artifacthub.io/packages/helm/system-upgrade/system-upgrade-controller)

### Other charts

- [cloudflare-tunnel](https://artifacthub.io/packages/helm/cloudflare-tunnel/cloudflare-tunnel)
- [transmission](https://artifacthub.io/packages/helm/transmission/transmission)
- [papermc](https://artifacthub.io/packages/helm/minecraft/papermc)

## Go Libraries

### go-unifi

Library for UniFi API. Reverse-engineered API, covers main endpoints.

- [GitHub](https://github.com/lexfrei/go-unifi)

### go-transmission

Transmission RPC API client. Generated from official spec, 100% API coverage.

- [GitHub](https://github.com/lexfrei/go-transmission)

### goPaperMC

Library and CLI for PaperMC API.

- [GitHub](https://github.com/lexfrei/goPaperMC)

### go-hangar

Library for Hangar API (Minecraft plugin repository).

- [GitHub](https://github.com/lexfrei/go-hangar)

## Tools & Bots

### external-dns-unifios-webhook

Webhook for external-dns with UniFi OS support. DNS record management on UniFi controller.

- [GitHub](https://github.com/lexfrei/external-dns-unifios-webhook)
- [Documentation](https://unifi-edns.lex.la/)

### transmission-bot

Telegram bot for Transmission management. Add torrents, check status, manage downloads.

- [GitHub](https://github.com/lexfrei/transmission-bot)

## Other

### papermc-docker

Rootless Docker image for PaperMC. Daily automated builds.

- [GitHub](https://github.com/lexfrei/papermc-docker)
- [ArtifactHub](https://artifacthub.io/packages/helm/minecraft/papermc)
