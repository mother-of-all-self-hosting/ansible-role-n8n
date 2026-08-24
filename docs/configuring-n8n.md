<!--
SPDX-FileCopyrightText: 2020 Aaron Raimist
SPDX-FileCopyrightText: 2020 Chris van Dijk
SPDX-FileCopyrightText: 2020 Dominik Zajac
SPDX-FileCopyrightText: 2020 Mickaël Cornière
SPDX-FileCopyrightText: 2020-2024 MDAD project contributors
SPDX-FileCopyrightText: 2020-2024 Slavi Pantaleev
SPDX-FileCopyrightText: 2022 François Darveau
SPDX-FileCopyrightText: 2022 Julian Foad
SPDX-FileCopyrightText: 2022 Warren Bailey
SPDX-FileCopyrightText: 2023 Alejandro AR
SPDX-FileCopyrightText: 2023 Antonis Christofides
SPDX-FileCopyrightText: 2023 Felix Stupp
SPDX-FileCopyrightText: 2023 Julian-Samuel Gebühr
SPDX-FileCopyrightText: 2023 Pierre 'McFly' Marty
SPDX-FileCopyrightText: 2024 Thomas Miceli
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up n8n

This is an [Ansible](https://www.ansible.com/) role which installs [n8n](https://github.com/n8n-io/n8n) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

n8n is a workflow automation tool for technical people.

See the project's [documentation](https://docs.n8n.io/) to learn what n8n does and why it might be useful to you.

>[!WARNING]
> n8n is licensed under [Sustainable Use License](https://github.com/n8n-io/n8n/blob/master/LICENSE.md), and therefore non-free software.

## Prerequisites

By default n8n keeps its workflows, credentials and execution history in an [SQLite](https://www.sqlite.org/) database below the service's data directory, so no other service is required.

For anything beyond a small instance, [n8n recommends](https://docs.n8n.io/hosting/configuration/supported-databases-settings/) a [Postgres](https://www.postgresql.org/) database server instead. See [Configuring the database](#configuring-the-database) below for how to switch to one.

If you are looking for an Ansible role for Postgres, you can check out [ansible-role-postgres](https://github.com/mother-of-all-self-hosting/ansible-role-postgres) maintained by the [Mother-of-All-Self-Hosting (MASH)](https://github.com/mother-of-all-self-hosting) team.

## Adjusting the playbook configuration

To enable n8n with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# n8n                                                                  #
#                                                                      #
########################################################################

n8n_enabled: true

########################################################################
#                                                                      #
# /n8n                                                                 #
#                                                                      #
########################################################################
```

### Set the hostname

To enable n8n you need to set the hostname as well. To do so, add the following configuration to your `vars.yml` file. Make sure to replace `example.com` with your own value.

```yaml
n8n_hostname: "example.com"
```

After adjusting the hostname, make sure to adjust your DNS records to point the domain to your server.

**Note**: hosting n8n under a subpath (by configuring the `n8n_path_prefix` variable) does not seem to be possible due to n8n's technical limitations.

### Configuring the database

The role installs n8n with an SQLite database by default, kept in a file below the service's data directory. This requires no other service, and is n8n's own default.

#### Switching to Postgres

To have the n8n instance store its data in your Postgres server instead, add the following configuration to your `vars.yml` file.

```yaml
n8n_database_type: postgres

n8n_database_username: YOUR_POSTGRES_SERVER_USERNAME_HERE
n8n_database_password: YOUR_POSTGRES_SERVER_PASSWORD_HERE
n8n_database_name: YOUR_POSTGRES_SERVER_DATABASE_NAME_HERE
```

Make sure to replace the placeholders with your own values.

>[!WARNING]
> Switching an existing instance to Postgres does not carry its data across. n8n starts from an empty Postgres database, so your workflows, credentials and execution history will appear to be gone. The SQLite database is left untouched on the server (at `/n8n/data/.n8n/database.sqlite` by default), so nothing is lost, but moving the data over is up to you — see [n8n's documentation on the subject](https://docs.n8n.io/hosting/configuration/supported-databases-settings/).
>
> Until this role learned about `n8n_database_type`, it passed the `n8n_database_*` settings to n8n without also passing `DB_TYPE`, which n8n needs in order to use anything but SQLite. Instances configured for Postgres back then were in fact running on SQLite all along. Should you be in that position, the role now refuses to run until you say which of the two databases you mean to use.

#### Configuring connection to the database server (optional)

By default the role is configured to establish connection with the Postgres server via the Unix socket. You can mount the Unix socket by adding the following configuration to your `vars.yml` file:

```yaml
# Specify the path to the Postgres Unix socket path on the host (bind-mount source)
n8n_database_socket_path_host: ""
```

Setting it enables to connect to the Postgres server via Unix socket mounted in the container at `/run-postgres/.s.PGSQL.5432`.

If TCP connection is preferred, connection via the Unix socket can be disabled by adding the following configuration to your `vars.yml` file:

```yaml
# Disable the connection to Postgres server via a Unix socket
n8n_database_socket_enabled: false

n8n_database_hostname: YOUR_POSTGRES_SERVER_HOSTNAME_HERE
n8n_database_port: 5432
```

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can override settings (even those that don't have dedicated playbook variables) using the `n8n_environment_variables_additional_variables` variable

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, n8n becomes available at the specified hostname like `https://example.com`.

To get started, open the URL with a web browser, and register the account. **Note that the first registered user becomes an administrator automatically.**

## Troubleshooting

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu n8n` (or how you/your playbook named the service, e.g. `mash-n8n`).
