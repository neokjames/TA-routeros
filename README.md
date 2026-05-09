# Technology Add-On for Mikrotik RouterOS

A Splunk Technology Add-On that parses Mikrotik RouterOS syslog events and normalizes them to the Splunk Common Information Model (CIM).

Supports both RouterOS v6 and v7 syslog formats. CIM data models covered: **Authentication**, **Network_Traffic** (firewall: forward, srcnat, dstnat chains), **Network_Resolution** (DNS), and **Network_Sessions** (DHCP).

## Activation of logging on RouterOS

Send RouterOS events to a syslog server (a Splunk forwarder running a UDP/TCP input, or a syslog daemon Splunk monitors).

Create a logging action and profile via the RouterOS CLI:

```
/system logging action
add name=splunk remote=1.2.3.4 remote-port=5200 src-address=2.3.4.5 target=remote

/system logging
set 0 topics=info,!firewall
add action=splunk topics=info
add action=splunk topics=warning
add action=splunk topics=critical
add action=splunk topics=error
add action=splunk topics=dns,!debug
```

## Firewall logging

Logging must be enabled per rule. Set `log=yes` and use a `log-prefix` of the form `"<rulename> <action>"` where action is `accept` or `drop` (RouterOS v7 also emits `forward`, `srcnat`, `dstnat` as the chain).

Drop rule example:

```
add action=drop chain=input connection-nat-state=!dstnat connection-state=new in-interface=ether1 log=yes log-prefix="input_wan drop"
```

Forward (accept) rule example:

```
add action=accept chain=forward dst-address=0.0.0.0/0 in-interface=mynet log=yes log-prefix="fwd_bw_internet accept" out-interface=ether1 src-address=192.168.1.0/24
```

## Installation

Place this TA in `$SPLUNK_HOME/etc/apps/` on your search heads, indexers, and any forwarders that ingest the syslog stream.

Example `inputs.conf`:

```
[monitor:///var/log/cases/routeros/*/*.log]
index = batchworks
sourcetype = routeros
host_segment = 5
disabled = 0
```

Verify ingestion and CIM normalization:

```
sourcetype=routeros tag=network tag=communicate
sourcetype=routeros tag=authentication
```

## Credits

Forked from [schose/TA-routeros](https://github.com/schose/TA-routeros) by Andreas Roth. This fork removes the RouterOS API modular inputs to focus on syslog ingestion, adds RouterOS v7 event format support, and tightens CIM Network_Traffic compliance.
