# MISP Initial Configuration: Feeds, Orgs, and Scheduling

## Overview

This document covers the post-installation configuration of MISP — the work that turns a fresh install into a functional threat intelligence platform. It is written for a portfolio audience, focusing on feed curation strategy, organizational structure, and the operational decisions that make MISP useful rather than just running.

Installing MISP is step one. Configuring it to pull relevant feeds, suppress false positives via warninglists, schedule metadata updates, and organize events by source is where the operational value lives.

## Configuration Scope

| Area | What's Configured |
|------|-------------------|
| Organisation | Default org identity, admin user setup |
| Users & Roles | Role-based access for automation vs. interactive use |
| OSINT Feeds | Community threat intel sources (abuse.ch, CIRCL, Botvrij) |
| Warninglists | Known-good infrastructure filtering (CDNs, DNS resolvers) |
| Taxonomies | Standardized classification (TLP, admiralty-scale, type) |
| Scheduled Tasks | Feed pull frequency, cache updates, correlation |

## Design Decisions

- **Selective feed enablement over "enable all":** Only feeds relevant to the lab's threat model are activated. This avoids IOC bloat and keeps export files manageable for downstream Cribl enrichment.
- **Warninglists enabled immediately:** Without warninglists, feeds include Cloudflare, Google DNS, and AWS IPs as "malicious." Enabling warninglists before the first feed pull prevents false positive pollution from day one.
- **Automation-friendly API key:** A dedicated sync user with API access supports scheduled exports without using the admin account's credentials.
- **Hourly feed pulls with daily correlation:** Balances freshness against processing load on a single-VM deployment.

## What This Demonstrates

Threat intelligence platform configuration requires more thought than just enabling feeds. This shows feed curation, false positive management, scheduling design, and preparation for automated downstream consumption — the operational practices that make a TIP useful in a security monitoring pipeline.

---

Sanitized for public portfolio use.

For step-by-step configuration procedures, see [OPERATIONS.md](./OPERATIONS.md).
For the base MISP installation, see the [MISP-Runbook](https://github.com/jared-labs/MISP-Runbook) repo.
