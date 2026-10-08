# Phase 5 — Wazuh SIEM (Centralised Detection Platform)

## Objective

Deploy the **SIEM** — the central platform that will aggregate, correlate, and surface security events from across the lab on a single dashboard.

## Build

Provisioned a dedicated **Ubuntu Server** VM on the lab network at `10.10.10.101` and installed **Wazuh** in its all-in-one configuration, which bundles the three components of the platform:

1. **Manager** — the analysis engine that decodes incoming events, matches them against rules, and raises alerts.
2. **Indexer** — the event datastore (the searchable backend that holds the telemetry).
3. **Dashboard** — the web interface for triage, searching, and visualisation.

## Verification — the meaningful "before" state

On completion, the dashboard was live but **reporting zero agents and zero events** — and that is the correct, meaningful baseline. The SIEM has no visibility until data sources are wired into it (Phase 7). Capturing this clean "before" state matters: it is what makes the "after" — agents reporting, alerts firing during a live attack — demonstrably the result of the pipeline, not pre-existing noise.

## Real-world relevance

A SIEM is the nerve centre of any Security Operations Centre. Standing one up from scratch — manager, indexer, and dashboard — and understanding that it is only as useful as the telemetry feeding it is foundational SOC-analyst knowledge. This phase builds the platform; the following phases give it something to watch.

*Evidence: see `/screenshots` for the Wazuh dashboard at the zero-agent baseline.*
