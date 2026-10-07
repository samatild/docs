---
title: Documentation
linkTitle: Docs
menu: {main: {weight: 20}}
type: docs
---

<section class="kb-hero" data-hero-image="true">
  <h1 class="kb-hero__title">Production troubleshooting &amp; automation</h1>
  <p class="kb-hero__subtitle">
    Field notes, diagnostic playbooks, performance tooling, and practical
    infrastructure patterns for Linux, Windows, Azure, Kubernetes, and AI.
    Production lessons first — theory only when it helps fix the box in front
    of you.
  </p>
  <div class="kb-hero__cta">
    <a class="btn btn-primary" href="#featured-guides"><i class="fas fa-compass" aria-hidden="true"></i>&nbsp; Browse featured guides</a>
    <a class="btn btn-outline-light" href="/ai/honcho-litellm-github-copilot/"><i class="fas fa-brain" aria-hidden="true"></i>&nbsp; Latest guide</a>
    <a class="btn btn-outline-light" href="https://github.com/samatild" target="_blank" rel="noopener"><i class="fab fa-github" aria-hidden="true"></i>&nbsp; GitHub</a>
  </div>
</section>

## Featured guides {#featured-guides}

Start with the field guides and architecture patterns that best represent the
site today.

<div class="kb-grid kb-grid--guides">
{{< feature-card title="Linux Performance — Field Playbook" icon="fa-gauge-high" href="/linux/admin/linux-performance-playbook/" desc="A practical workflow for investigating Linux performance from symptoms to evidence." >}}
{{< feature-card title="Self-Host Honcho with LiteLLM" icon="fa-brain" href="/ai/honcho-litellm-github-copilot/" desc="Persistent agent memory, vector retrieval, and provider-neutral model routing on Kubernetes." >}}
{{< feature-card title="MicroK8s with Traefik" icon="fa-dharmachakra" href="/kubernetes/microk8s-simple-implementation/" desc="A practical private and public application deployment pattern for a single-node cluster." >}}
</div>

## Browse by topic

<div class="kb-grid">
{{< feature-card title="Linux"                icon="fa-linux"        iconStyle="brands" href="/linux/"                desc="Kernel internals, performance, diagnostics, and admin tooling." >}}
{{< feature-card title="Windows"              icon="fa-windows"      iconStyle="brands" href="/windows/"              desc="Event logs, administration workflows, and practical Windows tooling." >}}
{{< feature-card title="Azure"                icon="fa-microsoft"    iconStyle="brands" href="/azure/"                desc="VM troubleshooting, serial console, and profiling on Azure." >}}
{{< feature-card title="Kubernetes"           icon="fa-dharmachakra"                    href="/kubernetes/"           desc="Cluster recipes, ingress, and practical application deployment patterns." >}}
{{< feature-card title="AI"                    icon="fa-brain"                           href="/ai/"                   desc="Self-hosted agents, durable memory, and model gateways." >}}
{{< feature-card title="Software Engineering" icon="fa-code"                            href="/software-engineering/" desc="Patterns, optimisation, and lessons from production code." >}}
</div>

## Tools I build

Open-source tools built from recurring production troubleshooting problems.

<div class="kb-tools">
{{< tool-card name="LinuxAiOPerf"
              category="Performance collection"
              image="/images/tools/linuxaioperf.png"
              repo="https://github.com/samatild/LinuxAiOPerf"
              desc="All-in-one Linux performance data collector. One command, one timestamped output directory." >}}

{{< tool-card name="Tux Toaster"
              category="Benchmarking"
              image="/images/tools/tuxtoaster.png"
              repo="https://github.com/samatild/tuxtoaster"
              desc="Stress-test and benchmark CPU, memory, disk and network from a friendly TUI." >}}

{{< tool-card name="SOSParser"
              category="Support analysis"
              image="/images/tools/sosparser.png"
              repo="https://github.com/samatild/SOSParser"
              desc="Turn sosreport / supportconfig archives into interactive HTML reports." >}}

{{< tool-card name="evtxparser"
              category="Event log parsing"
              image="/windows/admin/evtxparser/images/evtxparser.png"
              repo="https://github.com/samatild/evtxparser"
              href="/windows/admin/evtxparser/"
              desc="Export Windows .evtx event logs to CSV with a fast, streaming CLI." >}}
</div>

## Recently updated

<div class="kb-update-list">
  <a class="kb-update" href="/ai/honcho-litellm-github-copilot/">
    <i class="fas fa-brain" aria-hidden="true"></i>
    <span><strong>Honcho + LiteLLM on Kubernetes</strong><small>Persistent agent memory, pgvector, Redis cache, and model routing.</small></span>
  </a>
  <a class="kb-update" href="/kubernetes/microk8s-simple-implementation/">
    <i class="fas fa-dharmachakra" aria-hidden="true"></i>
    <span><strong>MicroK8s with Traefik</strong><small>Private and public application deployment patterns.</small></span>
  </a>
  <a class="kb-update" href="/linux/admin/linux-performance-playbook/">
    <i class="fas fa-gauge-high" aria-hidden="true"></i>
    <span><strong>Linux Performance — Field Playbook</strong><small>A systematic path from production symptoms to useful evidence.</small></span>
  </a>
</div>

## About this site

Built with [Hugo](https://gohugo.io/) and the [Docsy](https://www.docsy.dev/)
theme. Content is continuously updated; use the navigation menu or search to
find a specific guide.
