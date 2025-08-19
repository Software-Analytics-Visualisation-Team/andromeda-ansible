# Andromeda Ansible Playbook

This repository contains the Ansible playbooks and configuration files for deploying and managing the Andromeda tools on an existing Galaxy instance.

## Moonshot: add and update the Moonshot tools

---

## Table of Contents

* [Quickstart](#quickstart)
* [Which playbook should I run?](#which-playbook-should-i-run)
* [Key variables in `moonshot.yml` (what each path points to)](#key-variables-in-moonshotyml-what-each-path-points-to)
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

> This repository assumes Galaxy and nginx are already installed and running. These playbooks **wire the Andromeda tools into that existing instance** and set up gx-it-proxy and Docker for ClassViz.

---

## Which playbook should I run?

* **`moonshot.yml`** → first install, or whenever you need **gx-it-proxy**, **datatypes**, and the **nginx wiring** (re)applied.
* **`update_tools.yml`** → refresh the **tool code** and **rebuild ClassViz** **without** touching datatypes, gx-it-proxy, or nginx.

---

## Key variables in `moonshot.yml` (what each path points to)

A newcomer needs to know what each path really is on their Galaxy. Adjust these to your instance; defaults target a standard **`/srv/galaxy/server`** layout.

* **`galaxy_root`** → your Galaxy **`server/`** root.
* **`tool_conf`** → your **live toolbox XML** (not the `.sample`; point to the XML your instance actually loads).
* **`datatypes_conf_dir`** → the directory containing the **datatypes XML** that Galaxy loads.
* **`datatypes_dir`** → **`lib/galaxy/datatypes`** inside your Galaxy checkout.
* **`nginx_galaxy_path`** → the **vhost file** your nginx actually uses for Galaxy.
* **`interactivetools_map`** → a **writable** var file path for Galaxy’s interactive tools map (Galaxy must be able to read/write it).

> Tip: keep your paths consistent with how Galaxy is started (uWSGI/systemd/virtualenv) so the same files are visible to both Galaxy and Ansible.

---

## Designite license (vaulted)

Store your Designite license key in a **vaulted** vars file and reference it with `vault_designite_license_key`.

```yaml
# group_vars/secret.yml  (encrypt with ansible-vault)
vault_designite_license_key: "XXXX-XXXX-XXXX-XXXX"
```

Keep the file encrypted with `ansible-vault`. Do **not** commit real secrets.

---

## Prerequisites

* Ubuntu/Debian host with sudo access
* Python **3.12**
* Ansible **2.14+** (recommended)
* Existing **Galaxy** + **nginx** installation
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

Before running, review and adapt variables in **`moonshot.yml`** to match your paths (see [Key variables](#key-variables-in-moonshotyml-what-each-path-points-to)).

### First install / full wiring

```bash
ansible-playbook moonshot.yml --ask-become --ask-vault-pass
```

### Update only the tool code and rebuild ClassViz

```bash
ansible-playbook update_tools.yml --ask-become --ask-vault-pass
```

### (Optional) Install/update **only** the Designite tool

```bash
ansible-playbook moonshot.yml --ask-become --ask-vault-pass --tags designite
# or
ansible-playbook update_tools.yml --ask-become --ask-vault-pass --tags designite
```

---

## Prerequisite configuration for the ClassViz tool

> ⚠️ These settings are required for ClassViz to work (Docker, gx-it-proxy, and Galaxy interactive tools must be enabled and reachable). Configure these according to your environment before first run.

### Galaxy Configuration (`galaxy.yml`)

Your Galaxy configuration now requires specific settings under the `gravity` section of your `config/galaxy.yml`. Add or verify the following entries:

```yaml
gravity:
  gx_it_proxy:
    enable: true
    port: 4002
```

And you will also need these settings under the 'galaxy' section : 
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

Galaxy now prefers a YAML-based job configuration. In your `config/job_conf.yml` (create or edit this file), define an environment for interactive tools and assign it to the ClassViz tool:

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
Don't forget you will need an environment for your non-interactive tools like :

```yaml
execution:
  default: default
  environments:
    # Non interactive tools
    default:
      runner: local
      docker_enabled: false 
      interactive: false      
```
If you still work with XML, a reference XML-based job configuration (`job_conf.xml`) is still available under `files/classviz` if you need them.

Full examples are available under :
- [files/classviz/galaxy.yml](files/classviz/galaxy.yml)  
- [files/classviz/job_conf.yml](files/classviz/job_conf.yml)
- [files/classviz/job_conf.xml](files/classviz/job_conf.xml)

---

## Managing dependencies in Galaxy

After the playbook completes, open Galaxy as an **admin** and go to **Admin → Manage Dependencies**. Select and install the dependencies for all Andromeda tools. On the first run you may need to re-run the code-to-SPIF role once (known quirk).

---

## What the playbooks do

At a high level:

* Clone/update the Andromeda tool repositories in the expected tools directory.
* Ensure required Python packages are declared so Galaxy installs them automatically.
* Build the **Docker image for ClassViz**.
* Update Galaxy’s toolbox to expose the **Moonshot** section and tools.
* Wire up **gx-it-proxy** and adjust **nginx** configuration as needed.
* Place/update the Galaxy **interactive tools map** file.

`update_tools.yml` skips datatypes, gx-it-proxy and nginx changes; it focuses on pulling latest tool code and rebuilding ClassViz.

---

## Troubleshooting

* **Dependencies fail to install in Galaxy** → retry from **Admin → Manage Dependencies**; verify the conda env is executable and internet access is available.
* **nginx changes not applied** → confirm `nginx_galaxy_path` points to the vhost file actually loaded by your nginx service, then reload nginx.
* **Interactive tools don’t start** → check gx-it-proxy is running and that the `interactivetools_map` path is writable by Galaxy.

---

## Recap checklist

* [ ] Prerequisites installed (Python 3.12, Ansible, Docker)
* [ ] Repo cloned + galaxy requirements installed
* [ ] Variables reviewed (`galaxy_root`, `tool_conf`, `datatypes_*`, `nginx_galaxy_path`, `interactivetools_map`)
* [ ] Designite license stored in **vaulted** `group_vars/secret.yml`
* [ ] `moonshot.yml` run for first install (wiring)
* [ ] `update_tools.yml` used for subsequent updates
* [ ] Dependencies installed from Galaxy admin UI



```
