---
title: "Публичный HTTPS для домашнего кластера без единого входящего порта"
date: 2026-09-21T12:00:00+03:00
slug: "zero-inbound-ports"
draft: true
tags: ["kubernetes", "gateway-api", "cloudflare", "external-dns", "self-hosting"]
categories: ["tech"]
description: "Один HTTPRoute, один исходящий туннель, ни одного проброшенного порта. Как выставить сервис из домашнего Kubernetes в интернет через Gateway API и что при этом ломается."
---

Домашний кластер живёт за NAT провайдера, публичного IP у него нет, и открывать входящие порты в него я не хочу даже теоретически. При этом сервисы из него нужны снаружи: мне, друзьям, паре ботов. Задача звучит так: написать один HTTPRoute, сделать `kubectl apply`, и через минуту `https://app.example.com` отвечает. Без балансировщика, без публичного адреса, без правила на файрволе.

Этот текст — про то, как это собрать на Gateway API поверх Cloudflare Tunnel, с автоматическим DNS через external-dns. И про то, что по дороге ломается, потому что первую версию этой схемы я собирал [в марте 2024-го](https://t.me/kubealexis/53) руками, и с тех пор набил достаточно шишек, чтобы написать под неё контроллер.

## Что получится в конце

Один манифест:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
  annotations:
    external-dns.kubernetes.io/cloudflare-proxied: "true"
spec:
  parentRefs:
    - name: cloudflare-tunnel
      namespace: cloudflare-tunnel-system
  hostnames:
    - app.example.com
  rules:
    - backendRefs:
        - name: my-service
          port: 80
```

После `kubectl apply` контроллер добавляет маршрут в туннель, external-dns заводит CNAME в Cloudflare, и сервис доступен из интернета по HTTPS. Обратно к кластеру идёт ровно одно соединение, исходящее: туннель сам держит его с edge Cloudflare.

```text
$ kubectl get gateway --namespace cloudflare-tunnel-system
NAME                CLASS               ADDRESS                          PROGRAMMED   AGE
cloudflare-tunnel   cloudflare-tunnel   2001-db8.cfargotunnel.com        True         3m
```

Дальше по шагам.

## Откуда это взялось

Небольшое отступление, потому что оно объясняет, почему инструмент устроен так, а не иначе.

В марте 2024-го я открыл PR в официальный Helm-чарт Cloudflare ([cloudflare/helm-charts#69](https://github.com/cloudflare/helm-charts/pull/69)): опцию отключить дефолтную 404-ю. Ответа нет до сих пор. Так я стал мейнтейнером [собственного чарта](https://artifacthub.io/packages/helm/cloudflare-tunnel/cloudflare-tunnel) для cloudflared и заодно подружился с самим cloudflared.

Первая версия контроллера была честной обёрткой: она смотрела на Gateway API ресурсы и генерировала конфиг для моего же чарта. По сути, управляла им. Дальше упёрлось в две вещи: cloudflared не умеет горячо перезагружать конфиг, а часть нужных мне полей не публична. Пришлось [форкнуть cloudflared](https://github.com/lexfrei/cloudflared). А чтобы поддержать маршрутизацию Gateway API целиком, а не тот кусок, который умеет ingress-конфиг туннеля, форк пришлось переписать значительно: теперь внутри процесса cloudflared живёт L7-прокси, который и применяет правила HTTPRoute и GRPCRoute. Конфиг туннеля в Cloudflare используется только для DNS и edge-маршрутизации, вся логика запросов исполняется в кластере.

Потом я начал исполнять спецификацию Gateway API. Фанатично.

<details>
<summary>Насколько фанатично</summary>

Настолько, что официальный conformance-suite пришлось чинить, чтобы он смог пройти мой контроллер, а не наоборот. В сюите нашлись гонки и утечки, которые на других реализациях не стреляли: gRPC-клиент, небезопасный для параллельных `SendRPC` ([kubernetes-sigs/gateway-api#4946](https://github.com/kubernetes-sigs/gateway-api/pull/4946)), закрытие чужого gRPC-клиента ([#4939](https://github.com/kubernetes-sigs/gateway-api/pull/4939)), окно логов зеркалирования, которое открывалось после отправки запросов ([#4947](https://github.com/kubernetes-sigs/gateway-api/pull/4947)), очистка `MustApply`, не дожидавшаяся удаления ресурсов ([#4948](https://github.com/kubernetes-sigs/gateway-api/pull/4948)), плюс инжектируемые WebSocket-диалер и gRPC-клиент ([#4936](https://github.com/kubernetes-sigs/gateway-api/pull/4936), [#4937](https://github.com/kubernetes-sigs/gateway-api/pull/4937)). После этого контроллер прошёл сюиту для Gateway API v1.6.1 и попал в [апстримный список реализаций](https://gateway-api.sigs.k8s.io/implementations/) ([#4945](https://github.com/kubernetes-sigs/gateway-api/pull/4945)).

</details>

Всё, что ниже, работает благодаря этим трём вещам: свой чарт, свой cloudflared, строгая спецификация.

## Что нужно на стороне Cloudflare

Домен на Cloudflare, аккаунт Zero Trust (бесплатного тарифа хватает) и два секрета, которые легко перепутать.

**Туннель.** Zero Trust → Networks → Tunnels → Create a tunnel, коннектор Cloudflared. Сохраните Tunnel ID и Tunnel Token. Токен — это то, чем прокси в кластере авторизуется у edge; ID — то, на что будут смотреть DNS-записи.

**API-токен для контроллера.** Один, с правами `Account → Cloudflare Tunnel → Edit`. Им контроллер меняет конфигурацию туннеля.

**API-токен для external-dns.** Второй, с правами `Zone → Zone → Read` и `Zone → DNS → Edit`. Им external-dns заводит записи. Можно выдать один токен на всё, но два токена с минимальными правами дешевле объяснять самому себе через год.

## Контроллер

Gateway API CRD ставятся отдельно, стандартный набор:

```bash
kubectl apply --filename https://github.com/kubernetes-sigs/gateway-api/releases/download/v1.6.2/standard-install.yaml
```

Секреты и контроллер:

```bash
kubectl create namespace cloudflare-tunnel-system
kubectl create secret generic cloudflare-credentials \
  --namespace cloudflare-tunnel-system \
  --from-literal=api-token="<API_TOKEN>"
kubectl create secret generic cloudflare-tunnel-token \
  --namespace cloudflare-tunnel-system \
  --from-literal=tunnel-token="<TUNNEL_TOKEN>"

helm install cloudflare-tunnel-gateway-controller \
  oci://ghcr.io/lexfrei/charts/cloudflare-tunnel-gateway-controller \
  --namespace cloudflare-tunnel-system \
  --set gatewayClassConfig.create=true \
  --set gatewayClassConfig.tunnelID=<TUNNEL_ID> \
  --set gatewayClassConfig.cloudflareCredentialsSecretRef.name=cloudflare-credentials \
  --set proxy.tunnelTokenSecretRef.name=cloudflare-tunnel-token
```

Чарт ставит контроллер, прокси с форком cloudflared и GatewayClass `cloudflare-tunnel`, привязанный к вашему туннелю через GatewayClassConfig. Дальше нужен сам Gateway:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: Gateway
metadata:
  name: cloudflare-tunnel
  namespace: cloudflare-tunnel-system
spec:
  gatewayClassName: cloudflare-tunnel
  listeners:
    - name: https
      protocol: HTTPS
      port: 443
      allowedRoutes:
        namespaces:
          from: All
```

`allowedRoutes: from: All` разрешает HTTPRoute из любого namespace цепляться к этому Gateway. Для домашнего кластера это то, что нужно; в мультитенантной среде здесь ставят `Selector`. TLS-секции у листенера нет: TLS терминирует edge Cloudflare, до кластера доходит уже расшифрованный трафик внутри туннеля.

Когда Gateway готов, контроллер пишет ему адрес — CNAME туннеля вида `<TUNNEL_ID>.cfargotunnel.com` — и ставит `PROGRAMMED True`. Этот адрес и будет целью всех DNS-записей.

## DNS автоматически

Записи руками заводить не надо, для этого есть external-dns с источником `gateway-httproute`: он читает hostnames из HTTPRoute, а цель берёт из адреса Gateway, к которому маршрут привязан. Values для чарта external-dns:

```yaml
provider:
  name: cloudflare
policy: sync
txtOwnerId: cloudflare
sources:
  - gateway-httproute
extraArgs:
  - --gateway-namespace=cloudflare-tunnel-system
  - --annotation-filter=external-dns.kubernetes.io/cloudflare-proxied
env:
  - name: CF_API_TOKEN
    valueFrom:
      secretKeyRef:
        name: cloudflare-api-token
        key: api-token
```

Три строки здесь важнее остальных.

`--gateway-namespace` ограничивает external-dns одним Gateway: только маршруты, привязанные к Gateway в этом namespace, превращаются в записи. Без этого он начнёт заводить публичные записи для всего, что в кластере зовётся HTTPRoute, включая внутренние шлюзы, если они есть.

`--annotation-filter` делает то же самое с другой стороны: в DNS попадают только маршруты, на которых стоит аннотация `cloudflare-proxied`. Двойной фильтр выглядит паранойей ровно до первого случая, когда кто-то привязал тестовый маршрут не к тому Gateway.

`txtOwnerId` подписывает каждую запись TXT-записью с именем владельца. Именно по ней external-dns понимает, что запись его и её можно править или удалять. Если в одной зоне работают два инстанса external-dns, у них должны быть разные owner id, иначе они начнут удалять записи друг друга. К этому мы вернёмся во второй части, про split-horizon.

Аннотации external-dns ставятся на HTTPRoute, а не на Gateway: TTL, provider-specific настройки, всё. На Gateway имеет смысл только `external-dns.kubernetes.io/target`, если адрес нужно переопределить. У нас не нужно.

## Первый сервис

HTTPRoute из начала статьи, ещё раз, теперь с пояснениями:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: HTTPRoute
metadata:
  name: my-app
  namespace: my-app
  annotations:
    external-dns.kubernetes.io/cloudflare-proxied: "true"
spec:
  parentRefs:
    - name: cloudflare-tunnel
      namespace: cloudflare-tunnel-system
  hostnames:
    - app.example.com
  rules:
    - backendRefs:
        - name: my-service
          port: 80
```

`parentRefs` привязывает маршрут к Gateway. `hostnames` — то, что попадёт и в конфиг туннеля, и в DNS. Аннотация `cloudflare-proxied` делает две вещи одновременно: пропускает маршрут через `--annotation-filter` и говорит провайдеру Cloudflare завести запись проксированной. Почему это обязательно, ниже, в граблях.

После `kubectl apply`:

```text
$ kubectl get httproute --namespace my-app
NAME     HOSTNAMES             AGE
my-app   ["app.example.com"]   40s

$ kubectl get gateway cloudflare-tunnel --namespace cloudflare-tunnel-system
NAME                CLASS               ADDRESS                     PROGRAMMED   AGE
cloudflare-tunnel   cloudflare-tunnel   2001-db8.cfargotunnel.com   True         12m

$ dig +short app.example.com CNAME
2001-db8.cfargotunnel.com.

$ curl -sI https://app.example.com | head -1
HTTP/2 200
```

Ни одного входящего порта. Проверить можно с любого хоста снаружи: `nmap` в сторону вашего домашнего IP покажет ровно то, что показывал до этого.

### Отдельный hostname без правки Gateway

Если нужен hostname, который живёт по своим правилам, а Gateway трогать не хочется, есть ListenerSet:

```yaml
apiVersion: gateway.networking.k8s.io/v1
kind: ListenerSet
metadata:
  name: tools
  namespace: cloudflare-tunnel-system
spec:
  parentRef:
    group: gateway.networking.k8s.io
    kind: Gateway
    name: cloudflare-tunnel
  listeners:
    - name: tools
      port: 443
      protocol: HTTPS
      hostname: tools.example.com
      allowedRoutes:
        namespaces:
          from: All
```

Чтобы external-dns умел ходить по ListenerSet к адресу Gateway, ему нужен флаг `--gateway-listener-sets` и право читать `listenersets` в RBAC; в чарте это `rbac.additionalPermissions`.

## Грабли

Здесь начинается та часть, ради которой я бы сам прочитал этот текст два года назад.

**Запись обязана быть проксированной.** CNAME на `<TUNNEL_ID>.cfargotunnel.com` работает только через edge Cloudflare. Сам этот hostname не резолвится ни в один адрес:

```text
$ dig +short 2001-db8.cfargotunnel.com A
$
```

Пусто. DNS-only запись (серое облако) отдаст клиенту CNAME, у которого нет A-записей, и сайт просто не откроется, без единой ошибки со стороны туннеля. Отсюда аннотация `cloudflare-proxied: "true"` на каждом маршруте. Если лень ставить её на каждый, у провайдера есть флаг `--cloudflare-proxied`, который делает проксирование умолчанием, но тогда `--annotation-filter` придётся строить на другой аннотации.

**HTTPS только до второго уровня.** Бесплатный Universal SSL на зоне с full setup покрывает apex и один уровень поддоменов: `example.com` и `*.example.com`. Имя `app.example.com` получит сертификат, `app.home.example.com` — нет: у клиента будет ошибка сертификата, и никакие настройки туннеля это не лечат. Глубже нужен платный [Advanced Certificate Manager](https://developers.cloudflare.com/ssl/edge-certificates/advanced-certificate-manager/). Планируйте имена с двумя точками. Я на это наступил ещё в 2024-м и с тех пор все публичные имена у меня лежат прямо под apex.

**Префикс аннотаций сменился.** Начиная с external-dns v0.22.0 префикс по умолчанию — `external-dns.kubernetes.io/`, а старый `external-dns.alpha.kubernetes.io/` [работает только за флагом](https://kubernetes-sigs.github.io/external-dns/latest/docs/annotations/annotations/) `--enable-legacy-annotation-prefix`, и то временно. Если ваши манифесты написаны в прошлом году, после обновления записи начнут заводиться без `proxied`, и это ровно первая грабля, только тихая: `--annotation-filter` фильтрует по сырым аннотациям и продолжит пропускать маршруты, а провайдер аннотацию со старым префиксом уже не прочитает. Проверьте свои манифесты до апгрейда, а не после.

**Два секрета, а не один.** Токен туннеля и API-токен — разные вещи с разными правами. Токен туннеля ходит в прокси и держит соединение с edge, API-токен ходит в контроллер и правит конфигурацию. Перепутать их в секретах легко, ошибка в логах при этом выглядит как проблема сети.

**Историческая грабля.** Когда я собирал схему в 2024-м на официальном чарте, туннель авторизовался файлом `credentials.json` из трёх полей `AccountTag`, `TunnelSecret`, `TunnelID`, и строка должна была быть ровно такой, без форматирования, иначе cloudflared молча не стартовал. Плюс тег образа по умолчанию в чарте отставал на год. Контроллер этого не касается: он живёт на токене туннеля, а образ прокси едет вместе с релизом чарта. Упоминаю на случай, если вы найдёте в интернете инструкции того периода.

## Куда дальше

gRPC выставляется тем же способом, через GRPCRoute: тот же Gateway, тот же прокси, а если повесить на бэкенд BackendTLSPolicy, до него пойдёт TLS с mTLS по клиентскому сертификату Gateway. Это в [документации](https://cf.k8s.lex.la/latest/gateway-api/grpcroute/).

А главный вопрос, который остаётся после этой статьи: что делать с сервисами, которым снаружи не место. Домашний кластер — это не только то, что показываешь друзьям, но и Grafana, принтер и полка с медиа. Им нужен свой Gateway, свой DNS и своя зона видимости, а приложениям хочется описывать это одним манифестом. Это split-horizon, и он вторая часть.

Контроллер: [github.com/lexfrei/cloudflare-tunnel-gateway-controller](https://github.com/lexfrei/cloudflare-tunnel-gateway-controller), документация: [cf.k8s.lex.la](https://cf.k8s.lex.la/). BSD-3-Clause, pull request'ы принимаю и, если надо, помогу довести.
