# SCM Pro Sandbox

A hands-on lab for **Palo Alto Networks Strata Cloud Manager (SCM) Pro**: real VM-Series firewalls on the
internet, a high-availability pair, a branch, an OT segment managed by Panorama, and generated traffic so
the logs and dashboards have something in them.

If you've been given a login, this repo explains what's in the lab and what's worth trying.

[![Overview](drawings/overview.svg)](https://cdn.jsdelivr.net/gh/gegrsyn/SCMPro-Sandbox@main/drawings/overview.svg)

*Click any drawing to open it full size (zoomable in your browser).* Per-site detail:
[HQ](#hq) · [Branch](#branch) · [OT](#ot)

## What's in it

| Site | Firewalls | Managed by | Notes |
|---|---|---|---|
| **HQ** | SBX-HQ-FW-A / SBX-HQ-FW-B, active/passive HA | SCM Pro | users, servers, DMZ with a published web app (OWASP Juice Shop) |
| **Branch** | SBX-BR-FW | SCM Pro | a newer PAN-OS release than HQ, on purpose; IPsec to HQ |
| **OT** | SBX-OT-FW-A / SBX-OT-FW-B, HA | Panorama (connected to SCM) | Purdue-style segmentation behind HQ; locally managed "for compliance" |

All firewalls are VM-Series. PAN-OS versions are deliberately **not** the latest, so you can schedule and run
upgrades yourself.

## Site detail

### HQ
[![HQ](drawings/hq.svg)](https://cdn.jsdelivr.net/gh/gegrsyn/SCMPro-Sandbox@main/drawings/hq.svg)

### Branch
[![Branch](drawings/branch.svg)](https://cdn.jsdelivr.net/gh/gegrsyn/SCMPro-Sandbox@main/drawings/branch.svg)

### OT
[![OT segment](drawings/ot.svg)](https://cdn.jsdelivr.net/gh/gegrsyn/SCMPro-Sandbox@main/drawings/ot.svg)

Drawings are generated from the sandbox's configuration-as-code with [D2](https://d2lang.com); the `.d2`
sources sit next to each SVG.

## Things to try

See [GUIDE.md](GUIDE.md) for step-by-step exercises:

- explore the folder hierarchy and how config is inherited (`SCM-Sandbox` › `SBX-HQ` / `SBX-Branch`);
- read traffic, threat and URL logs from generated user, server and OT traffic;
- schedule a **PAN-OS upgrade** on the HA pair and watch SCM handle the peers;
- fail over the HA pair;
- change a security rule, push it, and see who changed what in the config history;
- compare **Zero Trust / best-practice posture** in SCM and in Panorama on the same kind of rule base.

## Runbooks

- [Connect Panorama to Strata Cloud Manager and Strata Logging Service](PANORAMA-TO-SCM.md): the two plugins, the steps, the gotchas the docs leave out, and how to verify each stage.

## Known limitations

- **ngfwsandbox.com is a newly registered domain (October 2026).** For roughly its first 30 days, Palo Alto Networks
  URL filtering (and most other vendors') classifies it as *newly-registered-domain*, which best-practice profiles
  block. If you're behind a corporate firewall and `shop.ngfwsandbox.com` won't load, that's the likely reason. Try
  from another network, or check the category with [Test A Site](https://urlfiltering.paloaltonetworks.com/).
- **Inbound HTTPS to the shop is not decrypted** at the firewall. TLS terminates on the DMZ reverse proxy, and the
  firewall inspects the clear-text hop from the proxy to the app server instead (rule *DMZ Web to App Server*).
- **The shop is intentionally vulnerable** (OWASP Juice Shop). Attack it, not anything else; it's reset regularly.

## Ground rules

- **Break things freely.** The sandbox is reset to a known-good state regularly. Anything you add or change in
  the sandbox folders can disappear at the next reset.
- **Don't use it for anything real.** No production data, credentials or customer configuration.
- **Shared space.** Other people may be in the tenant at the same time; scheduled upgrades and pushes affect
  everyone.

## Status

| Part | State |
|---|---|
| HQ HA pair and Branch (SCM) | live on the internet, policy pushed |
| OT HA pair and Panorama | built; 35-rule Purdue rule base pushed from Panorama |
| Panorama connected to SCM and Strata Logging Service | done (see the runbook) |
| Traffic generators | running in every zone (users, servers, OT Modbus/SCADA) |
| HQ to Branch IPsec | up, carrying branch-to-HQ file and web traffic |
| SSL decryption | outbound (HQ, Branch, OT DMZ); inbound inspected on the proxy-to-app hop |
| Public web app | [shop.ngfwsandbox.com](https://shop.ngfwsandbox.com) (OWASP Juice Shop) |

## License

Drawings and documentation are licensed under [CC BY 4.0](LICENSE). Reuse them, with credit.

Palo Alto Networks, Strata Cloud Manager, Panorama, PAN-OS and VM-Series are trademarks of Palo Alto Networks,
Inc. This is a personal lab, not an official Palo Alto Networks resource.
