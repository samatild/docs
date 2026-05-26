---
title: Documentation
linkTitle: Docs
menu: {main: {weight: 20}}
type: docs
---

<section class="kb-hero" data-hero-image="true">
  <h1 class="kb-hero__title">Samuel Matildes — Knowledge Base</h1>
  <p class="kb-hero__subtitle">
    Field notes, playbooks, and tools from production work across Linux,
    Windows, Azure, and Kubernetes. Practical content first — theory only when
    it helps you fix the box in front of you.
  </p>
  <div class="kb-hero__cta">
    <a class="btn btn-primary" href="/linux/"><i class="fab fa-linux"></i>&nbsp; Explore Linux</a>
    <a class="btn btn-outline-light" href="https://github.com/samatild" target="_blank" rel="noopener"><i class="fab fa-github"></i>&nbsp; GitHub</a>
  </div>
</section>

## Browse by topic

<div class="kb-grid">
{{< feature-card title="Linux"                icon="fa-linux"        iconStyle="brands" href="/linux/"                desc="Kernel internals, performance, diagnostics, admin tooling." >}}
{{< feature-card title="Windows"              icon="fa-windows"      iconStyle="brands" href="/windows/"              desc="Event logs, administration workflows, and practical Windows tooling." >}}
{{< feature-card title="Azure"                icon="fa-microsoft"    iconStyle="brands" href="/azure/"                desc="VM troubleshooting, serial console, profiling on Azure." >}}
{{< feature-card title="Kubernetes"           icon="fa-dharmachakra"                    href="/kubernetes/"           desc="Hands-on cluster recipes and walk-throughs." >}}
{{< feature-card title="Software Engineering" icon="fa-code"                            href="/software-engineering/" desc="Patterns, optimisation, lessons from production code." >}}
</div>

## Tools I build

Open-source companions to the content on this site.

<div class="kb-tools">
{{< tool-card name="LinuxAiOPerf"
              image="/images/tools/linuxaioperf.png"
              repo="https://github.com/samatild/LinuxAiOPerf"
              desc="All-in-one Linux performance data collector. One command, one timestamped output directory." >}}

{{< tool-card name="Tux Toaster"
              image="/images/tools/tuxtoaster.png"
              repo="https://github.com/samatild/tuxtoaster"
              desc="Stress-test and benchmark CPU, memory, disk and network from a friendly TUI." >}}

{{< tool-card name="SOSParser"
              image="/images/tools/sosparser.png"
              repo="https://github.com/samatild/SOSParser"
              desc="Turn sosreport / supportconfig archives into interactive HTML reports." >}}

{{< tool-card name="evtxparser"
              image="/windows/admin/evtxparser/images/evtxparser.png"
              repo="https://github.com/samatild/evtxparser"
              href="/windows/admin/evtxparser/"
              desc="Export Windows .evtx event logs to CSV with a fast, streaming CLI." >}}
</div>

## About this site

Built with [Hugo](https://gohugo.io/) and the [Docsy](https://www.docsy.dev/)
theme. Content is continuously updated — explore via the navigation menu, the
search box, or jump straight into the latest field playbook:
**[Linux Performance — Field Playbook](/linux/admin/linux-performance-playbook/)**.

