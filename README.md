# Best VPS for Docker: How Much RAM You Actually Need, the Fees That Aren't in the Headline Price, and Where RackNerd Fits

A Docker Compose stack with an app, Postgres, Redis and a reverse proxy needs 4 GB of RAM to run properly. A single-node K3s cluster wants 4 to 8 GB, and full kubeadm eats 2 GB per node before etcd has done anything useful. Those three numbers eliminate most of the guesswork, and they're the reason a 2 GB box that "looks fine" in a control panel falls over the first time you build an image.

RackNerd is where I'd point most people running containers on a budget: KVM virtualization with full root, SSD storage, a 1 Gbps port on every plan, and a one-click Docker plus Portainer install from its ctrl panel. The company rotates VPS specials and limits stock per region, so live pricing belongs on the order page rather than in a table that goes stale. What follows is the sizing math, the costs that don't appear in the headline price, and a straight answer on when a different provider is the better buy.

> **Quick pick:** one Compose stack with a database and a cache runs comfortably on 4 GB RAM and 2 vCPU. RackNerd's KVM VPS specials cover that without kernel restrictions or manual Docker setup, and you can have Portainer running from the control panel instead of the command line.
> 👉 [Check the current RackNerd VPS specials and live pricing](https://bit.ly/RacKnerd)

## How much RAM does a Docker VPS actually need?

Four gigabytes is the practical floor for anything with a database in it, and 8 GB is where production stacks with monitoring land. Everything below that is development territory.

| Workload | Minimum | Comfortable | What's running |
| --- | --- | --- | --- |
| Single container (dev/staging) | 1 GB | 2 GB | One service, no database |
| Docker Compose, 2–4 services | 2 GB | 4 GB | App + Postgres + Redis + reverse proxy |
| Docker Compose, 5+ services, production | 4 GB | 8 GB | Same stack plus monitoring |
| K3s, single node | 4 GB | 8 GB | K3s only, not kubeadm |
| K3s, control plane + 1 worker | 8 GB total | 12–16 GB total | Split across two VPS instances |

Two things trip people up here. The first is K3s marketing: Rancher documents a 1 GB minimum, which is enough to boot the thing and run nothing you care about. Real workloads want 4 GB, and 8 GB if you're running an ingress controller and a couple of stateful services alongside it. The second is that a container hitting its memory ceiling doesn't degrade gracefully. The kernel OOM killer terminates it, usually the process you least wanted to lose, and it does so without a warning you'll see before the downtime.

Full kubeadm Kubernetes is a different weight class: 2 GB per node just for the control plane components and etcd, before a single pod is scheduled. That math only works out when you're running five or more nodes and need a highly available control plane. Below that scale, K3s is the realistic option, and it's a CNCF-certified distribution with the same kubectl interface.

## What RackNerd gives you on every VPS special

The lineup is KVM-only, which matters more than any single spec on the page. OpenVZ-style containers don't hand you a real kernel, and Docker needs kernel namespaces, cgroup control and the ability to load modules. No amount of configuration fixes that.

| Item | Verified detail | Why it matters for containers |
| --- | --- | --- |
| Virtualization | KVM, full root access | Docker runs natively; no nested-container workarounds |
| Storage | SSD | The overlay2 driver generates heavy random I/O during image builds and layer extraction |
| Network port | 1 Gbps on every plan | Base image pulls and registry syncs are bandwidth-bound |
| IP addresses | 1 IPv4 included | Bind containers to a public address or put a reverse proxy on 80/443 |
| Docker setup | One-click Docker with Portainer from ctrl, or install over SSH | Skips the GPG key and repo dance entirely |
| Control panel | ctrl: reboot, OS reinstall, HTML5 VNC console, one-click apps, network firewall, root password reset, upgrades | Recovery and OS swaps without a support ticket |
| Management | Unmanaged, 24/7 support | You get root and a panel, not a sysadmin |

The one-click catalog doubles as a sizing guide, and it's more concrete than most vendor documentation. RackNerd publishes memory requirements per app: n8n and Coolify are comfortable from 2 GB, while Ollama, LibreChat, AnythingLLM and Hermes Agent want at least 4 GB. OpenClaw wants 8 GB. If the thing you actually plan to run is on that list, its stated requirement is a better starting point than any general rule about container density.

What you don't get is management. Patching, firewall rules and backups are yours to handle, which is the standard trade at this price level. Teams without anyone to own that work tend to be happier on a managed container platform, where the per-unit compute cost is higher but the operational surface is close to zero.

👉 [See which RackNerd VPS specials are in stock for your region](https://bit.ly/RacKnerd)

## Three costs that aren't in the headline price

**Egress.** Pull a 1.5 GB base image thirty times during a day of CI work and you've moved 45 GB, none of which felt like "traffic" at the time. Hetzner Cloud includes 20 TB per month and bills per TB beyond that. DigitalOcean charges $0.01 per GiB above the plan's included transfer. Contabo runs a flat-rate model with unlimited outgoing traffic under fair use. Cloudzy includes egress in the monthly allotment and doesn't charge extra for snapshots. If you run a private registry or a multi-node cluster, node-to-node traffic lands on the same meter.

**Renewal versus introductory pricing.** Contabo's own price table quotes Core VPS 4 at €5.24/mo on a 12-month subscription with taxes included, verified in August 2026, while third-party roundups still list the brand at $13.00/mo because they're quoting a different configuration or term. Hetzner's Cloud prices exclude VAT. DigitalOcean bills monthly with a cap. Normalize before comparing: tax in or out, term length, and whether the number you're looking at is a first-term rate or the renewal rate. Two plans that look €5 apart can reverse order once you fix those three variables.

**Hourly billing versus monthly caps.** Hetzner and DigitalOcean both bill by the hour with a monthly cap, so a box you keep for a year never costs more than the listed monthly rate, and a box you keep for three days costs three days' worth. LightNode bills Docker VPS from $0.012 per hour. Cloudzy offers hourly, monthly and annual terms. RackNerd's specials are built around prepaid annual billing, which is why the effective monthly figure looks the way it does. For a permanent host, prepaid annual usually wins; for a migration weekend, hourly wins by a wide margin.

## Don't put Docker and K3s on the same machine

They use different container runtimes. K3s ships with containerd, Docker uses dockerd, and running both on one box means two image stores, two sets of network rules and two processes competing for the same RAM. If you want a Compose stack and a small cluster, buy two VPS instances. It's cheaper than the debugging time.

K3s also bundles Traefik as its ingress controller and CoreDNS by default. Installing a second ingress on top is a common first mistake and produces port conflicts that look like application bugs.

For Compose on a single host, set memory limits per service so one runaway process can't take out its neighbours:

yaml
deploy:
  resources:
    limits:
      memory: 512m


Leave headroom on top of those limits. A 4 GB machine running 3.5 GB of containers is one image pull away from the OOM killer choosing which of your services dies.

## When a different provider is the better answer

The numbers below are as published by each provider, with the qualifiers they attach to them.

Hetzner Cloud is the strongest option if your users are in Europe: CPX22 at 2 vCPU / 4 GB / 80 GB SSD is €19.49/mo, CPX32 at 4 vCPU / 8 GB / 160 GB is €35.49/mo, CPX42 at 8 vCPU / 16 GB / 320 GB is €69.49/mo, all excluding VAT in the EU region, with 20 TB of egress included. The tooling and community templates around it are better than most.

DigitalOcean is the choice when you want managed Kubernetes without assembling it: a 2 vCPU / 4 GB / 80 GB Droplet is $24/mo, 4 vCPU / 8 GB / 160 GB is $48/mo, billed per second with a monthly cap, and DOKS is available as an add-on. You pay more per gigabyte of RAM than at Hetzner, and the documentation and ecosystem are the reason people do it.

Contabo wins on RAM per euro by a wide margin: Core VPS 4 at 4 vCPU / 8 GB / 200 GB SSD for €5.24/mo, Core VPS 6 at 6 vCPU / 12 GB / 400 GB for €7.14/mo, Core VPS 8 at 8 vCPU / 24 GB / 400 GB for €13.33/mo, all on a 12-month term with taxes included, flat-rate egress. If you need 24 GB of RAM cheaply, that's the plan to price against.

Vultr starts at $2.50/mo with 30+ locations and up to 60 TB of transfer, which makes it the pick when latency to a specific region drives the decision. LightNode's Docker VPS starts at $7.71/mo or $0.012/hour across 40+ locations with Docker pre-installed. Cloudzy starts at $2.48/mo across 13 regions with egress included.

RackNerd's case rests on the combination rather than a single headline spec: KVM with full root, a 1 Gbps port, one-click Docker plus Portainer, and a control panel that handles reinstalls and recovery without a ticket, all on prepaid specials. If you need a managed control plane, a European latency guarantee or 24 GB of RAM in one box, one of the names above is the honest answer.

## FAQ

**Is 2 GB enough to run Docker?**
For one or two small containers, yes. For Compose with a database and a cache, it's tight enough that a build will trigger the OOM killer. RackNerd's own app guidance puts n8n and Coolify at 2 GB minimum, which is a reasonable proxy for a single lightweight service.

**Do I need KVM for Docker?**
Yes. Docker needs kernel namespaces, cgroup control and the ability to manipulate iptables. OpenVZ-style virtual containers don't give you a real kernel, and the restrictions can't be worked around from inside. RackNerd's entire VPS lineup is KVM.

**Why can't I run Docker on shared hosting?**
Shared plans restrict the kernel features Docker depends on and cap resources per account, so multiple containers end up fighting each other. It's a structural limit, not a permissions setting you can change.

**How much RAM does K3s need?**
Rancher documents 1 GB as the minimum to run K3s itself. Budget 4 GB for anything with real workloads, and 8 GB for a node running an ingress controller plus stateful services.

**What happens if I go over my transfer allowance?**
It depends entirely on the provider. Hetzner bills per TB beyond 20 TB, DigitalOcean bills $0.01 per GiB beyond the included allowance, and Contabo's flat-rate model doesn't meter it under fair use. Ask before you start syncing a registry between nodes.

## Which one to order

A single container or a development box: 2 GB RAM, 1 to 2 vCPU. A Compose stack with a database and a cache: 4 GB and 2 vCPU, which is where most readers land. Production Compose with monitoring, or a single-node K3s: 8 GB and 4 vCPU. Two-node K3s: two separate instances, 8 GB total at the floor and 12 to 16 GB to be comfortable.

Two things to confirm before you pay. Check that the plan is KVM with full root, since a cheap OpenVZ offer will waste your afternoon. Then check whether the price you're looking at is a first-term rate or a renewal rate, and whether tax is included, because that's where the comparison between two "similar" plans usually flips.

👉 [Compare RackNerd's current KVM VPS specials and pick your region](https://bit.ly/RacKnerd)
