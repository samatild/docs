---
title: Deploy Hermes Agent in a Proxmox LXC with Community Scripts
description: Deploy Hermes Agent in a Proxmox LXC using the Community Scripts installer, then configure and verify a minimal working setup.
date: 2026-10-07
type: docs
author: Samuel Matildes
tags: [linux, proxmox, lxc, hermes, ai-agent, homelab, automation]
aliases:
  - /linux/admin/hermes-agent-proxmox-lxc/
---

<i class="fas fa-cube" aria-hidden="true"></i> A small, repeatable Hermes Agent deployment for a Proxmox VE container.

{{< callout type="warning" title="Community-maintained installer" >}}
This installer is maintained by the Community Scripts project, not by Nous Research. It downloads and runs code from external sources. Review the current script and use it only on a Proxmox host you administer.
{{< /callout >}}

## Before you start

Have the following ready:

- A Proxmox VE host with capacity for a new LXC container.
- A current backup or recovery plan for the host and any affected storage.
- Outbound HTTPS access from the host and the new container.
- A model-provider account or an approved OpenAI-compatible endpoint. Complete provider authentication only in Hermes' interactive setup; do not put credentials in shell history or documentation.

The Community Scripts page currently lists a Debian 13 profile with 2 vCPUs, 4 GiB of RAM, and 20 GiB of disk. Treat these as installer defaults, not sizing guidance. Select the advanced installer flow when your workload needs different limits.

## Create the container

1. In the Proxmox VE shell, review the installer source linked from the Community Scripts page.
2. Run the installer:

```bash
bash -c "$(curl -fsSL https://raw.githubusercontent.com/community-scripts/ProxmoxVE/main/ct/hermesagent.sh)"
```

3. Follow the prompts to create the LXC, then start it.

The script creates a dedicated `hermes` service user. The container is a useful isolation boundary, but Hermes can execute commands inside it. Do not treat it as safe to expose directly to an untrusted network.

## Configure Hermes

Open the LXC console and switch to the service user:

```bash
su - hermes
hermes setup
```

Use the setup wizard to select and authenticate a provider. Configure one working model first; defer gateways, messaging integrations, and other automation until a normal chat succeeds.

## Verify the deployment

Still as the `hermes` user, run:

```bash
whoami
hermes --version
hermes doctor
hermes chat -q "Reply with exactly: Hermes is ready."
```

Expected results:

- `whoami` returns `hermes`.
- `hermes --version` prints an installed version.
- `hermes doctor` completes without a blocking configuration error.
- The chat command receives the requested response from the configured model.

If the chat test fails, run `hermes model` to review the selected provider and model, then repeat the test. Do not add a dashboard, gateway, or messaging channel until this baseline works.

## Access the dashboard safely

The Community Scripts page documents a dashboard on the container's loopback interface. Reach it through an SSH tunnel instead of publishing the dashboard port:

```bash
ssh -NL 9119:localhost:9119 <administrator>@<container-host>
```

Open `http://localhost:9119` on the machine running the tunnel. Replace both placeholders with values appropriate to your environment. Keep the dashboard private unless you have separately designed and reviewed its authentication and network controls.

## Troubleshooting

| Symptom | Check |
| --- | --- |
| `hermes: command not found` | Start a new shell after `su - hermes`, then confirm the installer completed successfully. |
| `hermes doctor` reports a provider or model problem | Re-run `hermes setup` or `hermes model` as the `hermes` user and complete the provider configuration. |
| The dashboard does not load through the tunnel | Confirm the tunnel remains open and verify the container is running before changing firewall or exposure settings. |

## Sources

- [Community Scripts: Hermes Agent](https://community-scripts.org/scripts/hermesagent)
- [Community Scripts Hermes Agent installer source](https://github.com/community-scripts/ProxmoxVE/blob/main/ct/hermesagent.sh)
- [Hermes Agent installation documentation](https://hermes-agent.nousresearch.com/docs/getting-started/installation)
- [Hermes Agent quickstart](https://hermes-agent.nousresearch.com/docs/getting-started/quickstart)

## Related reading

- [How to Troubleshoot Linux Performance — Field Playbook](/linux/admin/linux-performance-playbook/) for investigating container or host resource pressure.
