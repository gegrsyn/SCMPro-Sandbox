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

## 7. Attack the shop, then find it in the logs

1. Browse to **https://shop.ngfwsandbox.com** (OWASP Juice Shop, deliberately vulnerable).
2. Try a classic: in the login form, use `' OR 1=1--` as the email and anything as the password (SQL injection).
3. In SCM's log viewer, find it twice: **Internet to DMZ Web** is the encrypted visit to the reverse proxy, and
   **DMZ Web to App Server** is the clear-text hop from the proxy to the app, where threat prevention can see the
   payload. Look for threat entries on that second hop.
4. Discuss: why inspect the internal hop instead of decrypting inbound at the edge? What would change with SSL
   Inbound Inspection?

## 8. See what decryption adds

1. Outbound user traffic is decrypted (SSL Forward Proxy). Compare the application names in the traffic logs with
   a no-decrypt category (banking, health, government), which stays as plain `ssl`.
2. Find the decryption log entries and check the TLS version and the certificate details.
3. On the OT side (Panorama), the OT DMZ patch server's traffic is decrypted too: same feature, other management
   plane.

## 9. Why didn't the website get a certificate? (a true story)

Publishing the shop failed four times, each time because a firewall feature was doing its job. Use the logs to
find all four:

1. **Outbound App-ID after decryption.** The web server's call to Let's Encrypt was decrypted and identified as
   `acme-protocol`, not `ssl`, so the outbound rule didn't match. Package updates turned into `apt-get` the same way.
   *Lesson: turning on decryption makes App-ID more specific, and your allow-lists have to follow.*
2. **Inbound App-ID.** Let's Encrypt's validation request coming *in* was also `acme-protocol`, and the inbound rule
   only allowed `web-browsing`/`ssl`.
3. **URL filtering on an inbound rule.** The brand-new site was categorized `insufficient-content` and blocked
   by the URL profile in the best-practice group. URL filtering protects *users browsing out*; a published server
   gets a profile group without it (`sbx-inbound-server`).
4. **URL filtering on the internal hop.** Once HTTPS worked, visitors got a *Web Page Blocked* page. The clear-text
   request from the reverse proxy to the app still carried `Host: shop.ngfwsandbox.com`, and the app-server rule's
   URL profile blocked it. That page travelled back out through the proxy.

Find each one: the traffic log's session end reason (`policy-deny`, `threat`), the rule that matched, and the URL log.

## 10. Clean up with Policy Optimizer and Config Cleanup

The rule bases are built to give these tools something to say. Most rules carry real traffic; some never will.

1. In SCM, open **Policy Optimizer** for the HQ and Branch firewalls. Find:
   - **Rules with no hits:** the guest kiosk, legacy FTP, partner extranet and branch printer rules (their hosts are
     gone), plus the P2P and remote-support blocks (nobody tries).
   - **Rules with `any` application:** *HQ Users to Internet* and *Branch Users to Internet* are catch-alls under
     the focused App-ID rules (GenAI, collaboration, SaaS, storage, social, streaming, dev tools, web). See which
     apps they still match, and convert them.
2. Open **Config Cleanup** and find the unused objects: decommissioned servers, an old printer and a vendor range,
   unused services, the `decom-servers` address group and the `sbx-legacy-file-transfer` application group.
3. Do the same for the OT pair (Panorama, visible in SCM): a disabled rule, a shadowed duplicate, a rule pointing
   at a host that no longer exists, and broad "temporary" rules.
4. Apply one recommendation, push, and confirm the hit counts afterwards. `Reset` puts everything back.

## Reset

Everything here is reset to the known-good state periodically. If you broke something and want it back
sooner, ask the person who gave you access.
