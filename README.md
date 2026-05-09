# Zabbix Template — Fortinet FortiGate Firewall

A generic Zabbix 7.4 template for monitoring Fortinet FortiGate firewalls via SNMP v2c.
Designed to work across multiple sites with different configurations.

## What it monitors

**System**
- Management CPU usage (triggers at configurable thresholds)
- IPS/AV engine CPU usage
- Memory usage (triggers at configurable thresholds)
- Active session count
- Firmware version
- System uptime (restart detection)
- HA cluster mode

**Network Interfaces (auto-discovered)**
- Traffic in/out per interface (bps, 64-bit counters)
- Operational status with down alert and auto-recovery
- Filters out internal virtual interfaces (ssl.root, naf.root, etc.)

**WAN Interfaces (auto-discovered)**
- Separate discovery rule for WAN-specific interfaces
- Combined traffic in/out graph prototype per interface
- Adjust the filter regex to match your WAN interface names

**VPN Tunnels (auto-discovered)**
- IPSec tunnel status (up/down) with HIGH alert and auto-recovery
- Traffic in/out per tunnel (bps)

**SD-WAN Performance SLA (auto-discovered)**
- RTT in ms (trigger at configurable threshold)
- Jitter in ms (trigger at configurable threshold)
- Packet loss % (trigger at configurable threshold)
- MOS score
- Filter set to match specific SLA names — adjust to your environment

**Temperature Sensors (auto-discovered)**
- Hardware sensor readings in Celsius
- Triggers at warning and critical thresholds
- Note: not all FortiGate models expose temperature sensors via SNMP

**Security (optional)**
- IPS intrusion detections
- AV virus detections
- Disable these items if your model does not support them

**SSL Certificates**
- Discovery rule included but disabled by default
- Requires either web.certificate.get or an external script
- See SSL Certificate Setup section below

**ICMP**
- Ping availability (value map: Reachable / Not reachable)
- Ping response time

## Requirements

- Zabbix 7.4 or higher
- Zabbix Agent not required — uses SNMP only
- SNMP v2c enabled on FortiGate: System > SNMP
- UDP port 161 open between Zabbix server/proxy and FortiGate
- MIBs: FORTINET-FORTIGATE-MIB, IF-MIB (standard)

## Installation

1. Import `fortigate_zabbix_template.xml` via Data collection > Templates > Import
2. Create a host for each firewall
3. Add an SNMP interface (port 161) with the management IP
4. Override the macro `{$SNMP_COMMUNITY}` at host level with your community string
5. Run LLD rules manually: Hosts > [host] > Discovery rules > Execute now

## Macros

All thresholds are configurable via macros. Override at host level for site-specific values.

| Macro | Default | Description |
|-------|---------|-------------|
| {$SNMP_COMMUNITY} | public | SNMP v2c community string |
| {$CPU_WARN} | 80 | CPU warning threshold (%) |
| {$CPU_CRIT} | 95 | CPU critical threshold (%) |
| {$MEM_WARN} | 85 | Memory warning threshold (%) |
| {$TEMP_WARN} | 65 | Temperature warning threshold (°C) |
| {$TEMP_CRIT} | 80 | Temperature critical threshold (°C) |
| {$SLA_LATENCY_WARN} | 100 | SLA RTT warning threshold (ms) |
| {$SLA_JITTER_WARN} | 50 | SLA jitter warning threshold (ms) |
| {$SLA_PACKETLOSS_WARN} | 5 | SLA packet loss warning threshold (%) |
| {$SSL_CERT_WARN_DAYS} | 30 | Days before SSL cert expiry to warn |
| {$SSL_CERT_CRIT_DAYS} | 7 | Days before SSL cert expiry - critical |

## Adjustments required before use

**WAN Interface filter**
Edit the WAN Interface Discovery rule filter to match your WAN interface names:
```
^(WAN-LAG|MPLS|wan1|wan2)$
```

**SD-WAN SLA filter**
Edit the SD-WAN Performance SLA Discovery rule filter to match your SLA names:
```
^YOUR_SLA_NAME
```

**Interface indexes**
Verify interface indexes with snmpwalk before importing:
```bash
snmpwalk -v2c -c YOUR_COMMUNITY YOUR_FORTIGATE_IP 1.3.6.1.2.1.31.1.1.1.1
```

## SSL Certificate Setup

The SSL certificate discovery rule is disabled by default. Two options to enable it:

**Option A — web.certificate.get (Zabbix 5.4+)**
Create an item of type Zabbix agent with key:
```
web.certificate.get[https://YOUR_FORTIGATE_IP:PORT]
```
Use dependent items to extract expiry date via JSONPath `$.x509.not_after`.

**Option B — External script**
Create `/usr/lib/zabbix/externalscripts/check_ssl_cert.sh`:
```bash
#!/bin/bash
expiry=$(echo | openssl s_client -connect $1:$2 -servername $1 2>/dev/null | openssl x509 -noout -enddate 2>/dev/null | sed 's/notAfter=//')
expiry_epoch=$(date -d "$expiry" +%s)
now_epoch=$(date +%s)
echo $(( ($expiry_epoch - $now_epoch) / 86400 ))
```

## Important notes

- SD-WAN Performance SLA OIDs: FortiOS with SD-WAN uses `fgSdwanSlaTable` (`.4.9`) instead of the older `fgLinkMonitorTable` (`.4.8`). This template uses the correct OID path.
- Security items (IPS/AV) may show "not supported" on models without the relevant license — disable them if not needed.
- Temperature sensor discovery may return no data on some FortiGate models.

## Tested on

- FortiOS 7.x
- Zabbix 7.4

## License

MIT License — free to use, modify and distribute.
