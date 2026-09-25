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
SPDX-FileCopyrightText: 2023 Antonis Christofides
SPDX-FileCopyrightText: 2023 Felix Stupp
SPDX-FileCopyrightText: 2023 Pierre 'McFly' Marty
SPDX-FileCopyrightText: 2024-2026 Suguru Hirahara

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up APISIX

This is an [Ansible](https://www.ansible.com/) role which installs [APISIX](https://apisix.apache.org/docs/apisix/getting-started/README/) to run as a [Docker](https://www.docker.com/) container wrapped in a systemd service.

APISIX is an [API Gateway](https://apisix.apache.org/docs/apisix/terminology/api-gateway/) and Ingress Controller.

APISIX has a complex [architecture](https://apisix.apache.org/docs/apisix/architecture-design/apisix/) in which APISIX can serve multiple roles (data plane, control plane). There are different [deployment modes](https://apisix.apache.org/docs/apisix/deployment-modes/) for achieving a more decoupled setup.

See the project's [documentation](https://apisix.apache.org/docs/) to learn what APISIX does and why it might be useful to you.

>[!NOTE]
> What we're configuring here is a `traditional` deployment in which one APISIX instance acts as both the data plane and the control plane. By tweaking the configuration, you may be able to install multiple instances (on separate machines), each serving a different role. This is beyond the scope of this documentation page.

## Prerequisites

To run an APISIX instance it is necessary to prepare an [etcd](https://etcd.io/) key-value store.

If you are looking for an Ansible role for etcd, you can check out [ansible-role-etcd](https://github.com/mother-of-all-self-hosting/ansible-role-etcd) maintained by the [Mother-of-All-Self-Hosting (MASH)](https://github.com/mother-of-all-self-hosting) team.

## Adjusting the playbook configuration

To enable APISIX with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# apisix                                                               #
#                                                                      #
########################################################################

apisix_enabled: true

########################################################################
#                                                                      #
# /apisix                                                              #
#                                                                      #
########################################################################
```

### Set the hostname

To enable APISIX you need to set the hostname at which the API would be exposed as well. To do so, add the following configuration to your `vars.yml` file. Make sure to replace `example.com` with your own value.

```yaml
apisix_hostname: "example.com"
```

After adjusting the hostname, make sure to adjust your DNS records to point the domain to your server.

### Set a subpath for the API (optional)

It is possible to serve the API under a subpath by adding the following configuration to your `vars.yml` file:

```yaml
apisix_path_prefix: YOUR_SUBPATH_HERE
```

For example, setting this to `/api` will have the API served on `https://example.com/api`.

### Exposing Admin API (optional)

>[!WARNING]
> Take a look at the "Configuring the bundled dashboard (web UI)" section below before enabling it. This same listener also serves APISIX's built-in web UI, which has no authentication of its own. You might want to reach through an SSH tunnel instead of publishing them.

You can also expose the Admin API publicly by adding the following configuration to your `vars.yml` file:

```yaml
apisix_container_labels_admin_enabled: true
apisix_container_labels_admin_hostname: admin.api.example.com
apisix_container_labels_admin_path_prefix: /
```

When exposing the API, it is by default necessary to specify a valid key as below:

```yaml
apisix_config_deployment_admin_admin_key:
  - name: admin1
    key: secret-api-key-here
    role: admin
  - name: viewer1
    key: secret-api-key-here
    role: viewer
```

## Configuring the bundled dashboard (web UI)

Since APISIX 3.13, [APISIX Dashboard](https://github.com/apache/apisix-dashboard/releases/tag/notice) is shipped as a front-end inside the `apache/apisix` image this role installs, and APISIX's own nginx serves it — `/ui` redirects to `/ui/`, which serves the bundled assets.

> [!WARNING]
> The bundled UI is served by the Admin API's listener. It shares that listener's port (`apisix_config_deployment_admin_admin_listen_port`) and its `allow_admin` allowlist (`apisix_config_deployment_admin_allow_admin`).
>
> **Making the UI reachable therefore makes the Admin API reachable.** There is no way to publish one without the other short of routing individual paths yourself.
>
> What protects each of them is different, and worth being clear about:
>
> - the Admin API rejects requests without a valid key (`apisix_config_deployment_admin_admin_key`, enforced by `apisix_config_deployment_admin_admin_key_required`)
> - **the UI itself does not have authentication.** It asks the person using it for an Admin API key and talks to the Admin API from their browser
>
> So exposing this to the internet publishes an unauthenticated login-less admin console for your gateway. Anyone who finds it cannot change anything without a key, but they can see that you run APISIX and are handed the console to attack it with.

Put an authenticating reverse-proxy in front of it (with Traefik, a [basic-auth middleware](https://doc.traefik.io/traefik/reference/routing-configuration/http/middlewares/basicauth/) middleware via `apisix_container_labels_admin_middlewares`), restrict `allow_admin` to addresses you trust, or do not publish it at all.

Reaching it, from least to most exposed:

- **Through an SSH tunnel** — nothing is published to the network. Add `apisix_container_admin_http_bind_port: "127.0.0.1:9180"` to your `vars.yml`. After re-running the command for installation, run `ssh -L 9180:127.0.0.1:9180 example@example.com` from your own machine and open <http://127.0.0.1:9180/ui/>.

- **Through Traefik, with authentication** — enable the Admin API as in the example configuration above and put a basic-auth middleware in front of it. **Otherwise, you are publishing an admin console to the internet.**

  The role has no dedicated variable for this (unlike the metrics route), so you need to declare the middleware yourself and then reference it by name:

  ```yaml
  apisix_container_labels_additional_labels_custom:
    # Generate the entry with `htpasswd -nb USERNAME PASSWORD`
    - "traefik.http.middlewares.mash-apisix-gateway-admin-auth.basicauth.users=someone:$apr1$..."

  apisix_container_labels_admin_middlewares:
    - mash-apisix-gateway-admin-auth
  ```

  The UI is then at `https://admin.api.example.com/ui/`.

If you expose the Admin API but would rather not publish the console alongside it, add the following configuration to your `vars.yml` file:

```yaml
apisix_config_deployment_admin_enable_admin_ui: false
```

### Extending the configuration

There are some additional things you may wish to configure about the service.

Take a look at:

- [`defaults/main.yml`](../defaults/main.yml) for some variables that you can customize via your `vars.yml` file. You can override settings (even those that don't have dedicated playbook variables) using the `apisix_environment_variables_additional_variables` variable

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After running the command for installation, APISIX becomes available.

You can send API requests to your API gateway (as specified in `apisix_hostname` and `apisix_path_prefix`) as below:

```sh
curl https://example.com/api
```

Since no routes are configured by default, you'll receive 404 requests. To configure routes, either use the Admin API or the bundled web UI.

If you have enabled the Admin API, it will be available at `https://admin.api.example.com/` (as specified in `apisix_container_labels_admin_hostname` and `apisix_container_labels_admin_path_prefix`).

### Using Admin API

You will be able to manage the APISIX configuration (managing routes, upstreams, etc.) by sending API requests to the Admin API URL as below:

```sh
curl -H 'X-API-KEY: YOUR_SECRET_API_KEY_HERE' https://admin.api.example.com/apisix/admin/routes
```

`/ui/` then returns a 404 while the Admin API keeps working on the same port.

## Troubleshooting

### Migrating from `apisix_gateway_*`

This role used to be called `apisix-gateway` and all of its variables were prefixed `apisix_gateway_`. Both lost the `_gateway`: the role is `apisix` and its variables are `apisix_*`. Rename them in your configuration — the role fails with the full list of what to change if it finds any of the old names, whether it is enabled or not.

Only the names changed. `apisix_identifier` still defaults to `apisix-gateway`, so the systemd service, the container, the container network and the base path are untouched by the rename, and nothing gets reinstalled or moved.

### Check the service's logs

You can find the logs in [systemd-journald](https://www.freedesktop.org/software/systemd/man/systemd-journald.service.html) by logging in to the server with SSH and running `journalctl -fu apisix` (or how you/your playbook named the service, e.g. `mash-apisix`).
