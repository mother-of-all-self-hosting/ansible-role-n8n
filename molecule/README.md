<!--
SPDX-FileCopyrightText: 2018-2025 Slavi Pantaleev
SPDX-FileCopyrightText: 2019-2022 Aaron Raimist
SPDX-FileCopyrightText: 2019-2023 MDAD project contributors
SPDX-FileCopyrightText: 2023 QEDeD
SPDX-FileCopyrightText: 2024 Fabio Bonelli
SPDX-FileCopyrightText: 2024 Nikita Chernyi
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara
SPDX-FileCopyrightText: 2026 spatterlight

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Molecule Testing

This role supports [Molecule](https://docs.ansible.com/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

## Prerequisites

To utilize Molecule you need to prepare several requirements:

- **x86** computer running one of these operating systems that make use of [systemd](https://systemd.io/):
  - **Archlinux**
  - **CentOS**, **Rocky Linux**, **AlmaLinux**, or possibly other RHEL alternatives (although your mileage may vary)
  - **Debian** (10/Buster or newer)
  - **Ubuntu** (18.04 or newer, although [20.04 may be problematic](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/ansible.md#supported-ansible-versions) if you run the Ansible playbook on it)
- `root` access on the computer which Molecule runs against
- [Ansible](http://ansible.com/) program
- [Python](https://www.python.org/)
  - Most distributions install Python by default, but some don't (e.g. Ubuntu 18.04) and require manual installation (something like `apt-get install python3`)
- [Docker](https://www.docker.com)
  - Access to Docker UNIX socket (`/var/run/docker.sock`) is required by default

## Installation

To set up the environment for using Molecule, run the command below on the terminal:

```bash
python3 -m venv ./molecule/venv
source ./molecule/venv/bin/activate
pip3 install -r ./molecule/requirements.txt
```

## What the scenarios verify

Each scenario sets n8n up first, in `side_effect.yml`, by posting the owner account to `/rest/owner/setup` - the same endpoint the setup wizard itself uses. That is done there rather than in `converge.yml` because it is a one-time state change that would make converging non-idempotent. n8n answers `400` once an owner exists, which keeps the playbook safe to re-run.

`side_effect.yml` then logs in, issues an API key, and uses n8n's documented public API to create a webhook-triggered workflow and activate it. Before doing so, it records that the webhook path answers `404`, so that the `200` the verifier later gets from it means the workflow was really activated, rather than n8n answering everything.

Here is a synopsis of what a successful run proves. Check the scenarios themselves for details about what are exactly checked.

- The systemd service is active and n8n answers over HTTP
- n8n is set up rather than serving the owner-setup wizard - it advertises `showSetupOnFirstLoad` to its own frontend, `true` until an owner exists
- n8n refuses to list workflows without authentication, and rejects a wrong password, before the right one is shown to be accepted
- The version n8n reports matches `n8n_version` from the role's defaults. n8n serves it only to an authenticated caller, so this assertion is unreachable on an instance that was never set up
- n8n names the database engine it connected to, and it is the one the scenario configured
- The n8n public API rejects an invalid API key, and returns the workflow - active - to a valid one issued after that workflow was created
- An HTTP request to the workflow's webhook runs the workflow and comes back carrying the marker its second node produces
- n8n's own execution history records that run as a successful execution in `webhook` mode

## Scenarios

Currently these testing scenarios are available:

### `default`

Tests an n8n installation with SQLite and asserts that the database was created below the role's data path.

### `postgres`

Tests an n8n installation backed by Postgres, connected over a Unix socket. This scenario looks for the workflow, and its execution in Postgres requires the SQLite database that a fallback would have produced to be absent.

## Running

By default it is configured to run the scenarios on Ubuntu 26.04.

```bash
molecule test --scenario-name default
```

You can utilize other distributions by setting one to the `MOLECULE_DISTRO` environment variable:

```bash
# Ubuntu 24.04
MOLECULE_DISTRO=ubuntu2404 molecule test --scenario-name default

# Debian 13
MOLECULE_DISTRO=debian13 molecule test --scenario-name default

# Debian 12
MOLECULE_DISTRO=debian12 molecule test --scenario-name default
```
