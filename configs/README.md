# Configuration

The configuration that makes the detection pipeline work, as running in the lab. Firewall rules are documented in the main README and will be published under `network/` once they are stable.

| File | Runs on | Purpose |
|---|---|---|
| [`wazuh/10-pfsense.conf`](wazuh/10-pfsense.conf) | lab-wazuh | rsyslog relay: adds pfSense's hostname, splits Suricata JSON into its own file |
| [`wazuh/ossec-localfiles.xml`](wazuh/ossec-localfiles.xml) | lab-wazuh | The two log inputs Wazuh reads from the relay |
| [`pfsense/remote-logging.md`](pfsense/remote-logging.md) | pfSense | Where and how pfSense sends syslog |
| [`pfsense/suricata-servers.md`](pfsense/suricata-servers.md) | pfSense | Suricata settings and rule categories on the SERVERS interface |

None of these files contain credentials. Addresses are the lab's private 10.10.x.x ranges.
