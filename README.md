<!--
SPDX-FileCopyrightText: 2026 Slavi Pantaleev

SPDX-License-Identifier: AGPL-3.0-or-later
-->

# kukaroll Ansible role

[![REUSE status](https://api.reuse.software/badge/github.com/mother-of-all-self-hosting/ansible-role-kukaroll)](https://api.reuse.software/info/github.com/mother-of-all-self-hosting/ansible-role-kukaroll)

This is an [Ansible](https://www.ansible.com/) role which installs [kukaroll](https://github.com/devture/kukaroll) to run as [Docker](https://www.docker.com/) containers wrapped in systemd units.
kukaroll deploys git checkouts on the host when their forge reports a push.

This role *implicitly* depends on:

- [`com.devture.ansible.role.playbook_help`](https://github.com/devture/com.devture.ansible.role.playbook_help)
- [`com.devture.ansible.role.systemd_docker_base`](https://github.com/devture/com.devture.ansible.role.systemd_docker_base)

Check [`defaults/main.yml`](defaults/main.yml) for the full list of supported options.
See [`docs/configuring-kukaroll.md`](docs/configuring-kukaroll.md) for setting up the service with this role.

💡 For an Ansible playbook which integrates this role and makes it easier to use, see the [Mother-of-All-Self-Hosting Ansible playbook](https://github.com/mother-of-all-self-hosting/mash-playbook).

## Development

### pre-commit

You can optionally install a Git pre-commit hook (via [mise](https://mise.jdx.dev/) + [prek](https://prek.j178.dev/)) that runs formatting and linting checks before each commit.
See [`.pre-commit-config.yaml`](./.pre-commit-config.yaml) for which hooks are to be executed.

To install the hook, run the [`just`](https://github.com/casey/just) command below:

```sh
just prek-install-git-pre-commit-hook
```

### Molecule

This role supports [Molecule](https://docs.ansible.com/projects/molecule/), an Ansible testing framework designed for developing and testing Ansible collections, playbooks, and roles.

See [`molecule/README.md`](./molecule/README.md) for how to use it.

### Releases

Tags are created on the GitHub workflow by [`.github/workflows/autotag.yml`](.github/workflows/autotag.yml), which asks [`bin/compute-next-tag.sh`](bin/compute-next-tag.sh) what the commit on `main` should be released as.
Tags look like `v<kukaroll version>-<release>` (e.g. `v2026.10.30.0-0`): the version comes from `kukaroll_version` in [`defaults/main.yml`](defaults/main.yml), and the release number counts the role's releases of that version.
Only commits that change `defaults/`, `meta/`, `tasks/` or `templates/` are released, right away, without waiting for a new kukaroll version.

[`bin/test-compute-next-tag.sh`](bin/test-compute-next-tag.sh) exercises that script against throwaway repositories, and runs as a prek hook.
