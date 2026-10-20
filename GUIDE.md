# Guided exercises

Each exercise takes 5 to 20 minutes. Do them in any order. Menu paths can shift between SCM releases, so if
a path has moved, search for the feature name.

## 1. Find your way around

1. Open **Configuration › NGFW and Prisma Access** and pick the `SCM-Sandbox` folder.
2. Walk the hierarchy: `SCM-Sandbox` holds shared objects (addresses, interface profile); `SBX-HQ` and
   `SBX-Branch` hold each site's interfaces, zones, routing, policy and NAT.
3. Notice that folder interfaces are **variables** (`$untrust`, `$hq-users` …) mapped to a physical port per
   firewall. That's how one folder definition serves both members of the HA pair.

## 2. Read the logs

1. Open the log viewer and filter to the last hour.
2. Find web browsing from `hq-users`, AD and file traffic to `hq-servers`, and inbound hits on the published
   web app in `hq-dmz`.
3. Find the explicit deny rules (`HQ Deny All`, `Branch Deny All`) and see what they block.

## 3. Schedule a PAN-OS upgrade

1. Go to the software upgrade workflow for NGFWs.
2. Pick the HQ HA pair and a newer release; schedule it.
3. Watch how SCM sequences the peers (passive first, then fail over) and what the devices report during it.

> Upgrades take the pair through reboots. Coordinate if others are using the lab.

## 4. Fail over HA

1. Check which HQ firewall is active.
2. Suspend the active peer from the device management actions, and watch the passive one take over.
3. Make it functional again and observe preemption.

## 5. Change policy and push

1. Add a rule in `SBX-HQ` (for example, block a URL category for `hq-users`).
2. Push, then check the config version history: who changed what, and the diff.
3. Generate a hit and find it in the logs.

## 6. Compare posture: SCM and Panorama

1. In SCM, open the best-practice / Zero Trust posture views for the HQ and Branch firewalls.
2. In Panorama, review the OT pair's rule base and its posture checks.
3. Compare the findings: overly broad rules, missing profiles, unused rules, logging gaps.

## Reset

Everything here is reset to the known-good state periodically. If you broke something and want it back
sooner, ask the person who gave you access.
