---
title: "GitHub's Rough Patch: Security Holes, AI Slop, and an Identity Crisis"
slug: "githubs-rough-patch-security-holes-ai-slop-and-an-identity-crisis"
description: "GitHub is having a rough time — a critical RCE vulnerability, embarrassing outages, an AI-generated content flood, and an identity slowly absorbed into Microsoft. Here's a breakdown."
featured: true
pubDatetime: 2026-05-01T11:12:00+02:00
tags: ["github", "security", "ai", "open-source", "microsoft", "devtools"]
---

<p dir="rtl">اوضاع گیت‌هاب این روزها خیلی خوب نیست. اخیرا گیت‌هاب با یک آسیب‌پذیری امنیتی جدی، قطعی‌های مکرر، سیل عظیمی از مشارکت‌های بی‌کیفیت تولیدشده توسط هوش مصنوعی (همون چیزی که به عنوان slopهای تولید شده از هوش مصنوعی شناخته می‌شه)، و مهاجرت کامل به Microsoft Azure روبه‌رو شده.. و همه اینها در حال تغییر بنیادی ماهیت این پلتفرم بزرگ هستند.</p>

![Github 2010](@assets/images/github-2010.jpeg)

If you’ve been keeping an eye on the developer scene lately, you’ve probably seen that GitHub, the platform we all thought would always be there, solid as a rock, has been struggling. We’re talking security issues, awkward outages, a ton of AI-generated clutter, and its identity slowly merging into Micro$oft’s machine.

On a personal note, about a week ago, I was asked to look into migration paths away from GitHub. Not because of any single event; just a general feeling that things weren't looking promising, I believe. I did some digging, even flagged some concerns about uptime inconsistencies to my colleagues, and honestly expected it to blow over.

Now a few days later, I stumbled across a [video](https://youtu.be/d53Zk28esmU) from [Fireship](https://www.youtube.com/@Fireship) Jeff Delaney's channel, which I've been following for a while, where he goes over a lot of the same stuff I'd been reading/worried about. That's when it clicked.. this isn't just us being paranoid internally. When a platform this big starts showing cracks, _everyone_ notices, and everyone starts panicking, for good reason.

So here's a breakdown of what's actually going on.

## TABLE OF CONTENTS

## A Critical RCE Vulnerability (Fixed, but still alarming)

![Github Struggles](@assets/images/wiz-github-rce.webp)

Wiz Research recently disclosed **CVE-2026-3854**, a critical remote code execution flaw in GitHub's infrastructure. The scary part? Any authenticated user could exploit it with a single `git push` command.. no fancy tooling required, just a standard git client. [Read more here](https://www.wiz.io/blog/github-rce-vulnerability-cve-2026-3854)

The root cause was an injection flaw in GitHub's internal `X-Stat` header, which carries security-critical fields but wasn't properly sanitized. On GitHub.com, this exposed shared storage nodes, meaning **millions of public and private repos belonging to other users were potentially accessible**. And on GitHub Enterprise Server (GHES), it was even worse: full server compromise, including all hosted repos and internal secrets.

It's patched now (GHES versions 3.14.25 through 3.19.4), but the damage to trust? That takes longer to fix, I would assume.

One thing worth highlighting: this [vulnerability](https://nvd.nist.gov/vuln/detail/CVE-2026-3854) was reportedly discovered using AI, making it one of the first critical flaws found in closed-source binaries this way. Double-edged sword, as always!

## Reliability? [Not Great, Bob](https://youtu.be/MpUWrl3-mc8)

![Github Struggles](@assets/images/github-status-90d.png)

The security issue doesn't exist in a vacuum. GitHub has been struggling w/ [uptime and stability](https://www.infoq.com/news/2026/04/github-outages-scaling/) more broadly. Major [disruptions](https://github.blog/news-insights/company-news/addressing-githubs-recent-availability-issues-2/) hit on **February 2nd, February 9th, and March 5th** ~ and at one point in 2025, uptime apparently dropped [below 90%](https://www.theregister.com/2026/02/10/github_outages/). For a platform that hundreds of millions of developers depend on daily, that's... not that great.

GitHub acknowledged it _"failed to meet its own reliability standards,"_ citing tight service coupling (one failure cascades into many), lack of proper load-shedding, and underestimating the traffic surge from AI-driven usage. The Feb 9th incident, for example, was a database saturation issue · databases are notoriously harder to scale than stateless services, and GitHub simply didn't [see the load coming](https://blog.pragmaticengineer.com/the-pulse-is-github-still-best-for-ai-native-development/).

There's also a transparency problem. The community has had to rely on reconstructed, unofficial status feeds to understand what's happening. That lack of communication makes a bad situation worse.

## The AI Slop Tsunami

![Github Struggles](@assets/images/github-struggles.jpg)

Here's the one that's personally annoying me as someone who follows open source code maintainers. The absolute flood of AI-generated pull requests and issues hitting repositories has become overwhelming.

GitHub's own product manager put it plainly — maintainers are _"dedicating substantial time to reviewing contributions that do not meet project quality standards."_ These PRs often fail to follow guidelines, get abandoned shortly after submission, and are increasingly AI-generated. [Read more](https://www.infoworld.com/article/4127156/github-eyes-restrictions-on-pull-requests-to-rein-in-ai-based-code-deluge-on-maintainers.html)

The deeper problem, as one Microsoft Azure engineer [noted](https://www.theregister.com/2026/02/03/github_kill_switch_pull_requests_ai/): _"AI-generated PRs can look structurally fine but be logically wrong, unsafe, or interact with systems the reviewer doesn't fully know."_ Line-by-line review is still mandatory for shipped code, it just doesn't scale anymore.

GitHub is now considering a **kill switch for pull requests**, letting maintainers disable PRs entirely, along with more granular contributor controls. Ironically, it's also floating the idea of using AI to filter AI-generated submissions, which is... a choice.

And on top of this, **over 29 million secrets were leaked on GitHub in 2025**, with AI-assisted commits leaking credentials at roughly double the baseline rate. [Source](https://www.techradar.com/pro/security/over-29-million-secrets-were-leaked-on-github-in-2025-and-ai-really-isnt-helping)

## The Azure Migration: End of an Era

The big picture context here is GitHub's full migration to Microsoft Azure. GitHub CTO [Vladimir Fedorov](https://github.blog/author/vfedorovgh/) has essentially [told teams](https://www.techbuzz.ai/articles/github-s-azure-migration-signals-end-of-independence-era) to _"delay feature work to focus on moving GitHub"_, an aggressive 12-month timeline with a full exit from GitHub's own data centers planned within two years. The capacity needed? A **30x boost**.

The move is being described internally as "existential." Microsoft's Core AI division now has significant pull over GitHub's direction, and the platform's developer-first identity is — let's be honest — quietly fading. The $7.5 billion acquisition is finally showing its true colors, seven years in.

## So What Does This Mean For Developers?

GitHub isn't going anywhere. But the platform we grew up with, scrappy, indie-feeling, dev-first, is undergoing a fundamental transformation. It's becoming an AI platform that happens to host code, rather than a code platform that happens to have AI features.

Switching? Still painful for most teams. GitLab is the obvious alternative, but migration costs are real; pipelines, integrations, muscle memory. For now, most of us are staying put and hoping things stabilize.

But I'll say this: if your team is still treating GitHub as an unquestionable default without thinking about resilience and vendor lock-in, now's a good time to at least have that conversation.
