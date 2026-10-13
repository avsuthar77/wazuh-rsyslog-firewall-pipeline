# wazuh-rsyslog-firewall-pipeline

How I receive WatchGuard Firebox syslog with rsyslog, filter the noise,
and feed it to Wazuh. Pairs with
[wazuh-watchguard-decoders](https://github.com/avsuthar77/wazuh-watchguard-decoders).

## Flow
Firebox -> UDP syslog -> rsyslog (filter + template) -> log file -> Wazuh

## What the config does
- Listens on UDP (port 5141 in this example) via `imudp`
- Drops noisy internal messages (configd_*, SIGNALED, etc.)
- Keeps only the event types Wazuh needs: authentication, alias/policy
  changes, and traffic logs
- Writes each line with the sender IP and firewall name first (the
  decoders depend on this format)
- Creates the file as `root:wazuh`, mode `0640`, so Wazuh can read it

## Setup
1. Copy `configs/rsyslog-watchguard.conf` to `/etc/rsyslog.d/`
2. `mkdir -p /var/log/watchguard`
3. Open the port: `firewall-cmd --permanent --add-port=5141/udp && firewall-cmd --reload`
4. Add `configs/ossec-localfile.xml` to `ossec.conf`
5. `rsyslogd -N1` (syntax check), then `systemctl restart rsyslog wazuh-manager`

## Test
- `ss -ulnp | grep 5141`
- `tail -f /var/log/watchguard/watchguard-firewall.log`
- Run a log line through `/var/ossec/bin/wazuh-logtest`

## Troubleshooting
- Wrong file owner: Wazuh runs as the `wazuh` user. A root-owned log
  file made an integration fail silently for me. Test with
  `sudo -u wazuh ...`, because running as root hides permission errors.
- Exposed port: restrict it in the firewall to the firewall's source IP.
- Order matters: rsyslog `if` rules run top to bottom and `stop` ends
  processing, so drop rules must come before keep rules.

## Note
Port and paths are examples. No client data is included. The config is
from a working setup; the setup steps above are generic and not
re-verified step by step.
