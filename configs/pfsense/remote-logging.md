# pfSense remote logging

**Status → System Logs → Settings.** Verified Oct 26, 2026.

| Setting | Value |
|---|---|
| Log message format | BSD (RFC 3164) |
| Enable remote logging | On |
| Source address | SECURITY (so syslog leaves from 10.10.30.1) |
| IP protocol | IPv4 |
| Remote log server | 10.10.30.10:514 (rsyslog relay on lab-wazuh) |
| Remote syslog contents | System Events, Firewall Events |

Suricata's EVE output (facility LOCAL1) travels with **System Events**; there is no separate checkbox for it.

On the receiving side, the relay accepts messages only from 10.10.30.1, so the source address setting matters: changing it would make the relay drop everything. See [`configs/wazuh/10-pfsense.conf`](../wazuh/10-pfsense.conf).
