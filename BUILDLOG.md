# Homelab Build Log: Proxmox VE + Pi-hole

**Author:** [Your Name]
**Project Start Date:** July 27, 2026
**Status:** Complete — live in production on home network

---

## Project Overview

This project documents the setup of a home virtualization lab using **Proxmox VE** (a type-1 bare-metal hypervisor) repurposed on a laptop, with **Pi-hole** deployed as a network-wide DNS ad-blocker as the first hosted service. This is intended as a foundational project for building broader IT infrastructure skills and as a portfolio piece demonstrating hands-on virtualization, networking, and Linux administration experience.

## Objectives

- Learn and apply core virtualization concepts (type-1 vs. type-2 hypervisors, VMs vs. containers)
- Gain hands-on experience with Proxmox VE installation and administration
- Understand DNS fundamentals by deploying Pi-hole as a network-wide ad-blocking DNS server
- Practice foundational networking concepts (DHCP, DNS, IP addressing) relevant to CompTIA A+
- Build a documented, repeatable process suitable for a professional portfolio/resume

## Skills / A+ Objectives Demonstrated

- Virtualization fundamentals (hypervisor types, resource allocation, CPU virtualization extensions)
- Basic Linux system administration
- Networking concepts: DNS, DHCP, IP addressing, network topology
- Hardware evaluation and BIOS/UEFI configuration
- System documentation and change-log practices

---

## Hardware & Software Used

| Component | Spec |
|---|---|
| Device | Repurposed laptop (Windows 10, converted to Proxmox host) |
| CPU | Intel i5-5300U @ 2.4GHz (Broadwell, 2C/4T, VT-x capable) |
| RAM | 12GB DDR3 |
| Storage | 465GB SSD |
| Network | Onboard Ethernet NIC → 1GB switch → home router/modem |
| Hypervisor | Proxmox VE 9.2 (Debian 13 "Trixie" base) |
| First Service | Pi-hole (deployed in LXC container) |

---

## Build Log

### Phase 1: Hardware & BIOS Preparation

**Date:** [fill in]

**Goal:** Confirm the laptop supports hardware virtualization and prepare it to boot from USB installer media.

**Steps taken:**
- [x] Entered BIOS/UEFI setup (ThinkPad: tap Enter at boot, then F1 for Setup)
- [x] Confirmed Intel VT-x (Virtualization Technology) is enabled — found under **Security > Virtualization**
- [x] Noted Boot Mode (UEFI vs. Legacy/CSM): was originally set to **"Both"** — changed to **UEFI Only** (found under **Startup** tab)
- [x] Noted Secure Boot status: already **Disabled** — left as-is
- [x] Saved changes with F10

**Notes / Issues encountered:**
- After changing Boot Mode from "Both" to "UEFI Only" and rebooting, the laptop failed to boot into Windows 10 automatically and instead dropped to a raw boot device list (`ATA HDD0: CRUCIAL...` and `PCI LAN`). This happened because the existing Windows 10 install was likely set up under Legacy/MBR, and switching the firmware to UEFI-only meant it could no longer find a valid UEFI bootloader for Windows.
- **Resolution:** Not an actual problem — since the laptop is being wiped for Proxmox anyway, Windows' boot state doesn't matter. No fix needed; proceeded straight to installer creation. Good real-world lesson in Legacy/MBR vs. UEFI/GPT boot mismatches.

**Why this step matters:**
Proxmox VE requires a CPU with hardware virtualization extensions (Intel VT-x or AMD-V) to run VMs efficiently. Without it enabled in BIOS, the hypervisor either won't function correctly or will fall back to much slower software emulation. This is also directly testable A+ knowledge — understanding hypervisor CPU requirements, and the practical difference between Legacy/MBR and UEFI/GPT boot standards.

---

### Phase 2: Creating Bootable Installer Media

**Date:** [fill in]

**Goal:** Create a bootable USB drive containing the Proxmox VE 9.2 installer.

**Tools used:** Rufus, run from a Windows 11 desktop (separate machine from the ThinkPad being converted)

**Steps taken:**
- [x] Downloaded Proxmox VE 9.2 ISO from the official Proxmox downloads page
- [x] Repurposed an existing USB drive that previously had a Kali Linux live image on it (fully overwritten — no data preserved)
- [x] Opened Rufus, selected the correct USB device (verified size/label before writing, since desktop had its own internal drives to avoid mistaking)
- [x] Selected the Proxmox ISO as boot selection
- [x] Rufus detected the Proxmox ISO as an **ISO hybrid image** and required **DD mode** instead of the usual "ISO Image mode"
- [x] Wrote the image in DD mode
- [x] Confirmed "READY" status in Rufus with no errors

**Notes / Issues encountered:**
- Rufus forced **DD (raw) mode** instead of ISO mode because Proxmox ships its installer as a hybrid ISO with a partition layout already baked in. This overrides manual Partition Scheme / Target System selections in Rufus — that's expected and correct for Proxmox specifically (not the case for most other Linux ISOs).
- After writing, Windows showed the USB with **two partitions**, one of which appeared inaccessible/"RAW" to Windows. This is normal — DD mode replicates the ISO's native structure, which includes a small EFI boot partition and a larger installer partition. Windows doesn't need to read these; the ThinkPad's UEFI firmware reads them directly. **Did not format or modify the drive** despite Windows' prompt to do so.

**Why this step matters:**
Understanding ISO hybrid images and the difference between ISO mode vs. DD (raw) mode in imaging tools is directly relevant to real-world OS deployment work, and explains why some distros need special handling versus the "default" write mode most guides assume.

---

### Phase 3: Proxmox VE Installation

**Date:** [fill in]

**Goal:** Install Proxmox VE onto the laptop's SSD, wiping Windows 10.

**Steps taken:**
- [x] Booted ThinkPad from USB (Enter, then F12 for boot menu, selected USB device)
- [x] Chose **Graphical Install** at the boot options screen (over Terminal Install — functionally identical, graphical is easier to document/screenshot and matches standard documentation)
- [x] Accepted EULA
- [x] **Target Harddisk:** selected the Crucial SSD (only disk present); confirmed filesystem set to **ext4** (ZFS considered and ruled out — ZFS's redundancy features only make sense with multiple disks; single-disk setup calls for ext4)
- [x] Connected ThinkPad to the 1GB switch via Ethernet **before** reaching network configuration (required since Proxmox needs a detected NIC to configure network settings during install, and the box will be managed remotely afterward — no monitor/keyboard long term)
- [x] Set Country/Timezone/Keyboard to match actual location
- [x] Set root password (generated via password manager, saved there) and admin notification email
- [x] **Network Configuration:**
  - Two management interface options appeared: `nic0` (MAC `68:f7:28:e7:60:35`, driver `e1000e`) and `nic1` (MAC `5c:e0:c5:47:ec:30`, driver `iwlwifi`)
  - Identified `nic0/e1000e` as the physical wired Ethernet port and `nic1/iwlwifi` as the Wi-Fi card — selected **nic0**, since Wi-Fi isn't suitable for a Proxmox management interface
  - IP Address: `192.168.100.2/24`
  - Gateway: `192.168.100.1`
  - DNS Server: `192.168.100.1`
  - Hostname (FQDN): changed installer's placeholder `pve.example.invalid` to **`pve.homelab.lan`** (using informal `.lan` convention for local-only naming)
- [x] Reviewed final Summary screen, confirmed all settings (disk/filesystem, timezone, management interface, IP config, hostname)
- [x] Clicked **Install**

**Notes / Issues encountered:**
- Installer listed two NICs (wired + Wi-Fi) rather than just one; had to identify the correct one by driver name (`e1000e` = wired, `iwlwifi` = Wi-Fi) rather than assuming the first option listed was correct.
- Noted for Phase 4 follow-up: need to check the router's DHCP range to confirm `192.168.100.2` falls outside the pool it hands out automatically, to avoid a future IP conflict. If it's inside the DHCP range, will either reserve it by MAC address in the router or reassign Proxmox to an address outside the pool.

**Why this step matters:**
This phase covers disk partitioning/filesystem choice (ext4 vs. ZFS — relevant to A+ storage objectives), static vs. DHCP IP addressing for a server that needs a predictable address, and identifying network interfaces by driver/hardware ID rather than guessing — all core sysadmin/help-desk skills.

---

### Phase 4: Post-Install Configuration

**Date:** [fill in]

**Goal:** Access the Proxmox web GUI, apply updates, and configure repositories correctly.

#### Sub-section: Post-Install Network Troubleshooting (Case Study)

**Goal:** Get first access to the Proxmox web GUI (`https://<IP>:8006`) from a separate desktop machine.

**What went wrong (chronological):**

1. **Initial reboot** — ThinkPad booted successfully into Proxmox, console screen displayed an IP and login prompt as expected.
2. **First browser attempt failed** ("site can't be reached") using the IP configured during install (`192.168.100.2`).
3. **Root cause #1 — wrong subnet entirely:** Checked desktop's own IP via `ipconfig` and found it was on `192.168.88.x`, not `192.168.100.x`. The installer's "auto-detected" network settings during setup did not match the actual home network — likely grabbed a stale/incorrect DHCP response at install time. Confirmed via `ping` that `192.168.100.2` was unreachable from the desktop for this reason.
4. **Fix attempt #1:** Edited `/etc/network/interfaces` on the ThinkPad, changed the `vmbr0` static IP/gateway from the `192.168.100.x` range to `192.168.88.2/24` (matching the real network), applied with `ifreload -a`. Ping succeeded after this change — confirmed basic network connectivity.
5. **Browser still failed** after fixing the subnet — got `ERR_CONNECTION_REFUSED` specifically (not a timeout), which is a different, more specific error.
6. **Diagnostic steps to isolate cause:**
   - Confirmed `pveproxy` (the Proxmox web service) was active/running via `systemctl status pveproxy`
   - Checked port binding with `ss -tlnp | grep 8006` (note: `netstat` isn't installed by default on Proxmox 9/Debian 13 — had to use the modern `ss` equivalent instead) — confirmed the service was correctly listening on all interfaces, port 8006
   - Confirmed Proxmox's built-in firewall was disabled (`pve-firewall status`) — not the cause
   - Ran `curl -k https://localhost:8006` **on the ThinkPad itself** — returned valid HTML, proving the web service worked correctly locally. This confirmed the problem was somewhere on the network path, not in Proxmox's configuration.
   - Tested from a second device (phone on same Wi-Fi) — also failed, ruling out desktop-specific causes (Windows Firewall, antivirus, etc.)
   - Checked router for Client/AP Isolation setting — found disabled, not the cause
7. **Root cause #2 — IP address conflict:** Ran `arp -a` on the desktop and compared the MAC address returned for `192.168.100.2`/`192.168.88.2` against the ThinkPad's actual NIC MAC (`68:f7:28:e7:60:35`, noted earlier during install). **They didn't match** — another device on the network was already using that address, meaning all traffic had been going to the wrong device the entire time (explaining why `ping` worked in some cases but the actual Proxmox service was never reached).
8. **Fix attempt #2:** Checked router's DHCP pool range (`192.168.88.2–249`, covering nearly the whole subnet) and shrank it to `192.168.88.50–249` to free up address space for static assignments. Reassigned the ThinkPad to `192.168.88.10`, applied via `/etc/network/interfaces` + `ifreload -a`.
9. **False alarm:** After the change, `ping 192.168.88.10` still failed, and `arp -a` showed a MAC address that didn't initially match the ThinkPad — appeared to be *yet another* conflict. Cleared the ARP cache (`arp -d *`) and re-tested; the MAC address then correctly matched the ThinkPad. **Conclusion: this was a stale/cached ARP entry from the rapid IP changes made during troubleshooting, not a real conflict.**
10. **Resolution:** With ARP resolving correctly, tried the web GUI directly (bypassing ping, since ICMP and TCP/HTTPS can behave differently depending on host firewall rules) at `https://192.168.88.10:8006` — **successful connection**, reached the Proxmox login page.

**Key lessons learned:**
- Don't trust an installer's auto-detected network settings blindly — always verify against the actual device (`ipconfig`/`ip a`) rather than assuming they match.
- `ping` succeeding does not guarantee a specific service/port is reachable — ICMP and TCP behave independently, and a "connection refused" vs. a "timeout" are meaningfully different errors that point to different problems.
- Always check the ARP table (`arp -a`) when IP connectivity seems to succeed but the expected service doesn't respond — a MAC address mismatch is a fast, definitive way to catch IP conflicts.
- ARP caches can go stale and produce misleading results during periods of rapid network reconfiguration — clear the cache (`arp -d *`) before trusting a "conflict" reading.
- DHCP pool ranges only control automatic assignment — they don't restrict the subnet itself, and static IPs can coexist with them (though picking addresses outside the active pool avoids future conflicts).
- `netstat` is deprecated on modern Debian-based systems (including Proxmox VE 9); use `ss` instead.
- Systematic isolation (test locally via `curl` on the host itself → test from a second independent client device → check firewall/isolation settings → check for IP-level conflicts) is far more effective than guessing — each test eliminated an entire category of possible causes.

**Why this matters:**
This entire sequence is a realistic troubleshooting scenario covering IP addressing/subnetting, DHCP behavior, ARP resolution, and the difference between network-layer (ICMP) and transport-layer (TCP) connectivity — all core CompTIA A+ and Network+ objectives. Working through it hands-on, rather than just reading about it, is exactly the kind of experience worth highlighting on a resume or in an interview.

---

**Steps taken (remainder of Phase 4 — repository config, updates):**
- [x] Successfully logged into the Proxmox web GUI at `https://192.168.88.10:8006`
- [x] Disabled both enterprise repositories (`pve-enterprise` and the `ceph` enterprise repo) via **Node → Updates → Repositories** — confirmed via GUI showing "Enabled: false" for both
- [x] Added the free **no-subscription** repository via the "Add" button — confirmed GUI status changed to "You get updates for Proxmox VE" (green). The remaining "no-subscription repository is not recommended for production use" warning is expected/standard for any non-paying install and was left as-is — informational only, not an actual misconfiguration
- [x] Ran `apt update && apt full-upgrade -y` via the in-browser Shell

**Notes / Issues encountered — follow-up DNS bug from earlier subnet fix:**
- The update command hung at 0%, repeatedly showing "Ign" (ignored) for every repository — looked similar to the earlier connectivity saga, so tested systematically again rather than assuming:
  - `ping 8.8.8.8` succeeded (raw connectivity fine)
  - `ping google.com` failed (pointed specifically at DNS resolution)
  - `cat /etc/resolv.conf` revealed the nameserver was still set to the old, incorrect subnet's router IP (`192.168.100.1`) — leftover from before the network was corrected to `192.168.88.x`
- **Root cause:** when `/etc/network/interfaces` was edited earlier to fix the IP/gateway mismatch, only the `address` and `gateway` lines were updated. DNS resolution is controlled by a separate file (`/etc/resolv.conf`), which doesn't automatically follow changes made elsewhere — it kept pointing at the old, wrong router IP the whole time. This explains why basic ping/connectivity worked in earlier testing, but DNS-dependent operations (like `apt update`, which needs to resolve package repository hostnames) kept failing silently until this point.
- **Fix:** manually corrected `/etc/resolv.conf` to point to `192.168.88.1`, confirmed with a successful `ping google.com`. Then added a `dns-nameservers 192.168.88.1` line into the `vmbr0` block of `/etc/network/interfaces` itself, so the correct DNS server gets reapplied automatically on any future `ifreload`/reboot, rather than relying on a manually-edited file that could get silently overwritten again.
- `apt update && apt full-upgrade -y` proceeded successfully once DNS was corrected.

**Key lesson learned:** network configuration on Linux can be split across multiple files with different responsibilities (interface/IP/gateway vs. DNS resolution) — fixing one doesn't automatically fix the other. When troubleshooting, it's worth explicitly testing each layer (raw IP connectivity vs. DNS resolution vs. the actual service) rather than assuming a single fix fully resolved a category of problem.

**Why this matters:** this is a very close real-world parallel to the earlier IP conflict/subnet troubleshooting — reinforces the general principle that DNS and IP connectivity are separate, independently-testable layers of the network stack (directly relevant to A+ and Network+ objectives), and that partial fixes can leave a system in an inconsistent state that only surfaces later, under different symptoms.

---

### Phase 5: Creating the Pi-hole Container

**Date:** [fill in]

**Goal:** Deploy an LXC container to host Pi-hole.

**Steps taken:**
- [x] Downloaded **Debian 13 (trixie)** CT template via local storage → CT Templates (matches Proxmox's own Debian base — well-supported, standard choice)
- [x] Created LXC container ("Create CT") with:
  - Hostname: `pihole`
  - Template: Debian 13 (trixie)
  - Disk: 4GB (local-lvm)
  - CPU: 1 core
  - Memory: 512MB (Pi-hole's recommended minimum)
  - Network: static IP `192.168.88.11/24`, gateway `192.168.88.1`, bridged to `vmbr0`
  - DNS servers: `192.168.88.1` (router) — set here specifically because the container needs working DNS to install Pi-hole's software, before Pi-hole itself exists to serve as its own resolver
  - DNS domain: left blank (inherits from host)
  - Start after creation: enabled
- [x] Logged into container console, ran `apt update && apt full-upgrade -y` and `apt install curl -y` to prep the fresh template
- [x] Ran the official Pi-hole installer: `curl -sSL https://install.pi-hole.net | bash`
- [x] Worked through installer prompts:
  - Confirmed static IP (`192.168.88.11`, already set at the container level)
  - Upstream DNS provider: **Cloudflare (1.1.1.1)**
  - Blocklist: accepted default (StevenBlack aggregated list)
  - Query logging: **enabled**
  - FTL privacy mode: **Level 0 ("Show everything")** — chosen deliberately for full visibility during initial setup/learning phase and portfolio documentation; noted as a setting to revisit/discuss with roommate before the network-wide DNS rollout in Phase 7
- [x] Installation completed successfully, admin password captured
- [x] Confirmed access to Pi-hole web dashboard at `http://192.168.88.11/admin`

**Notes / Issues encountered:**
- No major issues on this phase — went smoothly, likely because the DNS bug from Phase 4 was already resolved at the host level before this container was created, so the container inherited working DNS from the start.

**Why this step matters:**
This phase covers LXC container creation/sizing (right-sizing CPU/RAM/disk for a lightweight service — a real resource-planning skill), static IP assignment for a service container (same reasoning as the host), and running/interacting with an official install script safely. The FTL privacy mode decision is also a good real-world tie-in to the roommate/shared-network privacy conversation already documented in Phase 7's plan.

---

### Phase 6: Installing & Configuring Pi-hole

**Date:** [fill in]

**Goal:** Install Pi-hole inside the container and configure it as the network's DNS server.

**Steps taken:**
- [x] Manually pointed a single test device (desktop, Windows 11) at Pi-hole (`192.168.88.11`) via Settings → Network → Edit DNS settings, Preferred DNS only (no fallback, deliberately, to make failures obvious)
- [x] Confirmed normal websites still loaded correctly (Pi-hole successfully resolving legitimate traffic)
- [x] Confirmed query activity appeared in the Pi-hole dashboard in real time
- [x] Initial test showed queries logging correctly, but **zero queries blocked** despite browsing ad-heavy sites — investigated rather than assuming it was broken

**Notes / Issues encountered:**
- **Root cause: Chrome's built-in "Secure DNS" (DNS-over-HTTPS) setting.** Modern browsers increasingly ship with DoH enabled by default, which makes the browser perform its own DNS lookups directly (e.g., to Cloudflare/Google) rather than using the OS-level DNS setting — meaning Pi-hole never saw the browser's DNS queries at all, regardless of blocklist configuration.
- **Fix:** disabled Chrome's "Use secure DNS" setting under Settings → Privacy and Security → Security. Confirmed this setting is **per Chrome profile**, not global — needs to be toggled in each profile individually (or handled once via a Windows registry/group policy for all profiles at once — not implemented yet, left as manual per-profile for now).
- **Noted for Phase 7 rollout:** this same DoH consideration applies to any device/browser on the network, including the roommate's — Firefox has an equivalent separate DNS-over-HTTPS setting. Worth mentioning during the planned rollout conversation, since Pi-hole will have zero visibility into traffic from a browser with DoH enabled, independent of any whitelist/blocklist tuning.
- After disabling Secure DNS, retested — confirmed ad/tracker domains now appearing in Query Log as blocked, and the "Queries Blocked" counter increasing in real time.

**Why this matters:**
This is a great example of a modern networking gotcha that didn't exist a few years ago — DNS-over-HTTPS was designed for legitimate privacy/security reasons (preventing ISPs or attackers from snooping/tampering with DNS), but it has the side effect of silently bypassing local network-level DNS filtering tools like Pi-hole. Understanding *why* a fix works (not just that toggling a setting fixed it) is the difference between troubleshooting and guessing.

---

### Phase 7: Router Configuration & Network-Wide Deployment

**Date:** [fill in]

**Goal:** Point the home network's DHCP/DNS settings at the Pi-hole container so all devices use it automatically.

**Real-world consideration: shared network, roommate's work device**

Before rolling Pi-hole out network-wide, identified a legitimate risk: a roommate works from home using a personal OptiPlex to remote into their work PC (exact remote-access software TBD — need to confirm: corporate VPN client like AnyConnect/GlobalProtect, RDP, Citrix, AnyDesk, etc.). Since a network-wide DNS change affects every device on the network, this needed a deliberate plan rather than just flipping the router's DNS setting.

**Decision update:** After weighing the options, decided the simplest and lowest-risk approach is to **exclude the roommate's work device entirely** rather than attempt to identify and whitelist every domain their VPN/remote-access software depends on. Since the device is used solely for work (not general browsing), there's no meaningful ad-blocking benefit to including it, and permanent exclusion removes the risk category entirely rather than managing it.

**Mitigation / rollout plan (revised):**
1. **Communicate first** — brief, low-effort conversation with roommate: explain a network-wide ad blocker is being set up, but their work device will be excluded entirely, so no action or compatibility concerns needed on their end. Still worth doing as basic courtesy for a shared network change, even without a technical dependency.
2. **Exclude the device from Pi-hole at the DNS level** — once the router's DNS is pointed at Pi-hole network-wide, manually override DNS on the roommate's OptiPlex specifically (Network adapter settings → set DNS to router IP or a public resolver like `1.1.1.1`/`8.8.8.8` instead of inheriting from DHCP). This overrides the network-wide setting for just that one device, regardless of router feature support.
   - Alternative considered: DHCP reservation with a per-device DNS override, if the Netgear/router setup supports it — not pursued as the primary method since manual override is simpler and more universally reliable.
3. **Fast rollback plan** — document the exact steps to revert DNS back to normal at the router level, so it can be undone in under a minute if something breaks while the roommate is mid-work (applies to the rest of the network, though roommate's device is unaffected either way given the exclusion).

**Steps taken:**
- [x] Gave roommate a quick heads-up before making the change (courtesy notice, low-effort)
- [x] **Reassessed the exclusion plan before executing:** confirmed via earlier `ipconfig /all` inspection that the roommate's device already receives three VPN-pushed DNS servers (two `10.x` internal company servers, one `192.168.104.x`) whenever connected — meaning their DNS resolution is already independent of the router/DHCP, VPN-connected or not (for the vast majority of their session time). **Decision: skipped the manual DNS override on their device**, since it would have been redundant insurance rather than a necessary safeguard, and verified live instead with the roommate on standby to test immediately after the change.
- [x] Logged into router, changed **Primary DNS** in DHCP Server settings from ISP default (`192.152.0.1`) to Pi-hole (`192.168.88.11`)
- [x] **Left Secondary DNS blank deliberately** — reasoning: filling it with a backup (e.g., the old ISP DNS) risks devices silently falling back to unfiltered DNS if Pi-hole ever has a hiccup, with no visibility into it happening. A single DNS entry means a Pi-hole outage causes an obvious, diagnosable total DNS failure rather than a silent, invisible bypass of the whole filtering setup.
- [x] Forced a fresh DHCP lease on desktop via `ipconfig /release` / `ipconfig /renew`, confirmed new DNS server (`192.168.88.11`) via `ipconfig /all`
- [x] Confirmed via Pi-hole Query Log that desktop traffic was flowing through and being filtered correctly
- [x] Roommate connected to work VPN and logged in normally — **no impact confirmed, live**

**Notes / Issues encountered:**
- No issues. The earlier decision to investigate the roommate's actual DNS configuration (rather than assume a whitelist or exclusion was strictly necessary) paid off directly here — real diagnostic data from a prior session (`ipconfig /all` output) informed a simpler, lower-effort rollout without sacrificing safety.

**Why this matters:**
This is a good example of **not over-engineering a solution** — the original plan (whitelist domains, or manually exclude the device) was reasonable given the information available at the time, but revisiting the plan once better data existed (the VPN's own DNS servers) led to a simpler, equally safe outcome. Also reinforces a real operational security/reliability principle: a single point of DNS failure that fails *loudly* (total resolution failure) is preferable to a redundant fallback that fails *silently* (quietly bypassing your filtering with no alert).

---

### Phase 8: Testing & Verification

**Date:** [fill in]

**Goal:** Confirm ad-blocking is functioning network-wide and Pi-hole's dashboard shows query logs.

**Steps taken:**
- [x] Confirmed desktop is resolving DNS through Pi-hole (`192.168.88.11`) after DHCP renewal
- [x] Confirmed Query Log shows live traffic and actual blocked entries when browsing ad-heavy sites
- [x] Confirmed roommate's work VPN connection and login worked normally post-rollout, with no manual intervention needed on their device
- [x] Project considered **live and stable** as of rollout night

**Notes / Issues encountered:**
- None — rollout was uneventful specifically because of the groundwork laid earlier (DNS troubleshooting experience from Phase 4, the DoH discovery from Phase 6, and the informed decision-making around the roommate's VPN in Phase 7).

---

## Lessons Learned

- **Networking fundamentals compound.** The DNS bug in Phase 4 (stale `/etc/resolv.conf`), the IP conflict in the earlier troubleshooting saga, and the DoH discovery in Phase 6 all turned out to be different flavors of the same underlying skill: isolating *which layer* of the network stack is actually failing (physical link → IP/ARP → DNS → application-level) rather than guessing at the whole system. By Phase 7, that same isolation logic made assessing the roommate's VPN setup fast and confident instead of another multi-hour troubleshooting session.
- **Not every warning is a problem.** Between the Proxmox "no-subscription repository" warning and various browser self-signed cert warnings, a recurring theme was learning to distinguish an informational notice from an actual misconfiguration — a skill that matters as much as fixing real issues.
- **Documentation habits paid off in real time, not just in hindsight.** Having the Phase 7 plan already written out (with the reasoning behind the original whitelist/exclusion approach) made it easy to revisit and simplify the plan later, once better information (the roommate's actual VPN DNS behavior) became available — the plan evolved deliberately rather than being reactively patched at the last minute.
- **A safe default beats a clever one.** Leaving Secondary DNS blank (rather than filling it with a fallback) was a small decision with a real principle behind it: prefer a failure mode that's loud and obvious over one that's silent and invisible, especially for something like DNS filtering where a silent bypass defeats the entire point of the project.
- **Real hardware/network problems are messier — and more valuable — than tutorials suggest.** Nearly every phase of this project hit at least one unplanned issue (BIOS/boot mode mismatch, Rufus DD mode quirk, wrong subnet during install, DNS resolv.conf bug, Chrome DoH bypass, IP conflict). None of these were failures — each one is a genuine, resume-worthy troubleshooting story that a clean, no-issues build wouldn't have produced.

## Resources / References

> Future project roadmap has moved to its own file: `homelab-roadmap.md`

- Proxmox VE official documentation: https://pve.proxmox.com/wiki/
- Pi-hole official documentation: https://docs.pi-hole.net/
