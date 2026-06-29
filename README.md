# Self-Hosted Infrastructure & AI Agent Platform

A hardened, self-hosted multi-service platform with a network-isolated AI agent, deployed and operated on a Linux VPS — captured end-to-end as reproducible infrastructure-as-config.

> **Status — work in progress.** This is the initial scaffold. Architecture overview, a security write-up, a diagram, and sanitized config are being added.

## What this is

A personal, self-directed infrastructure project: a fleet of containerized services behind a single reverse proxy with automatic HTTPS and single sign-on, plus a sandboxed autonomous AI agent isolated on its own Docker network. The whole stack is defined in Docker Compose, so it survives reboots and rebuilds.

## Tech stack

`Linux` · `Ubuntu` · `Docker` · `Docker Compose` · `Caddy` · `reverse proxy` · `HTTPS / Let's Encrypt` · `DNS` · `network isolation` · `SSH hardening` · `ufw` · `fail2ban` · `PostgreSQL` · `DigitalOcean / VPS` · `single sign-on` · `infrastructure-as-config`

## Roadmap

- [ ] Architecture overview and diagram
- [ ] Security write-up — SSH hardening, network segmentation, secrets isolation
- [ ] Sanitized stack config (docker-compose.example.yml, Caddyfile.example)

---

*Personal project, built and operated solo. Demonstrates foundational, junior-level infrastructure skills through hands-on practice.*
