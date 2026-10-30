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
  - **Debian** (12/Bookworm or newer)
  - **Ubuntu** (22.04 or newer)
- `root` access on the computer which Molecule runs against
- [Ansible](https://www.ansible.com/) program
- [Python](https://www.python.org/)
  - Most distributions install Python by default; others need it installed (something like `apt-get install python3`)
- [Docker](https://www.docker.com)
  - Access to Docker UNIX socket (`/var/run/docker.sock`) is required by default

## Installation

To set up the environment for using Molecule, run the command below on the terminal:

```bash
python3 -m venv ./molecule/venv
source ./molecule/venv/bin/activate
pip3 install -r ./molecule/requirements.txt
```

## Scenarios

Currently there is 1 testing scenario available.

### `default`

Tests a standard kukaroll installation, and then deploys with it.

A bare repository on the instance stands in for the forge, and a clone of it is the project's checkout.
The scenario checks that:

- the receiver runs the image version that `defaults/main.yml` pins;
- starting the receiver and the deploys target deploys the checkout's current commit (the catch-up);
- a webhook signed with a wrong secret is refused (401), and so is one for an unknown project (404);
- a signed push webhook moves the checkout to the pushed commit, and the project's deploy command runs there;
- a webhook received while `kukaroll-deploys.target` is stopped deploys nothing until the target starts again, and then deploys the pushed commit;
- the receiver does not restart over 45 seconds.

The executor runs privileged in the host's PID namespace, so the instance itself has to be privileged (it already is, to run Docker).

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
