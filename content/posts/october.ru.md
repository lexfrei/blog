---
title: "Отчёт за месяц: Kubernetes, проекты и разное"
date: 2024-11-04T23:00:00+03:00
draft: false
tags: ["kubernetes", "helm", "open-source", "homelab", "monthly-recap"]
categories: ["tech"]
---

## VIP для Kubernetes API

Ладно, похвастаюсь пока таким:

Я полностью разочаровался в решениях вроде metallb, cilium, kube-vip и т.п. для анонса kube api. Также, я не хочу делать это снаружи. Я не хочу менеджить это на хостах. Я не хочу менеджить список пиров. Поэтому [vipalived](https://artifacthub.io/packages/helm/vipalived/vipalived). (На конкурсе названий lube-vip занял второе место).

Немного контекста: у меня cilium заменил kube-proxy, metallb, ingress controller и CNI. Это крут и оптимально — раньше было несколько инструментов для сети, теперь один. Но есть побочка: cilium стал жесточайше зависим от kube api. Это делает его непригодным для VIP самого kube api — получается замкнутый круг. 

Мой use case конкретно: "рубанули электричество, как поднять vip для куба из куба". Я долго жил с хардкодом просто первой ноды и это работало, но захотел сделать правильно.

В сущности, максимально простая штука — vrrp, keepalived, ds (или static pod) и немного веры. Я даже (пока?) не стал делать свой образ, чисто apk add на старте.

Это полностью закрывает мои потребности в vip для kube api. Кратенько, что не так с остальными:

- **kube-vip** требует подключения к kube api для leader election, что не ок. Также, требует живого CNI для старта в других режимах.
- **metallb** в целом не может работать без kube api.
- **cilium** в моём кейсе с kube-proxy replace аналогично требует для начала подключения к kube api.

Это не решение для всех, это чисто моё, закрывающее мои потребности. Аналогов, бегущих в k8s я не нащёл, а варианты "менеджить снаружи" мне не нравится.

Мой сценарий использования:

- **Day 1**: вешаешь VIP мануально на интерфейс
- **Day 2**: закидываешь чарт и дальше он сам разберётся, никаких static подов, удобно менеджить из k8s

Да, это day 2 решение, но если очень надо — можно сгенерить static pod и использовать его как day 1. Но тогда я просто не вижу смысла, если честно: логичнее тогда просто мануально на ноде перед установкой куба vip прописать и уже потом перехватить это через vipalived.

Есть десятки более правильных способов делать это, но я захотел такой, идеально подходящий именно мне. В общем, судя по тому, что этого не сделали до меня — это нужно только мне. Но буду рад, если кому-то аналогично облегчит жизнь.

## IPMI Exporter

Я наконец вспомнил, что я основной и единственный мейнтейнер [IPMI Exporter](https://artifacthub.io/packages/helm/prometheus-community/prometheus-ipmi-exporter) чарта.

Дошли руки причесать, написать тесты, добавить схему, подкинуть пару новых фич и, наконец, заюзать у себя.

Не удивлюсь, если кто-то из вас им пользуется, да не знал, что это моё.

## Война с external-dns

Продолжается моя война с external-dns.

Я с боем пропушиваю фичу по изменению аннотаций: [kubernetes-sigs/external-dns#5889](https://github.com/kubernetes-sigs/external-dns/pull/5889).

А тем временем, правлю им документацию ([kubernetes-sigs/external-dns#5918](https://github.com/kubernetes-sigs/external-dns/pull/5918), [kubernetes-sigs/external-dns#5923](https://github.com/kubernetes-sigs/external-dns/pull/5923)), настраиваю линтинг ([kubernetes-sigs/external-dns#5929](https://github.com/kubernetes-sigs/external-dns/pull/5929)) и предлагаю глобальные изменения ([kubernetes-sigs/external-dns#5919](https://github.com/kubernetes-sigs/external-dns/pull/5919))

Процесс ревью оказался довольно непростым — ощущаю что-то вроде непоследовательной обратной связи от некоторых ревьюеров. Это опыт навигации по разным ожиданиям и стилям ревью. Несмотря на трения, я ощущаю поддержку сообщества, это даёт мне понимание, что я всё это делаю не зря и моя фича нужна людям.

## Gateway API и миграция с Ingress

Параллельная моя война с Ingress. Мне нравится Gateway API и я сейчас страдаю, но пытаюсь сделать праивильно для всего, что использую.

В проект Argo мне удалось легко принести [argoproj/argo-helm#3517](https://github.com/argoproj/argo-helm/pull/3517) для CD, а сейчас сделал [argoproj/argo-helm#3567](https://github.com/argoproj/argo-helm/pull/3567) для Workflows.

В longhorn меня слегка игнорируют ([argoproj/argo-helm#3517](https://github.com/argoproj/argo-helm/pull/3517)), зато я напросился исправить связанную проблему [longhorn/longhorn#10583](https://github.com/longhorn/longhorn/issues/10583) ([longhorn/longhorn#12050](https://github.com/longhorn/longhorn/pull/12050), [longhorn/longhorn-manager#4245](https://github.com/longhorn/longhorn-manager/pull/4245)).

Меня полностью игнорируют ([kubernetes/dashboard#10385](https://github.com/kubernetes/dashboard/pull/10385)) в kubernetes/dashboard.

Легко приняли ([kyverno/policy-reporter#1238](https://github.com/kyverno/policy-reporter/pull/1238)) в kyverno/policy-reporter.

В целом, никто не против таких фич, но много где это занимает время.

## Minecraft проекты

### papermc-docker

[lexfrei/papermc-docker](https://github.com/lexfrei/papermc-docker) — обзавёлся [чартом](https://artifacthub.io/packages/helm/minecraft/papermc), теперь точно собирается каждую ночь (а не только первые 3 месяца). Напомню, это мой взгляд на то, как должен выглядеть образ для PaperMC. Когда-то я пытался принести PR (ссылку искать не буду, мне лень) в [itzg/docker-minecraft-server](https://github.com/itzg/docker-minecraft-server), но там меня не поняли, сказали, что это слишком сложно и я сделал свой rootless с ништяками. В качестве бонуса родился CLI-tool/lib [goPaperMC](https://github.com/lexfrei/goPaperMC).

### minecraft-operator

Так как меня не устраивает история с тем, что сложно обновлять этот PaperMC, я начал писать свой [minecraft-operator](https://github.com/lexfrei/minecraft-operator). Побочный продукт — CLI-tool/lib [go-hangar](https://github.com/lexfrei/go-hangar).

Идея: управлять серверами через CRD:

```yaml
apiVersion: mc.k8s.lex.la/v1alpha1
kind: PaperMCServer
metadata:
  name: test-server
  namespace: default
  labels:
    environment: test
    type: prod
spec:
  updateStrategy: auto # Пытается подобрать самую лучшую версию для всех остальных компонентов
  eula: true
---
apiVersion: mc.k8s.lex.la/v1alpha1
kind: Plugin
metadata:
  name: bluemap
  namespace: default
spec:
  updateStrategy: latest # Применяет последнюю версию вопреки всему
  source:
    type: hangar
    project: "BlueMap"
  instanceSelector:
    matchLabels:
      type: prod
```

У плагинов есть автодискавери, солвер по совместимости и это применяется к каждому отдельному серверу. У серверов свои окна обслуживания и сами они тоже обновляются. Есть разные стратегии обновления. И главное: webUI для хождения по конфигам сервера! В общем, всё для того, чтоб племяшка играл с пацанами.

Оно пока на стадии тестирования (весь код в PR) и есть пара шероховатостей, но в целом — почти готов MVP.

## Другие проекты

### system-upgrade-controller

Из другого конфликта [rancher/system-upgrade-controller#384](https://github.com/rancher/system-upgrade-controller/pull/384) родился чарт [system-upgrade-controller](https://artifacthub.io/packages/helm/system-upgrade/system-upgrade-controller).

### cloudflare-tunnel

В чарте [cloudflare-tunnel](https://artifacthub.io/packages/helm/cloudflare-tunnel/cloudflare-tunnel) наконец появились все возможные фичи, автообновление чарта и вообще. Я не могу им пользовать без такой-то матери, но продолжаю поддерживать для тех, кому надо.

### transmission

Чарт [transmission](https://artifacthub.io/packages/helm/transmission/transmission) поддерживаю чисто для себя, готовых и красивых я не нашёл.

### mtg-scanner

Я наконец взялся за [проект](https://github.com/lexfrei/mtg-scanner) с распознованием mtg-карточек. Ничего не понимаю, чистый вайбкодинг, но работает. Планирую делать эмбеддинги на петухоне, а гонять уже на Go. На малине с AI-шилдом. Сейчас я на этапе подготовки датасета. Очень лениво, но я справлюсь.

## Переход на OCI для Helm репозиториев

Форсирую переход на oci для helm repos. В целом, идёт бодро, перевёл все свои чарты на такой вариант публикации. Но есть два момента:

- Renovate не может ([renovatebot/renovate#39067](https://github.com/renovatebot/renovate/discussions/39067)) в корректное автообновление, пришлось костылять.
- Я так и не разобрался как подписывать образы для artifacthub.io. Пока просто отказался.

## Разное

### Argo Events и Argo Workflows

Освоил Argo Events и Argo Workflows. Красивое. Возможно, я вообще выкину system-upgrade-controller в пользу этой связки.

Суть проста: system-upgrade-controller просто запускает привилегированные джобы для обновления нод. Argo Workflows может делать то же самое, но с гораздо большей гибкостью в оркестрации. 

Технически это вообще не проблема: монтируешь файловую систему хоста / в /mnt/system внутри контейнера, делаешь chroot туда — и всё, ты работаешь как будто на самой ноде. Упаковал в джобу, запустил с нужными привилегиями, profit. 

Плюс Argo Events даёт триггеры и автоматизацию, что открывает кучу возможностей для других сценариев в кластере. В общем, пока изучаю, но потенциал вижу огромный.

### Безопасность

Начал ковырять Trivy и Kyverno приводя куб к порядку. Мои первые и неуклюжие шаги в безопасность куба.

### Железо

Купил HP ML310e как замену MicroServer. Это показалось хорошей идеей, ведь хороший БП (который у меня под подозрением) стоит как это 310ый. Есть нюансы: я уже проиграл вентиляторам, простая перепайка разъёма не помогла. Буду думать дальше, возможно, придётся шить iLo. Но в планах всё равно 10g, LSI HBA и другие прелести.

### Одной строкой

Разобрался с мониторингом iLo (ждите отдельный пост, там всё чуть сложнее).

Разобрался с watchdogd (ждите отдельный пост).

Начал тюнить NFS/ZFS (ждите отдельный пост).

Начал тюнить ядро под куб (ну выпоняли).

Внезапно побывал на @homelab_meetup.

Сильно приболел.
