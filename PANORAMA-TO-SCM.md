# Runbook: connect Panorama to Strata Cloud Manager and Strata Logging Service

This runbook connects an existing Panorama and its managed firewalls to:

1. **Strata Logging Service (SLS).** Panorama-managed firewalls send their logs to the cloud, and Panorama can query them there.
2. **Strata Cloud Manager (SCM).** SCM gets visibility of Panorama's configuration and managed firewalls, for posture, best-practice and AIOps views.

It was written while doing this for real in the SCM Pro sandbox, so it includes the parts the official docs leave out. Those are called out as **Gotcha** throughout.

**Tested with:**
- Panorama 11.1.16-h3 and managed VM-Series on 11.1.13-h15
- Cloud Services plugin 6.2.0-h4
- CloudConnector plugin 3.0.3

## The one thing to know first

**These are two separate plugins doing two separate jobs.**

| Goal | Plugin | Notes |
|---|---|---|
| Logs to SLS for Panorama-managed firewalls | **Cloud Services** (`cloud_services`) | Required, even though most of its GUI is Prisma Access |
| Panorama visible in SCM | **CloudConnector** (`cloudconnector`) | Preinstalled on Panorama 11.0.1+ |

> **Gotcha:** the Panorama-to-SCM documentation only mentions CloudConnector. You find out that SLS needs Cloud Services when SLS shows an error saying the plugin is missing or too old.

## Prerequisites

- [ ] **Tenant association.** In the hub or support portal, associate **Panorama and every managed firewall** with the tenant. The SLS licence must be on the same support account.
- [ ] **Device certificates** on Panorama and on every firewall. On each device: Device → Setup → Management → Device Certificate → Get certificate, using an OTP from the support portal.
- [ ] **Restart the management server after installing any certificate.** Do this on each device: `debug software restart process management-server`. See the gotcha below; this one matters.
- [ ] **DNS and NTP** set on Panorama and on the firewalls.
- [ ] **Outbound access, with no SSL decryption** of this traffic:
  - TCP 443 and 444 to the SLS and cloud management endpoints;
  - TCP **3978** to the SLS receptor, which carries the log channel;
  - OCSP on port 80.

  If a firewall sits in the path, App-IDs such as `paloalto-logging-service`, `paloalto-device-telemetry`, `paloalto-shared-services` and `paloalto-updates` cover most of it.
- [ ] **Telemetry enabled.** Panorama → Setup → Telemetry, and in the template for the firewalls (Device → Setup → Telemetry). Pick the same region as your tenant.

> **Gotcha: restart after certificates.** After a device certificate or SLS certificate is installed, PAN-OS keeps using its old state until the management server restarts. Until then you get misleading errors:
> - "Failed to validate server certificate for endpoint api.paloaltonetworks.com"
> - "Failed to fetch ingest/query FQDN for customer (curl failed)"
> - SLS reporting that the plugin is too old
>
> Restarting `reportd` alone is not enough. Restarting the management server is non-disruptive: traffic keeps flowing and the GUI and API drop for about two minutes.

## Part A: Strata Logging Service

1. **Install Cloud Services.** Panorama → Plugins → Check Now. Download and install the newest version Panorama lists.
   > **Gotcha:** Panorama only lists versions compatible with its PAN-OS release. Don't second-guess a listed version from the compatibility matrix. In testing, 6.2.0-h4 installed and ran fine on 11.1.
2. **Generate an OTP in the SLS app itself:** Inventory → Panorama Appliances → Generate OTP. It's valid for 10 minutes.
   > **Gotcha:** this OTP comes from **the SLS app**, not the support portal's Device Certificates page.
3. **Verify the account.** Panorama → Cloud Services → Status → **Verify Account**, and paste the OTP.
   > **Gotcha:** everything else under Cloud Services is Prisma Access configuration. Ignore it.
   >
   > CLI equivalent: `request plugins cloud_services panorama-certificate fetch otp <OTP>`
4. **Configure the firewalls through a template:** Device → Setup → Management → Strata Logging Service.
   - Choose **Enable Duplicate Logging (Cloud and On-Premise)** to send logs to both SLS and Panorama's collectors, or plain **Enable Strata Logging Service** for cloud only.
     > **Gotcha:** these two are mutually exclusive. Enabling both makes the commit fail with "Duplicate logging and Logging service cannot be enabled at the same time".
   - **Region:** your tenant's region, for example `americas`.
   - Optionally, **Enhanced Application Logging**.
   - Duplicate logging requires the firewalls to be in a **Collector Group** on Panorama.
5. **Refresh licences:** Panorama → Device Deployment → Licenses → Refresh, then select the firewalls.
6. **Commit to Panorama and push** the template stack.
7. Restart the management server on each firewall if they were certificate-fresh. See the prerequisites gotcha.

## Part B: Strata Cloud Manager (CloudConnector)

1. **Plugin.** CloudConnector is built into Panorama 11.0.1 and later. Upgrade it from Panorama → Plugins if a newer version is listed.
2. **Turn the plugin on. This is the master switch.** From the **CLI**:

   ```
   request plugins cloudconnector enable basic
   ```

   Expected output: `CloudConnector plugin has been enabled.` It also provisions a service admin
   (`__cloudconnector-schedule`) and queues a Panorama commit by itself. Check it finishes: `show jobs id <n>`.

   > **Gotcha:** this command only works in the CLI. Sent through the XML API, it fails with an empty error.

3. **Turn on config sync**, which is a separate feature inside the plugin:

   ```
   request plugins cloudconnector scm-panorama-config-sync enable
   ```

   > **Gotcha: you need both commands.** The Panorama/SCM docs mention only `enable basic`. Other guidance often
   > gives only `scm-panorama-config-sync enable`. The config-sync command reports success even while the plugin
   > itself is off, and then nothing syncs. Run `enable basic` first, then the config-sync command.

   > **Gotcha:** the config-sync command is a `request` command and is documented as needing no commit. In testing
   > it still left a candidate change, a new password hash for the plugin's built-in `__cloudconnector` account.
   > **Check for pending changes and commit them** (Commit → Commit to Panorama).

4. **Check the switches:**

   ```
   show plugins cloudconnector status
   ```

   Healthy: `CloudConnector plugin is enabled` and `Connectivity to region … and license check is a success`.
   If it says `CloudConnector plugin is disabled`, step 2 hasn't been done, even when connectivity passes.

5. **Other switches you'll see:**
   - `request plugins cloudconnector sync enable|disable` is "Cloud Sync". It's listed separately in the status line and isn't covered by the SCM docs.
   - `enable cloudngfw`, `show plugins cloudconnector regions|tenants|resources|tenant-dg-mapping` are for Cloud NGFW (AWS/Azure) integration, so not needed here.
   - `request plugins cloudconnector pycurl-verbosity` turns on debug logging of the plugin's HTTP calls.

6. Telemetry and the device certificate from the prerequisites are what SCM uses. Nothing else needs configuring on Panorama.

## Verification

| What | Where | Healthy looks like |
|---|---|---|
| Panorama ↔ SLS | CLI: `request plugins cloud_services logging-service status` | overall `pass`, "Successfully fetched Logging Service customer info", region status `OK` |
| Panorama ↔ SLS | SLS app → Inventory → Panorama Appliances | Panorama shown as connected |
| Firewall → SLS | firewall CLI: `request logging-service-forwarding status` | your Customer ID, the region, "Successfully fetched Logging Service customer info" |
| Firewall → SLS log channel | network: firewall management IP → SLS receptor on TCP 3978 | several long-lived sessions (default connection count 5) |
| Firewall → Panorama | firewall CLI: `show logging-status` | Panorama collector `Active`, sequence numbers advancing |
| Panorama ↔ SCM | Panorama CLI: `show plugins cloudconnector status` | "CloudConnector plugin is **enabled**" **and** "Connectivity to region … and license check is a success" |
| Telemetry | SCM → Insights → NGFW → Status and Monitoring | the firewalls counted under *Sending Telemetry* |
| Logs | SCM → Log Viewer, filtered to a Panorama-managed firewall | recent traffic and threat entries |

> **Gotcha: misleading log status.** On Panorama-managed firewalls, `show logging-status` lists extra receivers named `lr-cms0` / `lr-cms1` as **Inactive** even when SLS forwarding works. They are not the SLS channel. Check `request logging-service-forwarding status` and the TCP 3978 sessions instead.

> **Gotcha: telemetry count.** SCM's *Sending Telemetry* tile counts firewalls only. Panorama sends telemetry too, every 5 minutes, but isn't included in that number.

> **Gotcha: stale SLS warning.** After the fixes above, SLS may still say "Cloud Services plugin must be 2.2 Innovation or above" for about 30 minutes. That's cached data, using the plugin's old version scheme, and it clears on its own.

## Troubleshooting quick reference

| Symptom | Likely cause | Fix |
|---|---|---|
| "Failed to validate server certificate for endpoint api.paloaltonetworks.com" | certificate installed after the management server started | `debug software restart process management-server` |
| "Customer is not provisioned in CSP" | device not associated with the tenant or SLS instance | associate the serial in the hub or support portal, then `request logging-service-forwarding customerinfo fetch` |
| Cloud Services status `fail: Exception 'not Before'` | Panorama account not verified yet | Part A, steps 2–3 (OTP from the SLS app) |
| "Failed to fetch ingest/query FQDN for customer (curl failed)" with no network denies | stale management-server state after certificates | restart the management server |
| SLS says the plugin is too old despite a new version | cached data | wait about 30 minutes |
| SCM doesn't show Panorama's config; status says "CloudConnector plugin is disabled" | only `scm-panorama-config-sync enable` was run | `request plugins cloudconnector enable basic` (CLI), let its commit finish, then re-run the config-sync enable |
| `request logging-service-forwarding certificate fetch` → "Thermite certificate is used for CDL communication" | not an error | the device certificate is used; no separate SLS certificate is needed |

## After an upgrade: the API key certificate

On 11.x, administrators may see a warning that API keys were generated with a deprecated algorithm. To fix it, set an **API key certificate**: Device (or Panorama) → Setup → Management → Authentication Settings → API Key Certificate.

- It must be a **self-signed RSA CA of at least 3072 bits**. Smaller keys, or certificates that aren't a CA, are rejected.
- Committing it **invalidates every existing API key**. Scripts and integrations need new keys afterwards.
- It did not affect the Cloud Services or CloudConnector plugins.
