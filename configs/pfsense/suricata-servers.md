# Suricata on pfSense: SERVERS interface

pfSense keeps these settings in its web GUI, so they are recorded here as tables. Verified Oct 26, 2026.

## Versions and rule source

| Item | Value |
|---|---|
| pfSense | CE 2.9.0 |
| Suricata package | 7.0.9_1 (engine suricata-8.0.5_1) |
| Rule set | Emerging Threats Open only |
| Update interval | 1 day |
| Live rule swap on update | Off |

## Prerequisite

**System → Advanced → Networking:** hardware checksum, TCP segmentation and large receive offloading all **disabled**, then a reboot. Suricata needs offloading off to see packets as they actually crossed the wire.

## Interface settings (Services → Suricata → Interfaces → SERVERS)

| Section | Setting | Value | Why |
|---|---|---|---|
| General | Interface | SERVERS (em2) | Sees all traffic into and out of the Servers zone, including Kali → web01 |
| General | Description | SERVERS-IDS | |
| General | Block offenders | Off | IDS (alert-only) mode |
| Detection | Pattern matcher | AUTO | |
| Logging | Send alerts to system log | Off | EVE already carries alerts; this duplicated them |
| EVE output | EVE JSON log | On | |
| EVE output | Output type | SYSLOG | Sent to the Wazuh VM through pfSense remote logging |
| EVE output | Facility / priority | LOCAL1 / NOTICE | |
| EVE output | Log alerts | On | |
| EVE output | Alert payload | NO | Keeps each event under the ~1 KB syslog limit (986 usable characters) |
| EVE output | Logged traffic types | None | Wazuh only needs alerts; also keeps volume and size down |

## Rule categories (SERVERS Categories tab)

- **Resolve flowbits:** on
- **Suricata default rules:** all enabled (app-layer-events through websocket-events)
- **ET Open:** `emerging-attack_response`, `emerging-exploit`, `emerging-scan`, `emerging-web_server`

## Verification

Kali sends harmless trigger text to a netcat listener on web01 across em2:

```bash
# web01
nc -l 9000
# Kali
echo "uid=0(root) gid=0(root) groups=0(root)" | nc -q 1 10.10.20.10 9000
```

Expected: Suricata SID 1:2100498 "GPL ATTACK_RESPONSE id check returned root" in pfSense, and Wazuh rule 86601 on lab-wazuh.

## Known limits

- Only em2 is monitored; Suricata on ATTACKER (em4) is planned.
- Events longer than 986 characters would still be truncated. Check after enabling new categories:
  `sudo grep -vc '}$' /var/log/suricata-eve.json` (expect 0).
