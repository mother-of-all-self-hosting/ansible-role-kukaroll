<!--
SPDX-FileCopyrightText: 2026 Slavi Pantaleev

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# Setting up kukaroll

This is an [Ansible](https://www.ansible.com/) role which installs [kukaroll](https://github.com/devture/kukaroll) to run as [Docker](https://www.docker.com/) containers wrapped in systemd units.

kukaroll deploys git checkouts on the host when their forge (Forgejo, Gitea or GitHub) reports a push: it moves the checkout to the tip of its branch and runs the project's own deploy command.
It knows nothing about how a project is deployed; that lives in the project's repository.

Two parts run on the host:

- the **receiver**, an unprivileged container that receives the webhooks and only requests deploys;
- the **executor**, a privileged container which systemd starts per project when a deploy is requested.
  It runs git and the project's commands on the host, as the users you configure.

[kukaroll's documentation](https://github.com/devture/kukaroll/blob/main/docs/README.md) describes how a deploy runs, how to write the project's commands, and what to trust.

## Prerequisites

- Each project's checkout exists on the host, with a remote that its git user can fetch from (e.g. with an SSH deploy key).
  The role does not clone it.
- The project's git user (`git_user`) and hooks user (`hooks_user`) exist in the host's `/etc/passwd`.
- Every directory above `kukaroll_base_path` is owned by root and writable only by root, or the role refuses to install: whoever can write there could swap in their own executor configuration.
  The MASH playbook's base directory (`/mash`) is, since 2026-10-30.
- A reverse proxy routes the webhooks to the receiver.
  The role can set up [Traefik](https://traefik.io/) for it (see below).

>[!WARNING]
> Whoever can push to a deployed branch, or write to the checkout (typically the git user's account), can run commands on the host as the project's `hooks_user`.
> With `hooks_user: root`, treat the git user's account as root.
> See [Trust](https://github.com/devture/kukaroll/blob/main/docs/architecture.md#trust) in kukaroll's architecture document.

## Adjusting the playbook configuration

To enable kukaroll with this role, add the following configuration to your `vars.yml` file.

**Note**: the path should be something like `inventory/host_vars/mash.example.com/vars.yml` if you use the [MASH Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

```yaml
########################################################################
#                                                                      #
# kukaroll                                                             #
#                                                                      #
########################################################################

kukaroll_enabled: true

kukaroll_hostname: example.com
kukaroll_path_prefix: /kukaroll

kukaroll_projects:
  myapp:
    checkout: /srv/myapp
    remote: origin
    branch: main
    git_user: myapp
    hooks_user: root
    lock_file: /srv/myapp/var/deploy.lock
    pre_flight: null
    deploy: [just, deploy]
    healthchecks_ping_url: null
    webhook_provider: forgejo
    webhook_secret: A_LONG_RANDOM_STRING
    webhook_authorization_header: "Bearer ANOTHER_LONG_RANDOM_STRING"

########################################################################
#                                                                      #
# /kukaroll                                                            #
#                                                                      #
########################################################################
```

With this, the webhook URL of `myapp` is `https://example.com/kukaroll/hooks/myapp`.
After adjusting the hostname, make sure that its DNS record points to your server.

### Projects

`kukaroll_projects` is keyed by project name (lowercase letters, digits, dashes and underscores, starting with a letter or digit, at most 64 characters), which is also the last part of its webhook URL.
Every key is required; use `null` for a feature you don't use.

| Key | Meaning |
| --- | --- |
| `checkout` | The git checkout to deploy |
| `remote`, `branch` | What to fetch and deploy (a branch name like `main`) |
| `git_user` | The user that runs git; usually the checkout's owner |
| `hooks_user` | The user that runs `pre_flight` and `deploy`, e.g. `root` to run `docker compose` |
| `lock_file` | Locked for the whole run; other tools can lock it to keep deploys out. Keep it outside what `git checkout` writes: in an ignored directory of the checkout (like `var/` above), or outside the checkout. Its directory must exist. See [The lock](https://github.com/devture/kukaroll/blob/main/docs/hooks.md#the-lock) |
| `pre_flight` | A command run before the checkout moves; a failure stops the deploy. Or `null` |
| `deploy` | A command run after the checkout has moved. Or `null` (moving the checkout is the whole deploy) |
| `healthchecks_ping_url` | A [Healthchecks](https://healthchecks.io/) ping URL to report each run to. Or `null`. With the daily catch-up, a period of 1 day works, with a grace time over 1 hour (the timer spreads its runs over an hour). See [Healthchecks](https://github.com/devture/kukaroll/blob/main/docs/operations.md#healthchecks) in kukaroll's documentation |
| `webhook_provider` | `forgejo`, `gitea` or `github` |
| `webhook_secret` | The webhook's secret, which the forge signs each delivery with |
| `webhook_authorization_header` | With Traefik, only requests carrying exactly this `Authorization` header reach the receiver. Or `null`. GitHub cannot send one. The value is in the receiver's container labels, so anyone who can see them (`docker inspect`, Traefik's dashboard or API) can read it |

Commands are lists of arguments, run without a shell.
Keep the deploy logic in the project's repository behind a stable command (like `just deploy` above), so that it changes with the code.
[Writing hooks](https://github.com/devture/kukaroll/blob/main/docs/hooks.md) in kukaroll's documentation describes what the commands get and what happens when they fail.

Generate the 2 secrets with e.g. `pwgen -s 64 1`.
The role checks every project's settings with kukaroll's own `check-config` before installing them.

### Setting up the webhook on the forge

In the repository's webhook settings on the forge, add a webhook with:

- the URL `https://example.com/kukaroll/hooks/myapp`;
- the secret from `webhook_secret`;
- on Forgejo and Gitea, the authorization header from `webhook_authorization_header`.

[Setting up the forge](https://github.com/devture/kukaroll/blob/main/docs/webhooks.md#setting-up-the-forge) in kukaroll's documentation lists the other fields for each forge.
The receiver ignores pushes to other branches.

### Reverse proxy

By default, the role adds Traefik labels to the receiver's container.
Each project gets its own route, matching only its webhook URL (and its `Authorization` header, when set).
Requests that match no route never reach the receiver, and bodies over 1 MiB are refused (`kukaroll_container_labels_traefik_max_request_body_bytes`).

To use another reverse proxy, disable the labels and publish the receiver's port on the host:

```yaml
kukaroll_container_labels_traefik_enabled: false
kukaroll_receiver_container_http_host_bind_port: "127.0.0.1:8080"
```

Then route `https://example.com/kukaroll/hooks/*` to `http://127.0.0.1:8080/hooks/*`.
[Behind a reverse proxy](https://github.com/devture/kukaroll/blob/main/docs/webhooks.md#behind-a-reverse-proxy) in kukaroll's documentation describes both checks, with a Traefik example of both and a Caddy example of the body limit.

### Pausing deploys while services restart (optional)

All deploy triggers belong to the `kukaroll-deploys.target` systemd unit: stopping it pauses deploys (running ones finish), starting it resumes them, and pushes that arrived meanwhile get deployed.

When the playbook starts, restarts or stops services, the role pauses deploys first (`kukaroll_pause_enabled`), so that no deploy works on a project while its services restart.
Whatever starts the services afterwards must start `kukaroll-deploys.target` again.
The MASH playbook does that; with another playbook, start it after the services, e.g. by adding it to `devture_systemd_service_manager_services_list`.

To also order deploys after the projects' own systemd units (e.g. at boot), list them:

```yaml
kukaroll_deploys_systemd_after_services_list_custom:
  - myapp.service
```

### Catch-up (optional)

Whenever deploys resume (after the playbook runs, at boot) and once a day, every project gets a deploy request.
It makes up for a webhook that got lost, e.g. while the reverse proxy restarted.
A project whose branch tip is already deployed gets a short run that changes nothing, but which still reports to its Healthchecks check.

To change how often it runs, or to disable the daily run:

```yaml
# A systemd calendar expression (see `man systemd.time`)
kukaroll_catch_up_timer_on_calendar: hourly

# Or: only when deploys resume
kukaroll_catch_up_timer_enabled: false
```

### Building the container image (optional)

The role pulls kukaroll's image from `ghcr.io/devture/kukaroll`.
To build it on the host from [kukaroll's repository](https://github.com/devture/kukaroll) instead, add the following configuration to your `vars.yml` file:

```yaml
kukaroll_container_image_self_build: true
```

To use an image you built or loaded by hand, set `kukaroll_container_image` to it.
Its name must start with `localhost/` (e.g. `just image tag=localhost/kukaroll:dev` in kukaroll's repository), or the role tries to pull it; the role only checks that it is on the host.

## Installing

After configuring the playbook, run the installation command of your playbook as below:

```sh
ansible-playbook -i inventory/hosts setup.yml --tags=setup-all,start
```

If you use the MASH playbook, the shortcut commands with the [`just` program](https://github.com/mother-of-all-self-hosting/mash-playbook/blob/main/docs/just.md) are also available: `just install-all` or `just setup-all`

## Usage

After installation, push to a deployed branch, or deploy right away from the host:

```sh
/kukaroll/bin/trigger myapp
/kukaroll/bin/trigger --all
```

The helpers are in `kukaroll_base_path` (`/mash/kukaroll` in the MASH playbook).
Unit names start with `kukaroll` (`kukaroll_identifier`, or however your playbook named it, e.g. `mash-kukaroll`).

To pause and resume deploys by hand:

```sh
systemctl stop kukaroll-deploys.target
systemctl start kukaroll-deploys.target
```

## Troubleshooting

Each project's deploys log to their own unit.
To follow them, log in to the server with SSH and run `journalctl -fu kukaroll-execute@myapp`.
The receiver logs a line per delivery to a configured project: `journalctl -fu kukaroll-receiver`.
That line and the forge's delivery history show a status for each delivery; [Responses](https://github.com/devture/kukaroll/blob/main/docs/webhooks.md#responses) in kukaroll's documentation lists what they mean.

A failed deploy is not recorded as deployed, so the next push, the catch-up or `trigger` retries it.
Failures before the executor runs (a missing image, a configuration that does not load) show only in that unit's journal and failed state, not in Healthchecks.

`trigger` alone does nothing when the branch tip is already deployed.
To run the commands again for it, see [Redeploying the same commit](https://github.com/devture/kukaroll/blob/main/docs/operations.md#redeploying-the-same-commit) in kukaroll's documentation.
