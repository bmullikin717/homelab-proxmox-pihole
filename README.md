# Home Network DNS Filtering with Proxmox VE + Pi-hole

A self-hosted virtualization lab built on repurposed hardware, running a network-wide DNS-based ad/tracker blocker as its first production service. Built as a hands-on complement to CompTIA A+ study, with a focus on real troubleshooting, safe change management on a shared network, and documentation practices used in professional IT environments.

## What This Project Does

- Converted a retired laptop into a **Proxmox VE** hypervisor host
- Deployed **Pi-hole** in a lightweight LXC container as a network-wide DNS filter
- Rolled out DNS filtering to every device on the home network via router-level DHCP configuration
- Planned and executed the rollout around a real constraint: a housemate's remote-work VPN connection, with zero disruption

## Architecture

```
Internet
   |
Router (DHCP + Gateway)
   |
1GbE Switch
   |
   +-- Proxmox VE Host (repurposed laptop)
   |      |
   |      +-- LXC Container: Pi-hole (static IP, DNS filtering)
   |
   +-- All other network devices (desktop, phone, IoT, TVs)
          DNS requests -> Pi-hole -> filtered / forwarded to Cloudflare (1.1.1.1)
```

## Tech Stack

| Layer | Tool |
|---|---|
| Hypervisor | Proxmox VE 9.2 (Debian 13 base) |
| Guest OS | Debian 13 (LXC container) |
| DNS Filtering | Pi-hole |
| Upstream DNS | Cloudflare (1.1.1.1) |
| Hardware | Repurposed laptop — Intel i5-5300U, 12GB RAM, 465GB SSD |

## Notable Problems Solved

Rather than a clean, linear build, this project involved real diagnostic work — a few highlights:

**IP address conflict masquerading as a DNS/service failure.** After getting Proxmox on the correct subnet, the web GUI was still unreachable — `ping` succeeded but the actual service connection was refused. Traced it down to a MAC address mismatch in the ARP table, revealing another device on the network was already using the target IP. Resolved by identifying a genuinely free address and confirming via ARP before committing to it.

**Silent DNS misconfiguration after a subnet change.** After correcting a wrong IP/gateway configuration, package updates still failed. Root cause: DNS resolution (`/etc/resolv.conf`) is configured independently of IP/gateway settings on Debian-based systems and doesn't automatically follow other network changes — a subtle but important distinction between Layer 3 connectivity and name resolution.

**Modern browser DNS-over-HTTPS silently bypassing the filter.** After deployment, Pi-hole showed zero blocked queries despite normal browsing. Traced to Chrome's built-in Secure DNS (DoH) setting, which performs DNS lookups independently of OS-level configuration — a good example of a non-obvious, current-day networking gotcha that wouldn't show up in older study material.

**Shared-network rollout without service disruption.** Before making a network-wide DNS change, investigated whether a housemate's VPN-connected work device would be affected. Found their VPN already pushes its own DNS servers, making a manual exclusion unnecessary — a case of gathering real data before defaulting to the more complex, "safe" solution.

## Skills Demonstrated

- Type-1 hypervisor administration (Proxmox VE)
- Linux system administration (Debian, LXC containers, systemd services)
- Networking fundamentals: subnetting, DHCP, DNS, ARP, static IP addressing
- Layered network troubleshooting (physical → IP → DNS → application)
- Change management on a shared, production-adjacent network
- Technical documentation

## What's Next

This is the first service in an ongoing homelab build. Planned next: a self-hosted password manager (Vaultwarden), followed by an Active Directory domain controller for hands-on directory services and Group Policy experience.

## Full Build Log

A detailed, phase-by-phase build log — including every troubleshooting step, command run, and decision made along the way — is available in [`BUILDLOG.md`](./BUILDLOG.md).
