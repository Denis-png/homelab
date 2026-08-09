# Architecture

How this homelab is put together, and how this repository relates to it.
Describes what exists today — see the README roadmap for what doesn't yet.

## The host

A single machine. A repurposed laptop, which is a deliberate constraint rather
than an aspiration: it is the hardware I have, it draws little power, and
everything here is shaped by having one node rather than a cluster.

| | |
|---|---|
| CPU | Intel i5-6300HQ, 4 cores |
| Memory | 16 GB |
| OS | Ubuntu 24.04 LTS |
| Workload | 32 containers across 11 Compose stacks |

Reachable from the LAN directly, and remotely over Tailscale. It sits behind
double NAT with no public IP, so nothing is exposed to the internet: there is
no port forwarding, and no inbound path that does not go through Tailscale.

## Storage

Five disks, split by role rather than pooled uniformly:

| Mount | Device | Purpose |
|---|---|---|
| `/` | 115 GB SSD | OS, `/var/lib/docker` |
| `/srv` | 234 GB SSD | service configuration and databases |
| `/pool` | 3.3 TB, mergerfs over 3 HDDs | bulk media |
| external | 2 × 1.8 TB USB | backups |

`/pool` is a mergerfs union rather than RAID — it presents three disks as one
namespace without striping, so a disk failure loses that disk's contents and
nothing else. For replaceable media that is the right trade; anything that
matters is backed up separately.

Databases and container state live on `/srv` (SSD) while bulk media lives on
`/pool` (spinning disks). Putting SQLite on mergerfs would be a bad idea.

## Source and runtime are separate

The single most important structural decision here:

```
/srv/infra/     ← this repository. Source of truth. Nothing runs from here.
/srv/homelab/   ← deploy target: rendered Compose files + persistent data
```

Ansible renders each stack's Compose file into `/srv/homelab/<stack>/`, which
is where it already lived before this repository existed. Every bind mount,
data directory, and path on the host is unchanged by the conversion.

That property is what makes converting a running system safe. A role can be
written, dry-run against the live host with `--check --diff`, and iterated
until it reports **zero changes** — proving it describes reality — before it is
ever allowed to alter anything.

```
   /srv/infra                                   /srv/homelab
   (git, source of truth)                       (deploy target)

   ansible.cfg           ansible-playbook      network/docker-compose.yml
   inventory/       ──────────────────────▶    media/docker-compose.yml
   roles/                                      monitoring/docker-compose.yml
   playbooks/                                  ...
        │                                              │
        │  check mode: reports, changes nothing        ▼
        └ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ─ ▶    Docker · 32 containers
                                                       │
                                     persistent data ──┘
                                     /srv (databases) · /pool (media)
```



## How Ansible reaches the host

There is one machine, and Ansible runs on it. The inventory uses
`ansible_connection: local` rather than SSH-to-self, which removes a moving
part for no loss of capability. Privilege escalation is `sudo` with a password,
so plays run with `-K`.

```
homeserver   ansible_connection: local                      become: sudo
practice     ansible_connection: community.general.incus    become: none
```

## The practice target

`practice` is a disposable Incus system container running the same Ubuntu
release as the host. New roles are developed against it and it is deleted and
recreated freely.

This is not ceremony. The host runs a password manager, a Nextcloud instance,
and home automation; a first draft of a role should not meet them. During the
`docker` role's development the practice container caught a malformed apt
sources file that would have broken package management host-wide, and a
templating bug that would have created a user with a literal `['name']`
username. Neither was caught by review.

It is reached through the Incus connection plugin rather than SSH, so it needs
no sshd, no keys, and no network configuration of its own.

## What the repository contains today

```
ansible.cfg              project defaults
requirements.yml         Galaxy collections
inventory/hosts.yml      two hosts: homeserver, practice
playbooks/site.yml       converge the homeserver
playbooks/practice.yml   converge the practice container
roles/common/            base packages, timezone
roles/docker/            Docker CE repository, engine, group membership
```

Every push runs yamllint, ansible-lint at the `production` profile, and
gitleaks over full history.

## What is not here yet

The eleven Compose stacks are still hand-managed; converting them is the work
of later phases. Secrets are not yet in the repository, encrypted or otherwise.
There is no bootstrap path from a bare OS, and no automated restore. The README
roadmap tracks all of it.
