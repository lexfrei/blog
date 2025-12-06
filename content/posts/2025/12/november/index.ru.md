---
title: "Ноябрь: железо, библиотеки и контроллеры"
date: 2025-12-06T12:00:00+03:00
slug: "november"
draft: false
tags: ["kubernetes", "golang", "open-source", "homelab", "monthly-recap"]
categories: ["tech"]
---

Новотрадиционная рубрика "Как я провёл прошлый месяц"!

По большей части, фокус был направлен на себя и на закрытие проблем, которые накопились.

## Миграция на ML310

Продолжаю миграцию с MicroServer на HP ML310e.

Столкнулся с горой проприетарных проблем вроде "вам нужен вентилятор именно от нас и за $150". Правда, всё это оказалось довольно легко обмануть — пины 4-5-6 замыкаются на землю и все ошибки уходят.

Половил Illegal OpCode ошибок с HBA-контроллером и сетевушкой, но вылечилось отключением Optional ROM в BIOS. Это не универсальный способ, но бутаюсь я всё равно с флешки, а не дисков или сети, так что сойдёт.

Отдельная боль была для меня в том, что 2 розетки для него просто жалко отдавать. Но, оказывается, существуют разветвители C13 → 2× C14.

В общем, все ошибки погашены, iLO прекратил мигать мне жёлтым. Осталось перешить DAC и поднять 10G сеть между рутером и хранилищем.

Не могу сказать, что прям доволен миграцией — ML310 значительно больше, шумнее, а из реальных плюсов только большее количество PCI. Я ввязывался в это с мыслью, что это будет дешевле, чем поменять БП в микросервере, но уже начинаю сомневаться.

## Свой webhook для external-dns

Сначала я нашёл уже готовый [external-dns-unifi-webhook](https://github.com/kashalls/external-dns-unifi-webhook). Принёс туда десяток PR по улучшениям, но в итоге понял, что переписывать чужой проект PR'ами слишком муторно.

Так родились [external-dns-unifios-webhook](https://github.com/lexfrei/external-dns-unifios-webhook) и лежащая под ним библиотека [go-unifi](https://github.com/lexfrei/go-unifi).

Просто работает, просто эффективно. С библиотекой интересно — API у UniFi не документирован от слова совсем. Я отреверсил нужные мне части, но вообще было бы интересно отреверсить вообще всё. Правда, пока не придумал как отреверсить 300 эндпоинтов (домножаем на методы) не деструктивно, да так, чтоб контекст у нейронки не кончался.

## Transmission API на Go

Просто потому что NIH были накиданы [go-transmission](https://github.com/lexfrei/go-transmission) и [transmission-bot](https://github.com/lexfrei/transmission-bot).

Библиотека генерируется из RPC-спеки первоисточника и покрывает 100% API. Бот для Telegram позволяет легко закидывать торренты на закачку, смотреть список со статусами и управлять загрузками.

Удивился, что кодогенерации по RPC-спеке на Go по факту нет. Но чем нейронка не кодогенерация?

## Gateway API поверх Cloudflare Tunnels

Наверное, самое интересное и объёмное за месяц.

Началось как шутка, но быстро стало понятно, что это реально полезная штука. Проблема простая: Cloudflare Tunnel — это круто для expose сервисов наружу, но конфигурируется он либо через их dashboard, либо через ConfigMap с кастомным форматом. А я хочу использовать Gateway API — стандартный способ описания маршрутизации в Kubernetes.

Так появился [cloudflare-tunnel-gateway-controller](https://github.com/lexfrei/cloudflare-tunnel-gateway-controller). Контроллер следит за ресурсами Gateway API (HTTPRoute, GRPCRoute) и автоматически конфигурирует туннель. Поддерживается hot reload — обновление маршрутов не требует рестарта cloudflared.

Что поддерживается:

- GatewayClass и Gateway для определения туннеля
- HTTPRoute и GRPCRoute для маршрутизации
- ReferenceGrant для кросс-неймспейсных маршрутов
- Leader election для HA

Бонус: есть заготовка под VPN через sidecar, уже протестировано с AmneziaWG. Актуально для тех, кому ТСПУ и GFW делают больно.

В общем, теперь у меня единообразный подход к маршрутизации: что Cilium для внутреннего трафика, что Cloudflare для внешнего — всё через Gateway API.

## Image Index в Helm

Предложил в Helm научиться использовать Image Index: [helm/community#424](https://github.com/helm/community/pull/424). Предложив попутно реализацию: [helm/helm#31583](https://github.com/helm/helm/pull/31583).

Я с удивлением обнаружил, что Helm много чего не умеет в смысле публикации. Например, он не поддерживает Image Index, что не позволяет одним манифестом опубликовать образы + чарт + SBOM + подписи.

В целом, фича straightforward, но ребята решили прогнать меня через proposal-механизм.

На мой взгляд это очень важная фича, которая позволит атомарно выпускать все артефакты, связанные с релизом.

## Обновление SSD на Raspberry Pi

Столкнулся с интересной проблемой обновления SSD Samsung — только через официальную утилиту. На ARM её нет.

Пришлось вытащить прошивку и ключ из обновлятора. [Детали тут](https://gist.github.com/lexfrei/3e8dc9e97608502cf226371f6badb1e2).

## PR'ы одной строкой

- Починил в Renovate обработку OCI-чартов для Argo CD: [renovatebot/renovate#39149](https://github.com/renovatebot/renovate/pull/39149)
- Добавил в Grafana Operator поддержку HTTPRoute: [grafana/grafana-operator#2293](https://github.com/grafana/grafana-operator/pull/2293) (PR не от моего имени, но [исходный](https://github.com/grafana/grafana-operator/pull/2283) был слит с ещё одним и я прохожу по делу как соучастник)
- Сунулся в Haskell, пришлось потрогать rules_haskell для Bazel: [tweag/rules_haskell#2359](https://github.com/tweag/rules_haskell/pull/2359)
- Предложил в Argo Workflows добавить HTTPRoute: [argoproj/argo-helm#3567](https://github.com/argoproj/argo-helm/pull/3567) — но тут почему-то тишина
- Предложил в k3s-ansible исправление бага: [k3s-io/k3s-ansible#474](https://github.com/k3s-io/k3s-ansible/pull/474)

## Проблемы, с которыми столкнулся

- Longhorn [не контролирует](https://github.com/longhorn/longhorn/issues/12294) здоровье реплик на самом деле
- Cilium [не умеет](https://github.com/cilium/cilium/issues/43130) нормально стартовать Gateway API reconciliation loop
- AmneziaWG [криво работает](https://github.com/amnezia-vpn/amnezia-client/issues/2022) с DNS на маке
- External-dns [подтекает памятью](https://github.com/kubernetes-sigs/external-dns/issues/5965), ИНОГДА
- Linux kernel [теряет интерфейсы](https://bugs.launchpad.net/ubuntu/+source/linux-raspi/+bug/2133877) и складывает сеть, ИНОГДА

## Полезные находки

- **[ttl.sh](https://ttl.sh)** — OCI-реджистри с TTL, отлично подходит для сборки всяких временных образов, типа PR'ных
- **[MkDocs](https://www.mkdocs.org/)** — проще, чем Hugo, для документации. Удобно для работы из GitHub Actions. Уже две доки опубликованы: [unifi-edns.lex.la](https://unifi-edns.lex.la/) и [cf.k8s.lex.la](https://cf.k8s.lex.la/latest/)
