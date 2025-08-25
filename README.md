# Andromeda Ansible Playbook

This repository contains the Ansible playbooks and configuration files for deploying a **Galaxy** instance and wiring the **Andromeda / Moonshot** tools onto it.

---

## Table of Contents

* [Install Galaxy: `galaxy.yml` Ansible playbook](#install-galaxy-galaxyyml-ansible-playbook)

  * [How it works](#how-it-works)
  * [Prerequisites](#prerequisites-for-galaxyyml)
  * [Inventory layout](#inventory-layout)
  * [Key variables](#key-variables-for-galaxyyml)
  * [Secrets](#secrets)
  * [Launch](#launch)
  * [Verify](#verify)
* [Moonshot: add and update the Moonshot tools](#moonshot-add-and-update-the-moonshot-tools)

  * [Quickstart](#quickstart)
  * [Which playbook should I run?](#which-playbook-should-i-run)
  * [Key variables in `moonshot.yml` (paths)](#key-variables-in-moonshotyml-paths)
  * [Designite license (vaulted)](#designite-license-vaulted)
  * [Prerequisites](#prerequisites)
  * [Installation](#installation)
  * [Running the playbooks](#running-the-playbooks)
  * [Prerequisite configuration for the ClassViz tool](#prerequisite-configuration-for-the-classviz-tool)
  * [Managing dependencies in Galaxy](#managing-dependencies-in-galaxy)
  * [What the playbooks do](#what-the-playbooks-do)
  * [Troubleshooting](#troubleshooting)
  * [Recap checklist](#recap-checklist)

---

## Install Galaxy: `galaxy.yml` Ansible playbook

Use this playbook to install a fresh Galaxy with PostgreSQL, Gunicorn/Celery (via **Gravity**), Miniconda for tool dependencies, and nginx as the reverse proxy.

### How it works

`galaxy.yml` is split in two plays:

1. **`hosts: dbservers`** — installs and configures PostgreSQL using the roles:

   * `galaxyproject.postgresql`
   * `galaxyproject.postgresql_objects` (creates the Galaxy DB and user)
2. **`hosts: galaxyservers`** — installs Galaxy and web stack using the roles:

   * `galaxyproject.galaxy` (Galaxy + Gravity services)
   * `galaxyproject.miniconda` (tool dependency resolver)
   * `galaxyproject.nginx` (reverse proxy; SSL via `usegalaxy_eu.certbot` if desired)

### Prerequisites for `galaxy.yml`

* Ubuntu/Debian hosts reachable over SSH with sudo
* Ansible 2.14+ on your control machine
* Open ports 80/443 on the Galaxy host (and a DNS name if you want HTTPS)
* On the Galaxy host, the playbook installs basic packages like `acl`, `bzip2`, `git`, `make`, `tar`, `python3-venv`, `python3-setuptools`, `python3-lxml`

### Inventory layout

Minimal example (single DB and single Galaxy node):

```ini
[dbservers]
db1.example.org

[galaxyservers]
galaxy1.example.org
```

You can also target one machine for both by putting the same host under both groups.

### Key variables for `galaxy.yml`

Variables live in `group_vars/`. Adjust them to your environment.

**`group_vars/all.yml`**

```yaml
galaxy_user_name: galaxy_usr
galaxy_db_name: galaxy_db
```

**`group_vars/dbservers.yml`**

Creates the Galaxy DB and user on PostgreSQL:

```yaml
postgresql_objects_users:
  - name: "{{ galaxy_user_name }}"
postgresql_objects_databases:
  - name: "{{ galaxy_db_name }}"
    owner: "{{ galaxy_user_name }}"
```

**`group_vars/galaxyservers.yml`** (high‑level highlights)

```yaml
# Where Galaxy lives and which version to deploy
galaxy_root: /srv/galaxy
galaxy_user: { name: "{{ galaxy_user_name }}", shell: /bin/bash }
galaxy_commit_id: release_24.0

# Miniconda for tool dependencies
miniconda_prefix: "{{ galaxy_tool_dependency_dir }}/_conda"
miniconda_version: 23.9
miniconda_channels: ["conda-forge", "defaults"]

# Job runners and execution (Local runner by default)
galaxy_job_config:
  runners:
    local_runner:
      load: galaxy.jobs.runners.local:LocalJobRunner
      workers: 4
  execution:
    default: local_env
    environments:
      local_env:
        runner: local_runner
        tmp_dir: true

# Main Galaxy config (trimmed)
galaxy_config:
  galaxy:
    admin_users: you@example.org
    database_connection: "postgresql:///{{ galaxy_db_name }}?host=/var/run/postgresql"
    file_path: /data/datasets
    job_working_directory: /data/jobs
    object_store_store_by: uuid
    id_secret: "{{ vault_id_secret }}"  # set in an ansible-vaulted file
    job_config: "{{ galaxy_job_config }}"
    outputs_to_working_directory: true
  gravity:
    process_manager: systemd
    galaxy_root: "{{ galaxy_root }}/server"
    galaxy_user: "{{ galaxy_user_name }}"
    virtualenv: "{{ galaxy_venv_dir }}"
    gunicorn:
      bind: "unix:{{ galaxy_mutable_config_dir }}/gunicorn.sock"
      workers: 2
    celery:
      concurrency: 2

# Nginx
nginx_servers: [redirect-ssl]
nginx_ssl_servers: [galaxy]
```

> Note: the default DB connection uses the local UNIX socket (`host=/var/run/postgresql`). If you use a remote DB or TCP, set `database_connection` accordingly.

### Secrets

Create `group_vars/secret.yml` and encrypt it with **ansible‑vault**. At minimum it should contain a strong `vault_id_secret` for Galaxy:

```yaml
vault_id_secret: "<generate with: openssl rand -hex 32>"
```

Encrypt the file:

```bash
ansible-vault create group_vars/secret.yml
```

Use `--ask-vault-pass` (or a vault ID) when running the playbook.

### Launch

Install dependencies and run the playbook:

```bash
# fetch role dependencies
ansible-galaxy install -r requirements.yml

# install Galaxy + DB + nginx
ansible-playbook -i inventory galaxy.yml \
  --ask-become --ask-vault-pass
```

You can limit to one tier during testing:

```bash
ansible-playbook -i inventory galaxy.yml --limit galaxyservers --ask-become --ask-vault-pass
```

### Verify

Check services on the Galaxy host:

```bash
sudo systemctl status galaxy
sudo systemctl status nginx
```

Then browse to your host (HTTP or HTTPS if you enabled certificates). Log in with an admin account listed in `galaxy_config.galaxy.admin_users`.

---
&nbsp;
&nbsp;
&nbsp;
&nbsp;

## Moonshot: add and update the Moonshot tools

These playbooks wire the Andromeda tools into an **existing Galaxy** (installed by you or via the section above) and set up gx‑it‑proxy and Docker for ClassViz.

---

## Quickstart

```bash
# prerequisites (Ubuntu/Debian)
sudo apt update && sudo apt install -y python3.12-dev build-essential python3-lib2to3

# clone + deps
git clone https://github.com/Software-Analytics-Visualisation-Team/andromeda-ansible.git
cd andromeda-ansible
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install community.general community.docker

# configure variables (see sections below), then:
ansible-playbook moonshot.yml --ask-become
```

If you need to **install Galaxy first**, run `galaxy.yml` as shown above, then come back to `moonshot.yml`.

---

## Which playbook should I run?

* **`galaxy.yml`** → install a new Galaxy + PostgreSQL + nginx (first‑time setup).
* **`moonshot.yml`** → wire and install Andromeda/Moonshot (gx‑it‑proxy, datatypes, nginx tweaks) on that Galaxy.
* **`update_tools.yml`** → refresh tool code and rebuild ClassViz **without** touching datatypes, gx‑it‑proxy, or nginx.

---

## Key variables in `moonshot.yml` (paths)

Adjust these to your instance; defaults target a standard `/srv/galaxy/server` layout.

* `galaxy_root` → your Galaxy `server/` root
* `tool_conf` → the **live** toolbox XML
* `datatypes_conf_dir` → directory containing the **datatypes XML** Galaxy loads
* `datatypes_dir` → `lib/galaxy/datatypes` inside your Galaxy checkout
* `nginx_galaxy_path` → the vhost file your nginx actually uses for Galaxy
* `interactivetools_map` → a **writable** path for Galaxy’s interactive tools map

Tip: keep paths consistent with how Galaxy is started (uWSGI/systemd/virtualenv) so the same files are visible to both Galaxy and Ansible.

---

## Designite license (vaulted)

Store your Designite license key in a vaulted vars file and reference it with `vault_designite_license_key`.

```yaml
# group_vars/secret.yml  (encrypt with ansible-vault)
vault_designite_license_key: "XXXX-XXXX-XXXX-XXXX"
```

Do not commit real secrets.

---

## Prerequisites

* Ubuntu/Debian host with sudo access
* Python 3.12
* Ansible 2.14+
* Existing Galaxy + nginx installation (use `galaxy.yml` above if needed)
* Docker available for building/running ClassViz images

Prepare system packages:

```bash
sudo apt update && sudo apt install -y python3.12-dev build-essential python3-lib2to3
```

---

## Installation

```bash
git clone https://github.com/Software-Analytics-Visualisation-Team/andromeda-ansible.git
cd andromeda-ansible

# Install role/collection dependencies
ansible-galaxy install -r requirements.yml
ansible-galaxy collection install community.general community.docker
```

---

## Running the playbooks

Before running, review and adapt variables in `moonshot.yml` to match your paths (see [Key variables](#key-variables-in-moonshotyml-paths)).

### First install / full wiring

```bash
ansible-playbook moonshot.yml --ask-become --ask-vault-pass
```

### Update only the tool code and rebuild ClassViz

```bash
ansible-playbook update_tools.yml --ask-become --ask-vault-pass
```

### (Optional) Install/update only the Designite tool

```bash
ansible-playbook moonshot.yml --ask-become --ask-vault-pass --tags designite
# or
ansible-playbook update_tools.yml --ask-become --ask-vault-pass --tags designite
```

---

## Prerequisite configuration for the ClassViz tool

⚠️ These settings are required for ClassViz to work (Docker, gx‑it‑proxy, and Galaxy interactive tools must be enabled and reachable). Configure these according to your environment before first run.

### Galaxy Configuration (`galaxy.yml`)

Your Galaxy configuration now requires specific settings under the `gravity` section of your `config/galaxy.yml`. Add or verify the following entries:

```yaml
gravity:
  gx_it_proxy:
    enable: true
    port: 4002
```

And you will also need these settings under the `galaxy` section:

```yaml
galaxy:
  job_config_file: config/job_conf.yml #If you already have a job_config: inline in your galaxy.yml, it won't load job_conf.yml. Comment it, or adjust your job_config.
  outputs_to_working_directory: true

  interactivetools_enable: true
  interactivetools_map: database/interactivetools_map.sqlite #Adapt this path according to your galaxy_root, example :
  # interactivetools_proxy_bin: /srv/galaxy/var/interactivetools_map.sqlite #Make sure this file exists
  # interactivetools_proxy_host: localhost:4002 #Active locally
  # interactivetools_upstream_proxy: false #Active locally

  # galaxy_infrastructure_url: http://localhost:8080 #Active locally
```

### Job Configuration (`job_conf.yml`)

Galaxy now prefers a YAML‑based job configuration. In your `config/job_conf.yml` define an environment for interactive tools and assign it to the ClassViz tool:

```yaml
environments:
  docker_env:
    runner: local
    docker_enabled: true
    interactive: true
    docker_set_user: root
tools:
  - id: "classviz"
    environment: docker_env
```

Also define a default (non‑interactive) environment, for example:

```yaml
execution:
  default: default
  environments:
    default:
      runner: local
      docker_enabled: false
      interactive: false
```

If you still rely on XML, a reference `job_conf.xml` is available under `files/classviz`.

Full examples:

* `files/classviz/galaxy.yml`
* `files/classviz/job_conf.yml`
* `files/classviz/job_conf.xml`

---

## Managing dependencies in Galaxy

After the playbook completes, open Galaxy as an admin and go to **Admin → Manage Dependencies**. Select and install the dependencies for all Andromeda tools. On the first run you may need to re‑run the code‑to‑SPIF role once (known quirk).

---

## What the playbooks do

At a high level:

* Clone/update the Andromeda tool repositories in the expected tools directory
* Ensure required Python packages are declared so Galaxy installs them automatically
* Build the Docker image for ClassViz
* Update Galaxy’s toolbox to expose the Moonshot section and tools
* Wire up gx‑it‑proxy and adjust nginx configuration as needed
* Place/update the Galaxy interactive tools map file

`update_tools.yml` skips datatypes, gx‑it‑proxy and nginx changes; it focuses on pulling latest tool code and rebuilding ClassViz.

---

## Troubleshooting

* **Dependencies fail to install in Galaxy** → retry from **Admin → Manage Dependencies**; verify the conda env is executable and internet access is available
* **nginx changes not applied** → confirm `nginx_galaxy_path` points to the vhost file actually loaded by your nginx service, then reload nginx
* **Interactive tools don’t start** → check gx‑it‑proxy is running and that the `interactivetools_map` path is writable by Galaxy

---

## Recap checklist

* [ ] If needed, installed Galaxy with `galaxy.yml`
* [ ] Prerequisites installed (Python 3.12, Ansible, Docker)
* [ ] Repo cloned + role requirements installed
* [ ] Variables reviewed (`galaxy_root`, `tool_conf`, `datatypes_*`, `nginx_galaxy_path`, `interactivetools_map`)
* [ ] Secrets stored in vaulted `group_vars/secret.yml` (`vault_id_secret`, licenses)
* [ ] `moonshot.yml` run for first Andromeda wiring
* [ ] `update_tools.yml` used for subsequent updates
* [ ] Dependencies installed from Galaxy admin UI
