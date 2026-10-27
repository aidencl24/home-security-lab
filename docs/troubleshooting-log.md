# Troubleshooting log

Every problem hit while building the lab, how it was diagnosed, and what fixed it. Newest first. Dates are 2026.

**Status:** ✅ resolved and verified · 🟡 open

---

## Summary

| Date | Problem | Root cause | Status |
|---|---|---|---|
| Oct 26 | web01 could not ping the internet; time sync blocked | SERVERS "Updates only" rule was TCP-only | ✅ |
| Oct 26 | `git clone` on web01: "Could not resolve host" | DNS lookups through pfSense intermittently slow or timing out | 🟡 Open |
| Oct 26 | Suricata warned that hardware offloading was enabled | pfSense default NIC offload settings | ✅ |
| Oct 26 | Some Suricata events arrived in Wazuh cut off | BSD syslog message size limit (~1 KB) | ✅ |
| Oct 26 | Wazuh timestamps were in UTC, pfSense in local time | Ubuntu's default time zone | ✅ |
| Oct 25 | Suricata alerts reached Wazuh but raised no Suricata alert | EVE JSON arrived wrapped in a syslog header | ✅ |
| Oct 25 | Suricata installed but the Alerts tab was empty | Test traffic never crossed the monitored interface | ✅ |
| Oct 23 | pfSense logs reached Wazuh but raised no alerts | BSD syslog from pfSense omits the hostname | ✅ |
| Oct 23 | Wazuh stopped accepting agents on TCP 1514 | Agent `<remote>` block overwritten by a syslog block | ✅ |
| Oct 23 | Wazuh VM would not power on: disk locked | Stale VMware process still held the virtual disk | ✅ |
| Oct 21 | web01 agent stuck at "Unknown" | Wazuh VM disk 100% full | ✅ |
| Oct 21 | Wazuh VM could not resolve names during install | No DNS rule on the SECURITY zone | ✅ |
| Oct 20 | web01 used only 10 of 20 GB | Ubuntu installer allocates half the LVM volume | ✅ |
| Oct 20 | web01 could not resolve names during install | No DNS rule on the SERVERS zone | ✅ |
| Oct 20 | Kali could not ping the Servers zone | New pfSense rules default to TCP; ping is ICMP | ✅ |
| Oct 20 | Kali apt downloads failing | Default mirror refusing connections; no IPv6 in the lab | ✅ |
| Oct 20 | Kali mouse cursor invisible | VMware and Kali graphics conflict | ✅ Worked around |
| Oct 12 | Users zone had no gateway address | LAN interface set as a DHCP client | ✅ |
| Oct 12 | VirtualBox VMs unstable | Likely conflict with Windows 11 Hyper-V features | ✅ Moved to VMware |

---

## Detailed entries

### Oct 26 · web01 could not ping out; time sync blocked ✅

**Symptom:** `ping 8.8.8.8` from web01 lost 100% of packets, while `apt` and HTTPS worked. web01's clock reported "System clock synchronized: no".

**Diagnosis:**
1. pfSense **Diagnostics → Ping** to 8.8.8.8 from WAN succeeded, so pfSense's own internet path was fine.
2. The firewall log filtered to source 10.10.20.10 showed blocks by **"Default deny rule IPv4"** on SERVERS: ICMP to 8.8.8.8 and **UDP 123 (NTP)** to Ubuntu's time servers, repeating every few minutes.
3. The SERVERS rules tab showed the "Updates only" rule as **IPv4 TCP**, although it was documented as Any.

**Root cause:** pfSense creates new rules with protocol TCP by default. The rule matched HTTP/HTTPS but not ICMP or UDP, so ping and NTP fell through to default deny.

**Fix:** Set the rule's protocol to **Any**. Verified with 3/3 ping replies and chrony reporting `Leap status: Normal` and "System clock synchronized: yes".

**Lesson:** Test egress per protocol, not with one tool. The same TCP default caused the Oct 20 Kali ping problem.

### Oct 26 · Intermittent DNS failures on web01 🟡

**Symptom:** `git clone https://github.com/...` failed with "Could not resolve host". Moments later, `resolvectl query github.com` succeeded but took 1.9 seconds, and a `curl` a minute after that failed to resolve after 4 seconds.

**What is known:** web01 uses pfSense (10.10.20.1) as its resolver, and the SERVERS DNS rule passes. HTTPS to GitHub works once a name resolves. A retry of the clone succeeded.

**Next step if it recurs:** time lookups from pfSense itself (**Diagnostics → DNS Lookup**) to see whether the delay is in pfSense's DNS Resolver or between web01 and pfSense.

### Oct 26 · Suricata events truncated in transit ✅

**Symptom:** some lines in `/var/log/suricata-eve.json` ended mid-value with no closing `}`.

**Diagnosis:**
```bash
sudo grep -vc '}$' /var/log/suricata-eve.json          # 2 incomplete lines
sudo awk '{print length($0)}' /var/log/suricata-eve.json | sort -n | tail -n 5
```
The longest lines stopped at exactly **986 characters**, three times: a hard cap.

**Root cause:** pfSense's BSD-format syslog limits each message to about 1 KB. Alerts with payload logged twice (base64 and printable), plus routine traffic events for 12 protocols, could exceed it. Wazuh can't decode incomplete JSON, so those events were silently lost.

**Fix:** in the SERVERS Suricata settings, EVE payload set to **NO**, all EVE traffic types unchecked, and "Send Alerts to System Log" unchecked (it duplicated every alert). Alerts now arrive at roughly 450 characters. Full payloads remain in pfSense's own Suricata Alerts tab.

**Lesson:** a transport can silently change the data it carries. Check that what arrives is complete, not only that something arrived.

### Oct 26 · Hardware offloading warning ✅

**Symptom:** a banner on the Suricata interface page: Suricata requires hardware checksum, TCP segmentation and large receive offloading to be disabled.

**Fix:** **System → Advanced → Networking**: checked all three "Disable hardware … offload" boxes, saved, rebooted pfSense. The end-to-end Suricata test still passed afterward.

**Why it matters:** with offloading on, the virtual NIC can hand Suricata packets with incorrect checksums or merged segments, which can cause missed detections.

### Oct 25 · Suricata alerts reached Wazuh but raised no Suricata alert ✅

**Symptom:** the Suricata alert was present in the relay's `/var/log/pfsense.log`, but no matching alert appeared in Wazuh's `alerts.log`.

**Diagnosis:** replaying the line through `wazuh-logtest`:
- Phase 1 (pre-decoding) worked: hostname `pfSense`, program `suricata`.
- Phase 2: **"No decoder matched."**
- Phase 3: generic rule **1002** "Unknown problem somewhere in the system" (level 2), below the level written to `alerts.log`.

Stripping the syslog header and replaying only the JSON produced the `json` decoder and rule **86601** "Suricata: Alert - GPL ATTACK_RESPONSE id check returned root". That confirmed the cause before anything was changed.

**Root cause:** Wazuh's Suricata rules expect pure JSON. Suricata's EVE output was sent as syslog, so each JSON object arrived behind a syslog header.

**Fix:** the rsyslog relay now writes Suricata JSON bodies to their own file, `/var/log/suricata-eve.json`, which Wazuh reads with `log_format json` ([config](../configs/wazuh/)). Verified end to end: Kali → web01 trigger, Suricata SID 2100498, Wazuh rule 86601.

**Lesson:** "the log arrived" and "the SIEM understood it" are different stages. `wazuh-logtest` shows exactly which stage fails.

### Oct 25 · Suricata installed but no alerts ✅

**Symptom:** Suricata running (green ✓), ET Open rules downloaded, empty Alerts tab.

**Diagnosis:**
- Suricata monitored only **SERVERS (em2)**.
- The test pings from Kali targeted pfSense's own interface addresses (10.10.x.1). Those packets enter on em4 and are answered by pfSense, so they never cross em2. The reply's `ttl=64` (no routing hop) confirmed it.
- Plain ICMP doesn't match any enabled rule category anyway.
- A public test page (testmynids.org) no longer returns its test content: the domain is parked.

**Fix (no configuration change needed):** a known-good test inside the lab. Kali sent the text `uid=0(root) gid=0(root) groups=0(root)` to a netcat listener on web01, across em2. Suricata raised SID 2100498 "GPL ATTACK_RESPONSE id check returned root".

**Lesson:** a sensor only sees traffic on its interface, and a test is only valid if it contains what the rule looks for.

### Oct 23 · pfSense logs reached Wazuh but raised no alerts ✅

**Diagnosis:** `tcpdump` on the Wazuh VM showed the syslog arriving; `wazuh-logtest` showed Wazuh reading `filterlog` as the hostname, so no pfSense decoder matched.

**Root cause:** pfSense's BSD (RFC 3164) syslog omits the hostname field.

**Fix:** an rsyslog relay on the Wazuh VM accepts UDP 514 only from 10.10.30.1, inserts the hostname `pfSense`, and writes `/var/log/pfsense.log`, which Wazuh reads as syslog. Blocked traffic now raises rules 87701 and 87702.

### Oct 23 · Wazuh stopped accepting agents ✅

**Root cause:** while adding syslog input, the default secure `<remote>` block (TCP 1514) in `ossec.conf` was overwritten.

**Fix:** restored the secure / 1514 / tcp block; web01 returned to Active. Syslog now arrives through rsyslog instead, so the agent block is never touched.

### Oct 23 · Wazuh VM disk locked ✅

**Root cause:** a leftover VMware process still held the `.vmdk` open after VMware was closed with VMs running.

**Fix:** cleared the stale lock. Rule since: suspend or shut down every VM before closing VMware or sleeping the laptop.

### Oct 21 · web01 agent stuck at "Unknown" ✅

**Symptom:** the agent registered, but the manager logged dbsync "Cannot save" errors.

**Root cause:** the Wazuh VM's disk was 100% full. The Ubuntu installer had used only half of the 30 GB LVM volume.

**Fix:** `lvextend -r -l +100%FREE /dev/ubuntu-vg/ubuntu-lv`, then restarted wazuh-manager; the agent went Active.

**Lesson:** expand LVM after every Ubuntu Server install (the same issue appeared on web01 on Oct 20).

### Oct 20–21 · New zones could not resolve names ✅

**Root cause:** default deny blocks DNS. The SERVERS and SECURITY zones had no rule allowing TCP/UDP 53 to pfSense.

**Fix:** a DNS pass rule on each zone. **Lesson:** every new zone needs its own DNS rule.

### Oct 20 · Kali could not ping the Servers zone ✅

**Root cause:** the ATTACKER pass rules were created with pfSense's default protocol, TCP.

**Fix:** set the ATTACKER rules to protocol Any.

### Oct 20 · Kali apt failures and display problems ✅

- **apt failures:** the default mirror refused connections and the lab has no IPv6. Fixed by switching to the `kali.download` mirror and forcing IPv4.
- **Invisible mouse cursor:** a VMware graphics conflict. Kali now runs command-line only, and pfSense is managed from the host's browser.

### Oct 12 · Early setup ✅

- **Users zone had no gateway:** the LAN interface was set as a DHCP client. Re-ran pfSense console option 2 with a static 10.10.10.1/24.
- **Unstable VMs:** VirtualBox ran poorly alongside Windows 11's Hyper-V features. The lab was rebuilt in VMware Workstation Pro.

---

## Open items

- 🟡 Intermittent DNS slowness on web01 (above).
- 🟡 Some Kali → web01 connections did not appear in the pfSense firewall log. Likely local log rotation (Kali's blocked DNS queries generate steady log volume); to recheck during the first detection exercise.
