# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## Status

The repo currently contains only the design spec, `docs/design.md`. No implementation exists yet. Treat that document as the source of truth for what to build, including its Conventions section. When an implementation decision differs from it, update the design doc in the same change.

## What this is

This is infrastructure-as-code for a proof of concept: a self-hosted [Chatto](https://docs.chatto.run) server (Slack alternative) for Kalkspace. It runs on a single Hetzner Cloud VM. The same repo deploys two environments: `test` (`chat.cyb3rk4t.me`, Kai's Hetzner project) and later `prod` (`chat.kalk.space`, Kalkspace's Hetzner project).

## Architecture (three layers, strict boundaries)

1. **OpenTofu (`terraform/`, `hcloud` provider):** creates the server, Primary IPv4/IPv6, Volume (attached via a separate attachment resource), Cloud Firewall and SSH key. One workspace per environment, with `envs/<env>.tfvars` (committed) and `envs/<env>.secret.tfvars` (gitignored, holds `hcloud_token`). `apply` generates `ansible/inventory/<env>/hosts.yml` from the outputs (IPs and domain). The state is local and gitignored. There is no cloud-init: Hetzner injects the root SSH key, and Ansible does the rest.
2. **Ansible (`ansible/`):** sets up the host and deploys. Roles: `base`, `volume`, `docker`, `chatto`, `backup`, `beszel`. Each environment has its own inventory directory (`inventory/<env>/group_vars/all/{vars.yml,vault.yml}`) and its own vault id. Pinned image versions live in `ansible/group_vars/all/versions.yml` and are shared by all environments.
3. **Docker Compose (`compose/`):** runs Caddy (TLS) → Chatto → NATS JetStream, plus LiveKit for calls. There is no Postgres: all Chatto data, **including attachments**, lives in NATS JetStream. It is bind-mounted from the Hetzner Volume at `/srv/chatto`. Call media bypasses Caddy and goes straight to LiveKit (UDP 7882, UDP 3478, TCP 7881).

## Commands

All operator commands go through the root `justfile` (not a Makefile). The environment is always a required argument with no default:

- `just lint`: `tofu fmt -check`, `tofu validate`, `ansible-lint`, `yamllint`
- `just plan <env>`, `just apply <env>`
- `just secrets-init <env>`, `just deploy <env>`
- `just backup-now <env>`, `just restore <env> <archive>`, `just reset <env>` (test only)

`just --list` shows all recipes, and `mise install` installs the pinned tools. Use these once they exist. Multi-step recipes should be shebang (`#!/usr/bin/env bash`) recipes rather than line-by-line.

Bootstrap order matters: `apply` → set the DNS records and confirm they resolve → `deploy` → immediately complete Chatto's setup wizard. Caddy's ACME fails without DNS, and the first visitor can become owner until the wizard is done.

## Conventions

- **English everywhere:** docs, README, code, comments, variable names, commits, PRs.
- **Git:** short-lived branch plus PR into `main` for every change, squash merge. Use Conventional Commits with the role or layer as scope (`feat(chatto): …`, `fix(backup): …`, `chore(deps): …`).
- **Ansible variables:**
  - Every variable is prefixed with its role name (`chatto_*`, `backup_*`, `beszel_*`).
  - Secrets are named `vault_*`, live only in `vault.yml`, and are referenced only from the environment's `vars.yml`.
  - Feature flags have defaults in role defaults and are set explicitly per environment.
- **Secrets are generated only by `just secrets-init <env>`**, locally, and never overwritten. Deploys only read secrets. Never write decrypted secrets into the repo.
- **Pinning:**
  - Images: tag plus digest (`image:tag@sha256:…`) in `versions.yml`.
  - Tools: in `mise.toml`.
  - Collections: in `ansible/requirements.yml`.
  - Providers: with `~>`, and `.terraform.lock.hcl` is committed.
- **Definition of done:** `just lint` passes, `just deploy test` succeeds, and a second `just deploy test` reports `changed=0`.

## Invariants to preserve

- **Never `docker compose down`** (it stops NATS), in Ansible or anywhere else. Use `docker compose up -d`. Never copy `/srv/chatto/nats` while it is running.
- **Upstream files stay unmodified:** `compose/compose.yml`, `Caddyfile` and `nats-server.conf` are copied verbatim from Chatto's `examples/dockercompose` (v0.5.0), plus the header defined in the design doc. All customization goes into `compose.override.yml`, `env.j2` and `livekit.yaml.j2`.
- **Rebuild safety:** Ansible must run against a fresh VM with an existing Volume without touching existing data. Volume and Primary IP have `prevent_destroy` plus Hetzner delete protection. Mount the Volume via `/dev/disk/by-id/…`, not Hetzner automount.
- **The Hetzner Cloud Firewall is the only filter** (there is no host firewall). Docker may publish only the ports in the design doc's firewall table. NATS (4222) and Chatto (4000) must never be exposed.
- **Nothing environment-specific in shared code.** Everything environment-specific goes in `envs/<env>.tfvars` or `inventory/<env>/`.
- **`reset.yml`** only runs when the inventory sets `chatto_env: test`, and it asks for confirmation first.
- **Never commit** Slack exports, import archives or local backup copies.
