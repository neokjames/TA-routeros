# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this is

A Splunk Technology Add-On (TA) for Mikrotik RouterOS. It ships Splunk configuration (`default/*.conf`), CSV lookups, and modular-input Python scripts that pull data from RouterOS devices via the RouterOS API. There is no test suite, no compiler, and no package manager — this is a Splunk app, not a standalone application.

## Build / package

The only "build" step is tarballing the directory for deployment into `$SPLUNK_HOME/etc/apps/`:

```
bin/build.sh   # produces ~/TA-routeros.tar.gz from ~/git/TA-routeros
```

The script assumes a Linux/macOS layout (`~/git/TA-routeros`) — adapt paths when packaging from this Windows checkout.

## Architecture

Two distinct data paths feed Splunk, and changes usually touch one or the other:

### 1. Syslog ingestion (sourcetype `routeros`)

RouterOS devices send syslog → Splunk forwarder file monitor → indexed as `sourcetype=routeros`. All parsing lives in [default/props.conf](default/props.conf) as `EXTRACT-*` regexes for each message family (dns query/response, auth ok/failed, firewall, dhcp, wifi). Field normalization to Splunk CIM happens via:

- `FIELDALIAS-*` (host→dvc, src_ip→src, dest_ip→dest)
- `LOOKUP-*` against [lookups/](lookups/) CSVs to map vendor terms to CIM `action` / `message_type`
- `EVAL-vendor_product = "Mikrotik RouterOS"`

CIM tagging is in [default/eventtypes.conf](default/eventtypes.conf) + [default/tags.conf](default/tags.conf) — eventtypes are pure search expressions over the raw syslog text, so changing log prefixes in RouterOS (see README firewall section: `log-prefix="rulename drop|accept"`) requires matching the eventtype searches.

### 2. RouterOS API polling (modular inputs, JSON sourcetypes)

Three Python modular inputs in [bin/](bin/) connect to the RouterOS API (default port 8728, *not* winbox) using the bundled [librouteros](bin/librouteros) library and emit JSON to stdout for Splunk:

- [bin/routerosint.py](bin/routerosint.py) — `/interface/getall` → `routeros:interfaces`
- [bin/routerosinv.py](bin/routerosinv.py) — inventory → `routeros:inventory`
- [bin/routerosperf.py](bin/routerosperf.py) — performance/resource stats → `routeros:perf`

Each script implements the Splunk modular-input contract: `--scheme` prints the input XML scheme, `--validate-arguments` validates, and no-arg invocation reads stanza params from stdin XML and prints events. The corresponding `[routeros:*]` stanzas in [default/props.conf](default/props.conf) use `INDEXED_EXTRACTIONS = json` and `TIME_PREFIX = time:`, so any new field emitted by the Python script is automatically indexed — there are no `EXTRACT-*` rules for these sourcetypes.

`librouteros` and `chainmap` are vendored under `bin/` so the scripts run under Splunk's bundled Python without requiring `pip install`.

## When editing

- **New syslog field**: add `EXTRACT-*` to [default/props.conf](default/props.conf) under `[routeros]`, then a `FIELDALIAS` if it needs to map to a CIM field. If it's an action/message_type vocabulary, extend the matching CSV in [lookups/](lookups/) instead of hard-coding.
- **New API-polled metric**: add a field to the Python script's JSON output; no props.conf change needed because of `INDEXED_EXTRACTIONS = json`.
- **New CIM eventtype**: add to [default/eventtypes.conf](default/eventtypes.conf) and tag it in [default/tags.conf](default/tags.conf).
- Bump `version` in [default/app.conf](default/app.conf) for releases.
