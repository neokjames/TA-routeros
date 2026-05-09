# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Splunk Technology Add-On (TA) for Mikrotik RouterOS. It ships Splunk configuration (`default/*.conf`) and CSV lookups that parse and CIM-normalize RouterOS syslog events. There is no test suite, no compiler, and no package manager — this is a Splunk app, not a standalone application.

## Build / package

The only "build" step is tarballing the directory for deployment into `$SPLUNK_HOME/etc/apps/`:

```
bin/build.sh   # produces ~/TA-routeros.tar.gz from ~/git/TA-routeros
```

The script assumes a Linux/macOS layout (`~/git/TA-routeros`) — adapt paths when packaging from this Windows checkout.

## Architecture

RouterOS devices send syslog → Splunk forwarder file monitor → indexed as `sourcetype=routeros`. All parsing lives in [default/props.conf](default/props.conf) as `EXTRACT-*` regexes for each message family (dns query/response, auth ok/failed, firewall, dhcp, wifi). Field normalization to Splunk CIM happens via:

- `FIELDALIAS-*` (host→dvc, src_ip→src, dest_ip→dest, host→dest for auth events)
- `LOOKUP-*` against [lookups/](lookups/) CSVs to map vendor terms to CIM `action` / `message_type`
- `EVAL-vendor_product = "Mikrotik RouterOS"`

CIM tagging is in [default/eventtypes.conf](default/eventtypes.conf) + [default/tags.conf](default/tags.conf) — eventtypes are pure search expressions over the raw syslog text, so changing log prefixes in RouterOS (see README firewall section: `log-prefix="rulename drop|accept"`) requires matching the eventtype searches.

Targeted CIM data models: Authentication, Network_Traffic, Network_Resolution (DNS), Network_Sessions (DHCP).

## When editing

- **New syslog field**: add `EXTRACT-*` to [default/props.conf](default/props.conf) under `[routeros]`, then a `FIELDALIAS` if it needs to map to a CIM field. If it's an action/message_type vocabulary, extend the matching CSV in [lookups/](lookups/) instead of hard-coding.
- **New CIM eventtype**: add to [default/eventtypes.conf](default/eventtypes.conf) and tag it in [default/tags.conf](default/tags.conf).
- Bump `version` in [default/app.conf](default/app.conf) for releases.
