---
title: "Проекты"
layout: "single"
url: "/ru/projects/"
summary: "Open source проекты, которые я разрабатываю и поддерживаю"
---

Open source проекты, которые я разрабатываю и поддерживаю.

## Kubernetes Controllers

### cloudflare-tunnel-gateway-controller

Реализация Gateway API поверх Cloudflare Tunnels. Позволяет управлять туннелями через стандартные ресурсы Kubernetes (HTTPRoute, GRPCRoute).

- [GitHub](https://github.com/lexfrei/cloudflare-tunnel-gateway-controller)
- [Документация](https://cf.k8s.lex.la/latest/)


## Helm Charts

### vipalived

VIP для Kubernetes API через VRRP/keepalived. Работает без зависимости от kube-api — решает проблему курицы и яйца для CNI.

- [ArtifactHub](https://artifacthub.io/packages/helm/vipalived/vipalived)
- [GitHub](https://github.com/lexfrei/helm-charts)

### prometheus-ipmi-exporter

Чарт для IPMI Exporter. Основной мейнтейнер в prometheus-community.

- [ArtifactHub](https://artifacthub.io/packages/helm/prometheus-community/prometheus-ipmi-exporter)

### system-upgrade-controller

Чарт для Rancher System Upgrade Controller.

- [ArtifactHub](https://artifacthub.io/packages/helm/system-upgrade/system-upgrade-controller)

### Другие чарты

- [cloudflare-tunnel](https://artifacthub.io/packages/helm/cloudflare-tunnel/cloudflare-tunnel)
- [transmission](https://artifacthub.io/packages/helm/transmission/transmission)
- [papermc](https://artifacthub.io/packages/helm/minecraft/papermc)

## Go Libraries

### go-unifi

Библиотека для работы с UniFi API. Реверс-инженерный API, покрывает основные эндпоинты.

- [GitHub](https://github.com/lexfrei/go-unifi)

### go-transmission

Клиент для Transmission RPC API. Генерируется из официальной спеки, 100% покрытие API.

- [GitHub](https://github.com/lexfrei/go-transmission)

### goPaperMC

Библиотека и CLI для работы с PaperMC API.

- [GitHub](https://github.com/lexfrei/goPaperMC)

### go-hangar

Библиотека для Hangar API (репозиторий плагинов Minecraft).

- [GitHub](https://github.com/lexfrei/go-hangar)

## Tools & Bots

### external-dns-unifios-webhook

Webhook для external-dns с поддержкой UniFi OS. Управление DNS-записями на UniFi контроллере.

- [GitHub](https://github.com/lexfrei/external-dns-unifios-webhook)
- [Документация](https://unifi-edns.lex.la/)

### transmission-bot

Telegram-бот для управления Transmission. Добавление торрентов, просмотр статуса, управление загрузками.

- [GitHub](https://github.com/lexfrei/transmission-bot)

## Other

### papermc-docker

Rootless Docker-образ для PaperMC. Ежедневные автосборки.

- [GitHub](https://github.com/lexfrei/papermc-docker)
- [ArtifactHub](https://artifacthub.io/packages/helm/minecraft/papermc)
