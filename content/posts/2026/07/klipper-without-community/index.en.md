---
title: "Klipper: open source without a community, or why your printer will never ship with a toolchanger"
date: 2026-07-24T17:39:46+03:00
slug: "klipper-without-community"
draft: false
tags: ["open-source", "community", "klipper", "3d-printing"]
categories: ["tech"]
---

*English version of the Russian original published on [Habr](https://habr.com/ru/companies/aenix/articles/1062774/).*

The man whose code has booted your virtual machines for the last 20 years or so (SeaBIOS has been the default BIOS of QEMU/KVM since about 2010) today decides single-handedly what goes into [Klipper](https://www.klipper3d.org/), the firmware behind a good half of the planet's 3D printers, from home builds to factory machines. This is not a metaphor about spiritual leadership. It is literally a one-row table in the official documentation.

I do not contribute to open source "on weekends". Upstream contributions are part of my job and I get paid for them: I work on CNCF infrastructure projects, and landing code in mainline is my production process. Some of what started as pet projects is now used by companies in production, so even my hobby code lives by grown-up rules. So what follows is not "a user is upset that his feature got rejected". It is the view of someone for whom the economics of open source is daily work. And I want to talk about Klipper. Not about the code: the code is excellent, hat off. About how the project is set up as an organization. Spoiler: it isn't.

## Disclaimer

Kevin O'Connor is an outstanding engineer. SeaBIOS and Klipper are truly fundamental things, made mostly by one person. Nothing below is about code quality or personality. It is about processes. More precisely, about their absence and what it costs everyone else.

Second: despite the mentions of my work and Cozystack in the text, everything here is my personal opinion as an active open source participant, not an official position of Aenix or the Cozystack project.

## A table with one row

Open Klipper's [CONTRIBUTING.md](https://github.com/Klipper3d/klipper/blob/master/docs/CONTRIBUTING.md). It has a table titled "The Klipper "maintainers" are":

|                |                                                  |
|----------------|--------------------------------------------------|
| Name           | GitHub name                                      |
| Kevin O'Connor | [@KevinOConnor](https://github.com/KevinOConnor) |

That's it. One row. There are four reviewers (Kevin included), but only the maintainer can commit to master. A project with 11,000+ stars, thousands of forks, and probably millions of installations (nearly every serious home build plus a pile of vendor machines) has a bus factor of one.

For comparison: this is not a small pet project that suddenly took off. Klipper is the de facto industry standard host firmware for FDM. Companies with hundreds of millions in revenue build products on it.

And to be clear: the one-person project model is legitimate in itself. The reference example is [Valetudo](https://valetudo.cloud/), firmware that cuts robot vacuums off from the manufacturer's cloud: the vacuum stops reporting to the vendor's servers and obeys only you, a cloud lobotomy for the greater good. Its author, Sören Beye, states on the front page that Valetudo is a hobby of some random guy from the internet, that he has no intention of commercializing the project or growing its audience, and moreover explicitly intends *not* to. He defines the distribution model as "Freeware with source available", and the section on the front page is titled "Valetudo is a garden". Think of the project, he writes, as a private garden open to the public: come in for free, walk around, bring friends, leave a tip at the entrance, take ideas home to your own garden, but remember that you are a guest on someone else's land, you have no power over it, and being let in at all is a gift. You can argue about the tone, but it is honest: the contract is announced before you have put a single minute into the project. Klipper lives by exactly the same rules, it just never wrote them down anywhere, while being the foundation of other people's businesses and having a CONTRIBUTING.md that *looks* like an invitation to collaborate. The complaint is not about dictatorship. It is about the gap between the declared and the actual contract at industrial scale.

## The hundred-user rule

The same CONTRIBUTING.md has a remarkable clause: a submission must have a "noteworthy target audience", rule of thumb at least 100 real users. Sounds reasonable? It did to me too, until I saw how the rule works in practice.

A case study. PR [#6826](https://github.com/Klipper3d/klipper/pull/6826): let klippy listen on a TCP socket instead of (more precisely, in addition to) the Unix domain socket. Strictly opt-in, default behavior unchanged. Why: containerized and distributed setups, weak printer hardware where you want to move [Moonraker](https://github.com/Arksine/moonraker) and the UI to a separate machine. Kevin replies in the PR: long-term contributors have concerns, "I'd be inclined to hold off on this until there's a clear audience of a notable size… we typically aim for a minimum of at least a 100 users".

Fine, the rule is stated, let's play by the rule. I bring numbers: the unofficial Docker image [mkuf/moonraker](https://hub.docker.com/r/mkuf/moonraker) has 50 thousand pulls. Divide by 120 tags under the deliberately absurd assumption that every user pulled every tag exactly once, and you get 400+ users by the most conservative estimate possible. Every one of them needs this feature, because the whole containerized Klipper setup is built around sharing a Unix socket between containers. Four times the stated threshold.

The answer: silence. Not "rejected, here is why", not "the numbers don't convince me because…", not "rework this part". The thread just ended.

And this is the core problem with rules enforced by one person: there is a rule, but there is no contract. You can meet every documented condition and that does not guarantee even a reply. The decision is still made by one person's internal criteria, written down nowhere. There is nobody to appeal to and nowhere to do it.

This, by the way, is what foundations exist for. The point of the [CNCF](https://www.cncf.io/) is not that it writes code; people write code. The point is that there is an authority above the maintainer: documented governance as a precondition for accepting a project, a technical committee, an escalation procedure. You can lose an argument, but you have somewhere to bring it, and losing by the rules is a normal part of the process. In Klipper you cannot lose: the argument simply never happens.

## The GitHub that isn't there

At the time of writing, the Klipper repository has **one** open issue. And you know which one? [#6804](https://github.com/Klipper3d/klipper/issues/6804), "This GitHub issue tracker is no longer used", opened by Kevin himself in February 2025. So even that single issue is not a bug but a sign nailed to the door: tracker closed, everyone to the forum. With 200+ pull requests hanging. No, this is not because there are no bugs. Bug tracking as a practice is absent from the project; everything is routed to [klipper.discourse.group](https://klipper.discourse.group).

A forum is a fine tool for discussion. But when it *replaces* the tracker, the main thing disappears: status. An issue has a lifecycle: open, triaged, assigned, closed with a reason. A forum thread has exactly one status: "Kevin replied" or "Kevin did not reply". And silence, as we found out with PR #6826, is also a reply.

A separate hello to the internal API. The Python modules in `klippy/extras` have no stability contract at all. Any refactoring can break every external integration and add-on, and that is considered normal: if you want stability, get into the core. Into which nothing gets. The circle is closed.

## What grows in that place: downstreams

When nothing substantial can land upstream, the physics of open source takes over: code starts living in forks. Not because everyone wants to fork, but because there is nowhere else to go.

**Toolchangers**: printers that swap their own print heads the way a CNC machine swaps cutters. Mainline Klipper has no support for them. None. The community standard is a third-party add-on, [klipper-toolchanger](https://github.com/viesturz/klipper-toolchanger), living on a handshake on top of an unstable internal API.

**Snapmaker U1.** The first truly mass-market toolchanger printer. On 30 March 2026, the last day before the GPL compliance deadline, Snapmaker published its forks of [Klipper](https://github.com/Snapmaker/u1-klipper), Moonraker and Fluidd. By their own estimate they modified **about 20% of the Klipper codebase** for the parallel multi-toolhead system. Twenty percent! That is not a patch on the side, that is a parallel branch of evolution. And there is little to blame them for: everyone knows that if Kevin does not need a toolchanger, the core will not have one. And a compatibility layer can break any Friday. So live on your fork.

**Sovol M1D.** A printer announced just now: IDEX plus a six-head toolchanger, up to seven materials. I make a testable prediction: it will be one more vendor fork of Klipper with one more home-grown, incompatible-with-everything implementation of tool switching. Let's check in a year.

**Creality** and the other vendors ship their Klipper forks frozen on ancient versions, because vendor patches do not survive a rebase. The user gets a "Klipper" that is three years old and no upgrade path.

Total: instead of one multi-tool implementation in the core, reviewed and maintained together, the industry has N incompatible implementations, each carried by one vendor. That is the price of missing governance, expressed in person-years.

The economics deserve spelling out here, because "greedy vendors don't want to share" is a popular explanation and a wrong one. It is exactly the opposite. A fork is not an asset, it is a liability: the vendor pays for it forever. Every rebase, every backport of a security fix, every incompatibility with the add-on ecosystem is payroll that drips as long as the product lives. An upstream contribution is a one-off (if expensive) investment that converts perpetual costs into shared ones: other people's eyes review the code, the community maintains it, ecosystem compatibility comes for free. I know this not from a textbook: landing code upstream is in my job description precisely because it is *cheaper* for my employer.

This is symbiosis in its pure form. Contributors fix their own pain with fixes and features. Maintainers make that possible, and that is work too, work it is fair to pay for. One side gets a living project it can monetize, the other a working community version. And Kevin already monetizes: BIGTREETECH is the official mainboard sponsor, Obico is a sponsor, plus personal donations via Patreon and Ko-fi. That is normal and right: a maintainer owes nobody free work, and nobody may demand a single line of code from him. The question is not whether he should take money; he does. The question is the form: money flows into the project and not a single obligation arises. Sponsorship without a seat at the table, without a committee, without a procedure. The symbiosis benefits every participant, the maintainer included, on one condition.

The condition: mainline must be able to take in that flow without choking. That is what processes exist for: a hierarchy of maintainers, a split into subsystems, documented acceptance criteria, delegated review. (I have separate thoughts on what happens to this condition in the era of AI-generated contributions; if you are interested, say so in the comments and it becomes the next post.) The hundred-user rule by itself is a sensible filter. But a filter without throughput is not a filter, it is a wall. Klipper's throughput equals one person, and therefore the only economically rational strategy for a vendor is a fork. Not because they want to, but because there is physically no alternative.

And believe me: companies would be happy to bring that code upstream and say "take it, maintain it not alone". More than that, they could take ownership of whole subsystems: here is your toolchanger module maintainer from Snapmaker, here is CI on our hardware, review us, hold us accountable. That is exactly how it works in the Linux kernel, where Intel, Red Hat, Google and dozens of others stand behind subsystems. But there is nowhere to bring it and nothing to own. No foundation, no technical committee, no roadmap, no procedure by which an interested company can buy itself even *consideration* of a feature. Not approval, just consideration. Collaboration is not for sale even for money.

## "So fork it, all together"

Been tried. [Kalico](https://github.com/KalicoCrew/kalico) (formerly Danger-Klipper; a separate thank-you for the naming, Calico from the Kubernetes world says hi, I still mix them up) is a living, likeable fork. But look at what it carries: a scattering of relatively small patches and rejected PRs. Kalico cannot pull in anything the size of a toolchanger subsystem: it has no resources for full parallel development and is doomed to rebase forever onto an upstream where code can change so that other people's patches stop applying.

And a "big fork by the whole community" is not about code, it is about management. The fresh MinIO case is telling: when the admin UI was cut from the community edition, a fork called OpenMaxIO appeared and stalled almost immediately after creation. MinIO had money, users and HN hype behind it, and it did not take off. Forking a project of Klipper's scale is titanic work that runs straight into what the Klipper community lacks: organization.

Plus the eternal "if it works, don't touch it". Klipper *prints*. Prints well. The average user has zero motivation for a revolution, and small things can go into Kalico.

## Even Linus

For the counterexample I deliberately take the least convenient one. The Linux kernel is not a cozy project with a Code of Conduct on the front page. In 2026 patches still go there by *email*: plain text, the way the elders decreed (yes, I did it this year, diffs for a NAND driver). And its creator is a man whose reputation in communication is, let's say, documented: [LKML](https://lkml.org/), the kernel developers' mailing list, has kept everything in plain text for thirty years.

Now look. Linus does not need Rust. He does not write it and does not share the hype. But the kernel has a machine: a hierarchy of lieutenants, subsystem maintainers, documented rules, a public process. When one of the lieutenants tried to single-handedly block Rust bindings in his subsystem, the machine, with Linus as arbiter, ran over the *lieutenant*, not the contributors. Rust is in the kernel. Through pain, through drama, through some people leaving, but the process exists and it can overrule even the most authoritative participants, including, when needed, Linus himself.

EVEN Linus, the industry's most archetypal [BDFL](https://en.wikipedia.org/wiki/Benevolent_dictator_for_life), built a system bigger than himself. Kevin built a system *exactly equal* to himself. That is the whole difference.

## Conclusions

Open source is not a license in the repository root. Klipper's license is impeccable, and in this configuration nearly useless. Look: the GPL worked, Snapmaker honestly published a mountain of code. What changed for the world? Nothing. There is nowhere to merge that code, nobody will rebase it, it will live exactly as long as Snapmaker sells the U1. GPL compliance without a working intake channel produces not collaboration but dead dumps, a bureaucratic tax paid on the last day of the reporting period. For contrast: FreeBSD with its permissive license obliges nobody to anything. Sony builds the PlayStation operating system on it, Apple has pulled its code into the macOS kernel for decades while giving almost nothing back upstream, and that is legal. And yet Netflix *voluntarily* invests in FreeBSD's network stack year after year, because there is somewhere to invest and it pays off. It is not the license that brings code back upstream. It is a working intake channel. (The GPL vs BSD holy war stays out of frame; only this particular case matters here.)

Open source is processes: a contract between the project and its contributors, predictability, a community's ability to influence direction. On all of these points Klipper is open source only formally.

What to do about it? The industry actually knows the answer: a vendor fork under a neutral foundation. That is how [OpenTofu](https://opentofu.org/) (a fork of Terraform), [Valkey](https://valkey.io/) (a fork of Redis) and [OpenSearch](https://opensearch.org/) did it: commercial players with a shared pain chip in engineers, the foundation provides neutrality and governance, and nobody takes the project away from anybody. The mechanism exists and works. The problem is that OpenTofu had an ecosystem behind it where such consortia are a habit: foundations, lawyers, ready procedures. 3D printing vendors are yesterday's startups with no culture of collaboration at all. Not out of malice, they just never did it. So we have what we have: Kalico, which has code but neither money nor a charter, and Kevin, whom I sincerely wish an honorable retirement and a monument in his lifetime for the code he wrote.

Maybe there is a path I don't see. What should a community do when its de facto standard is locked behind a bus factor of 1? Let's talk in the comments, I am genuinely curious.

Meanwhile, if you are a vendor and you are building a toolchanger, I have bad news for you: welcome to the club, your fork is already waiting.

## What you personally can do

So that the article does not end in pure whining, the practical part. If you have a personal pain, it is **worth** carrying to mainline. Where the intake channel works, it pays back out of all proportion to the cost. A fresh example from my own practice: the [HeYangTek SPI-NAND driver in the Linux kernel](https://git.kernel.org/pub/scm/linux/kernel/git/mtd/linux.git/commit/?id=a8374683868634012ac873d628fa581fcc452e9c). The core logic of the driver was written by a Keenetic engineer in their downstream, the classic fate of vendor code doomed to rot. My work was to comb it into mainline shape and carry it to the mtd maintainers: a couple of dozen lines of my own, some correspondence, and a cup of coffee. Result: the chip is now supported in the kernel forever, OpenWrt is unblocked on the Keenetic KN-3411 and other routers with this NAND, regardless of what happens next to the vendor, its firmware or its development department. Yes, kernel patches go by email. Yes, it is archaic. But it is archaic with a working contract: I met the conditions, the code is in the tree. An ROI no fork ever dreamed of.

For symmetry, a confession: we do "strange" things too, and not rarely. The Cozystack ecosystem has [blockstor](https://github.com/cozystack/blockstor), a control plane for LVM/ZFS storage with DRBD replication that speaks a [LINSTOR](https://linbit.com/linstor/)-compatible REST API but is rewritten from scratch inside the Kubernetes-native way: CRDs and reconcilers instead of a central controller with its own database. And [cozyplane](https://github.com/lllamnyp/cozyplane), a multi-tenant [eBPF](https://ebpf.io/) CNI replacing the [Cilium](https://cilium.io/) plus [kube-ovn](https://github.com/kubeovn/kube-ovn) pair. How is this different from the vendor forks of Klipper? In one word: choice. Bringing an architectural change of that size into LINSTOR through a PR is impossible in principle; it is not a feature, it is a different project. We did the math and consciously took the maintenance burden on ourselves, knowing its price. Reimplementation is a legitimate tool when carrying to mainline is *really* more expensive. Klipper's problem is not that vendors fork, it is that they have no moment where they could do the math and choose otherwise.

And since I am campaigning here anyway: I maintain [Cozystack](https://github.com/cozystack/cozystack) (CNCF Sandbox) and am personally ready to take your PRs, and if you don't know how, I will show you and teach you. Because the only honest answer to "project X has bad processes" is building projects with good ones.

---

*"Open source is when some crank has a project and somebody helps him with it." Klipper is when helping is not allowed.*
