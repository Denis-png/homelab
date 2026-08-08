# homelab

Infrastructure-as-code for a single-node home server: Docker Compose stacks
provisioned with Ansible, where service routes, DNS, monitoring targets, and
documentation are all generated from one declarative inventory.

> **Status: in progress.** This repository is being built in public, converting
> a hand-managed homelab into code one stack at a time. The roadmap below marks
> what has actually landed. Nothing is claimed before it works.

## The setup

An Ubuntu laptop behind double NAT, reachable over Tailscale, running media
services, file sync, home automation, monitoring, and self-hosted git. Storage
spans six disks; everything is served through a single Traefik instance with
wildcard TLS via a Cloudflare DNS challenge.

## Design principles

- **One source of truth.** `inventory/services.yml` describes every service.
  Traefik routes, AdGuard DNS rewrites, Prometheus scrape targets, and the
  service tables in these docs are generated from it. Adding a service is a
  data change, not six edits in six places.
- **Secrets encrypted in-repo.** SOPS + age. This repository is complete — a
  clone plus the age key reproduces the host. Nothing is kept out of band.
- **Reproducible, not merely documented.** `playbooks/bootstrap.yml` takes a
  bare OS to a running host.
- **Changes are proven before they are applied.** Conversions run under
  `ansible-playbook --check --diff` against the live host until the diff is
  empty, proving the change is a no-op before it is real.
- **Documentation cannot drift.** CI regenerates it from the inventory and
  fails the build if the committed copy differs.

## Target layout

```
inventory/     hosts, service inventory, encrypted group_vars
playbooks/     site, bootstrap, restore
roles/         one per stack, plus common and docker
docs/          architecture, runbooks, decision records
```

## Roadmap

- [x] **Phase 0** — repository scaffold, CI, secret scanning
- [ ] **Phase 1** — Ansible foundation: inventory, `common` and `docker` roles
- [ ] **Phase 2** — first stack converted end to end
- [ ] **Phase 3** — service inventory and templated Traefik routes
- [ ] **Phase 4** — SOPS-encrypted secrets
- [ ] **Phase 5** — remaining stacks
- [ ] **Phase 6** — generated documentation enforced by CI
- [ ] **Phase 7** — alert routing, runbooks, SLOs
- [ ] **Phase 8** — automated disaster-recovery drill

## Why this exists

The previous iteration of this homelab was managed by hand-edited Compose files.
It worked, but it drifted: at the point this rebuild started, only 13 of 22
service hostnames matched what the documentation claimed. Generating
configuration and documentation from shared data is the fix, and this repository
is that fix applied.