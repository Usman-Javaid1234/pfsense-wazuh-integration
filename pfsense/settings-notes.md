# pfSense Settings Notes

The settings I used in this lab, in the order I configured them. This is a record of what I did, not a full pfSense guide. Menu names are from pfSense CE 2.7.2 and pfBlockerNG-devel.

## 1. Network layout

| Item | Value |
|---|---|
| WAN (em0) | VirtualBox NAT adapter, DHCP |
| LAN (em1) | VirtualBox Host-only adapter |
| LAN subnet | 192.168.56.0/24 |
| pfSense LAN IP | 192.168.56.2 |
| Host adapter | 192.168.56.1 |
| Ubuntu client | 192.168.56.102 (DHCP) |
| Wazuh manager | 192.168.56.103 |

Bridged mode did not work on campus Wi-Fi, so WAN is on NAT and LAN is on Host-only.

## 2. DHCP server (Services > DHCP Server > LAN)

- DHCP enabled on LAN.
- **Gateway:** 192.168.56.2
- **DNS Servers:** 192.168.56.2

Both fields were blank at first and the Ubuntu client had no route or DNS. Setting both to pfSense's own LAN IP is what makes the client's traffic and DNS lookups pass through the firewall. Without this, the GeoIP and DNSBL rules never see the client's traffic.

## 3. DNS Resolver (Services > DNS Resolver)

Unbound is the resolver, and it answers the client's DNS queries. pfBlockerNG DNSBL hooks into Unbound, which is how it can return a sinkhole address for blocked domains.

## 4. pfBlockerNG-devel (Firewall > pfBlockerNG)

### General
- pfBlockerNG enabled.
- Keep Settings on.
- **Force Global IP Logging** turned on, so blocks are logged.

### IP > GeoIP
- MaxMind GeoLite2 license key entered under IP settings (not included in this repo).
- Update frequency set to something other than "Never". It was "Never" at first and no rules were created.

### IP > Custom list (`China_russia_block`)
- Countries selected: `CN` and `RU` only, not whole continents.
- Action: **Deny Inbound**.
- Logging enabled.
- Result: an alias of about 21,026 ranges and an auto-created rule `pfB_China_russia_block_v4`.
- After saving, run **Update > Force Update** so the list is built.

### DNS > DNSBL
- DNSBL enabled, virtual IP (sinkhole) `10.10.10.1`.
- Added a custom block entry for the test domain.
- **DNSBL Whitelist:** the default list includes big domains like Google and YouTube. I removed the matching entries, otherwise the block is ignored.
- Run a Force Reload after changing the lists.

## 5. Firewall rules and logging

- Per-rule logging enabled on the pfBlockerNG rule.
- An ICMP block rule on LAN stopped the Ubuntu client from pinging pfSense. I disabled it.
- Firewall log: Status > System Logs > Firewall.

## 6. Admin access control (System > User Manager)

- A second user in a restricted group, separate from the full `admin` account.
- Only the privileges needed for the lab are assigned to that group.

## 7. Log forwarding (Status > System Logs > Settings)

- **Enable Remote Logging:** on.
- **Remote log server:** `192.168.56.103:514`
- **Remote Syslog Contents:** Firewall Events.
- Protocol is UDP (the pfSense default).

## 8. Wazuh manager side

In `/var/ossec/etc/ossec.conf`:

```xml
<remote>
  <connection>syslog</connection>
  <port>514</port>
  <protocol>udp</protocol>
  <allowed-ips>192.168.56.2</allowed-ips>
</remote>
```

Then restart the manager: `systemctl restart wazuh-manager`.

To confirm syslog is reaching the manager: `tcpdump -i any udp port 514`.

The decoder and rules are in `wazuh/local_decoder.xml` and `wazuh/local_rules.xml`. I tested them with `/var/ossec/bin/wazuh-logtest` before relying on live traffic.

## 9. Quick verification checklist

| Test | Command (from the Ubuntu client) | Expected |
|---|---|---|
| Gateway reachable | `ping 192.168.56.2` | Replies |
| GeoIP block | `ping 1.0.1.0` | 100% loss, `CN` block in pfSense firewall log |
| DNSBL block | `nslookup <blocked domain>` | Returns `10.10.10.1` |
| Normal DNS | `nslookup <any other domain>` | Real address |
| Wazuh alert | check `wazuh-alerts-*` in the dashboard | Rule `100501` |

## 10. Known gaps

- DNSBL events are logged by pfBlockerNG/Unbound, not as `filterlog`, so they do not reach Wazuh yet.
- Syslog is UDP, so messages can be lost under load.