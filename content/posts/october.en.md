---
title: "Monthly Report: Kubernetes, Projects, and Miscellaneous"
date: 2024-11-04T23:00:00+03:00
draft: false
tags: ["kubernetes", "helm", "open-source", "homelab", "monthly-recap"]
categories: ["tech"]
---

## VIP for Kubernetes API

Alright, time to brag a bit:

I became completely disillusioned with solutions like metallb, cilium, kube-vip, etc. for announcing kube API. Also, I don't want to do this externally. I don't want to manage it on hosts. I don't want to manage peer lists. Therefore, [vipalived](https://artifacthub.io/packages/helm/vipalived/vipalived). (Won the naming contest; "lube-vip" came in second).

A bit of context: Cilium replaced my kube-proxy, metallb, ingress controller, and CNI. It's cool and optimal — previously there were several network tools, now just one. But there's a side effect: Cilium became critically dependent on the kube API. This makes it unsuitable for VIP of the kube API itself — a vicious cycle.

My specific use case: "power got cut, how to bring up VIP for kube from within kube." I lived with a hardcoded first node for a long time and it worked, but I wanted to do it properly.

Essentially, it's a maximally simple thing — vrrp, keepalived, ds (or static pod), and a bit of faith. I didn't even bother making my own image (yet?), just pure apk add at startup.

This fully covers my needs for kube API VIP. Briefly, what's wrong with the rest:

- **kube-vip** requires connection to kube API for leader election, which isn't okay. Also requires a live CNI to start in other modes.
- **metallb** generally can't work without kube API.
- **cilium** in my case with kube-proxy replacement similarly requires connection to kube API to start.

This isn't a solution for everyone, it's purely mine, covering my needs. I didn't find analogues running in k8s, and the "manage externally" options don't appeal to me.

My use case scenario:

- **Day 1**: manually assign VIP to the interface
- **Day 2**: deploy the chart and it figures it out from there, no static pods, convenient to manage from k8s

Yes, this is a day 2 solution, but if you really need — you can generate a static pod and use it as day 1. But then I just don't see the point, honestly: it's more logical to manually set up VIP on the node before installing kube and then take it over via vipalived.

There are dozens of more correct ways to do this, but I wanted one that's ideally suited for me. In general, judging by the fact that nobody did this before me — only I need it. But I'll be glad if it similarly makes someone's life easier.

## IPMI Exporter

I finally remembered that I'm the main and only maintainer of the [IPMI Exporter](https://artifacthub.io/packages/helm/prometheus-community/prometheus-ipmi-exporter) chart.

Got around to cleaning it up, writing tests, adding schema, throwing in a couple of new features, and finally using it myself.

Wouldn't be surprised if some of you use it but didn't know it was mine.

## War with external-dns

My war with external-dns continues.

I'm battling to push through the feature for annotation changes: [kubernetes-sigs/external-dns#5889](https://github.com/kubernetes-sigs/external-dns/pull/5889).

Meanwhile, I'm fixing documentation ([kubernetes-sigs/external-dns#5918](https://github.com/kubernetes-sigs/external-dns/pull/5918), [kubernetes-sigs/external-dns#5923](https://github.com/kubernetes-sigs/external-dns/pull/5923)), setting up linting ([kubernetes-sigs/external-dns#5929](https://github.com/kubernetes-sigs/external-dns/pull/5929)), and proposing global changes ([kubernetes-sigs/external-dns#5919](https://github.com/kubernetes-sigs/external-dns/pull/5919)).

The review process has been somewhat challenging — I'm experiencing what feels like inconsistent feedback from some reviewers. It's been a learning experience navigating different expectations and review styles. Despite the friction, I feel strong community support, which gives me confidence that this work isn't in vain and the feature is genuinely needed by people.

## Gateway API and Migration from Ingress

My parallel war with Ingress. I like Gateway API and I'm suffering now, but trying to do it right for everything I use.

I managed to easily bring [argoproj/argo-helm#3517](https://github.com/argoproj/argo-helm/pull/3517) to Argo project for CD, and now made [argoproj/argo-helm#3567](https://github.com/argoproj/argo-helm/pull/3567) for Workflows.

In longhorn they're slightly ignoring me ([longhorn/longhorn#9615](https://github.com/longhorn/longhorn/pull/9615)), but I managed to volunteer to fix a related problem [longhorn/longhorn#10583](https://github.com/longhorn/longhorn/issues/10583) ([longhorn/longhorn#12050](https://github.com/longhorn/longhorn/pull/12050), [longhorn/longhorn-manager#4245](https://github.com/longhorn/longhorn-manager/pull/4245)).

They're completely ignoring me ([kubernetes/dashboard#10385](https://github.com/kubernetes/dashboard/pull/10385)) in kubernetes/dashboard.

Easily accepted ([kyverno/policy-reporter#1238](https://github.com/kyverno/policy-reporter/pull/1238)) in kyverno/policy-reporter.

Overall, nobody's against such features, but many places take time.

## Minecraft Projects

### papermc-docker

[lexfrei/papermc-docker](https://github.com/lexfrei/papermc-docker) — got a [chart](https://artifacthub.io/packages/helm/minecraft/papermc), now definitely builds every night (not just the first 3 months). As a reminder, this is my view on what a PaperMC image should look like. I once tried to bring a PR (won't search for the link, too lazy) to [itzg/docker-minecraft-server](https://github.com/itzg/docker-minecraft-server), but they didn't understand me, said it's too complicated, so I made my own rootless with goodies. As a bonus, CLI-tool/lib [goPaperMC](https://github.com/lexfrei/goPaperMC) was born.

### minecraft-operator

Since I'm not satisfied with how difficult it is to update this PaperMC, I started writing my own [minecraft-operator](https://github.com/lexfrei/minecraft-operator). Side product — CLI-tool/lib [go-hangar](https://github.com/lexfrei/go-hangar).

The idea: manage servers through CRD:

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
  updateStrategy: auto # Tries to pick the best version for all other components
  eula: true
---
apiVersion: mc.k8s.lex.la/v1alpha1
kind: Plugin
metadata:
  name: bluemap
  namespace: default
spec:
  updateStrategy: latest # Applies the latest version despite everything
  source:
    type: hangar
    project: "BlueMap"
  instanceSelector:
    matchLabels:
      type: prod
```

Plugins have autodiscovery, compatibility solver, and this applies to each individual server. Servers have their own maintenance windows and update themselves too. There are different update strategies. And most importantly: webUI for navigating server configs! In general, everything so my nephew can play with the boys.

It's still in testing stage (all code in PR) and there are a couple of rough edges, but overall — MVP is almost ready.

## Other Projects

### system-upgrade-controller

From another conflict [rancher/system-upgrade-controller#384](https://github.com/rancher/system-upgrade-controller/pull/384), the chart [system-upgrade-controller](https://artifacthub.io/packages/helm/system-upgrade/system-upgrade-controller) was born.

### cloudflare-tunnel

In the [cloudflare-tunnel](https://artifacthub.io/packages/helm/cloudflare-tunnel/cloudflare-tunnel) chart, all possible features finally appeared, chart auto-update and so on. I can't use it without frustration, but continue to maintain it for those who need it.

### transmission

I maintain the [transmission](https://artifacthub.io/packages/helm/transmission/transmission) chart purely for myself, didn't find ready-made and nice ones.

### mtg-scanner

I finally tackled the [project](https://github.com/lexfrei/mtg-scanner) with MTG card recognition. Don't understand anything, pure vibes-based coding, but it works. Planning to do embeddings in Python, and run on Go. On Raspberry Pi with AI shield. Now I'm at the dataset preparation stage. Very lazily, but I'll manage.

## Transition to OCI for Helm Repositories

Forcing transition to OCI for helm repos. Overall, going well, transferred all my charts to this publication variant. But there are two points:

- Renovate can't ([renovatebot/renovate#39067](https://github.com/renovatebot/renovate/discussions/39067)) do correct auto-update, had to hack around it.
- I still haven't figured out how to sign images for artifacthub.io. For now just gave up.

## Miscellaneous

### Argo Events and Argo Workflows

Mastered Argo Events and Argo Workflows. Beautiful. Maybe I'll completely throw out system-upgrade-controller in favor of this combo.

The essence is simple: system-upgrade-controller just runs privileged jobs to update nodes. Argo Workflows can do the same, but with much greater flexibility in orchestration.

Technically it's not a problem at all: mount the host filesystem / into /mnt/system inside the container, do chroot there — and that's it, you're working as if on the node itself. Packed in a job, ran with needed privileges, profit.

Plus Argo Events provides triggers and automation, which opens a bunch of possibilities for other scenarios in the cluster. In general, still learning, but I see huge potential.

### Security

Started poking at Trivy and Kyverno bringing the cluster to order. My first and clumsy steps in cluster security.

### Hardware

Bought HP ML310e as a replacement for MicroServer. It seemed like a good idea, since a good PSU (which I suspect) costs as much as this 310. There are nuances: I already lost to the fans, simple connector re-soldering didn't help. Will think further, maybe will have to flash iLo. But the plan still includes 10g, LSI HBA, and other delights.

### One-liners

Figured out iLo monitoring (expect a separate post, everything's a bit more complicated there).

Figured out watchdogd (expect a separate post).

Started tuning NFS/ZFS (expect a separate post).

Started tuning kernel for kube (you get the idea).

Unexpectedly attended @homelab_meetup.

Got quite sick.
