# Andromeda Ansible Playbook

This repository contains the Ansible playbook and configuration files for deploying and managing the Andromeda platform.

# Moonshot.yml : Adding the Moonshot Tools


## Table of Contents

* [Prerequisites](#prerequisites)
* [Installation](#installation)
* [Running the Playbook](#running-the-playbook)
* [Prerequisite configuration for the ClassViz Tool](#prerequisite-configuration-for-the-classviz-tool)
* [Managing Dependencies](#managing-dependencies)
* [What Does the Playbook Do?](#what-does-the-playbook-do)
* [Recapitulative Check List](#recapitulative-check-list)

## Prerequisites

Ensure you have the following installed on your system:

* **Operating System**: Ubuntu (or Debian-based)
* **Python 3.12**
* **Ansible** (tested with Ansible 2.14+)
* **An operational Galaxy**

### System Packages

```bash
sudo apt update
sudo apt install python3.12-dev build-essential python3-lib2to3
```

### Install Ansible

```bash
sudo apt install ansible
# or via pip:
# pip3 install ansible
```

## Installation

Clone this repository:

```bash
git clone https://github.com/Software-Analytics-Visualisation-Team/andromeda-ansible.git
cd andromeda-ansible
```

## Running the Playbook

First, adapt in `moonshot.yml` the **vars** according to your configuration and file organization.

Execute the main playbook `moonshot.yml`:

```bash
ansible-playbook moonshot.yml \
  --ask-vault-pass \
  --ask-become
```

When prompted for the vault password, enter:

```text
galaxy
```

## Prerequisite configuration for the ClassViz Tool
⚠️ Warning: These settings are required for the ClassViz Tool to work.

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
  interactivetools_proxy_host: localhost:4002 #Active locally
  interactivetools_upstream_proxy: false #Active locally

  galaxy_infrastructure_url: http://localhost:8080 #Active locally
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


## Managing Dependencies

After the playbook completes, open the Ansible Galaxy web UI as an administrator:

1. Navigate to **Admin** → **Manage Dependencies**.
2. Select and install the dependencies for all four tools.

> **Note:** On the first run, you may need to re-run the `code to spif` role due to a bug.

## What Does the Playbook Do?

When you run `moonshot.yml`, Ansible will:

1. **Clone each Andromeda tool** into `tools/moonshot`:

   * Andromeda-GitHub-Extraction-Tool
   * Andromeda-Code-Parsing-Tool
   * Andromeda-SPIFGrahpBuilder-Tool
   * Andromeda-ClassViz-Tool

2. **Inject required Python packages** into each tool’s definition so Galaxy installs them automatically.

3. **Build The Docker Image for ClassViz**.

4. **Update Galaxy’s toolbox** in `tool_conf.xml.sample`, adding a `Moonshot` section with each tool.

### Datatypes checks

5. **Adapt** `config/datatypes_conf.xml.sample` by registering the **SPIF** and **SVIF** datatypes.
6. **Patch Galaxy core**:

   * `lib/galaxy/datatypes/triples.py` → add the **SPIF** Class.
   * `lib/galaxy/datatypes/text.py` → add the **SVIF** Class.

  
### Additional system tasks performed automatically

7. **Install and configure** the Galaxy Interactive Tools proxy:
   * `npm install -g @galaxyproject/gx-it-proxy`
   * Create / edit the systemd unit **galaxy-gx-it-proxy.service** `sudo systemctl edit galaxy-gx-it-proxy.service` with
   
      - `--sessions /srv/galaxy/var/interactivetools_map.sqlite` (in the var folder because it's an editable folder)
      - also make sure you have the correct gx-it-proxy path : `/usr/local/bin/gx-it-proxy`
   * `systemctl daemon-reload && systemctl restart galaxy-gx-it-proxy.service`
  

8. **Update Nginx** (file `/etc/nginx/sites-enabled/galaxy`) to add a location block that proxies `/interactivetool/ep/` to `127.0.0.1:4002`.
     ```bash
        location ~* ^/interactivetool/ep/(.+)$ {
            proxy_pass         http://127.0.0.1:4002;
            proxy_http_version 1.1;
            proxy_set_header   Host $host;
            proxy_set_header   Upgrade $http_upgrade;
            proxy_set_header   Connection "upgrade";
            proxy_read_timeout 3600s;
        }     ```
 
These extra steps mean that if anything fails you can reproduce them manually using the commands above.

## Debug Notes

While running ClassViz in production you may hit these occasional hiccups:

1. **Intermittent “file not found” errors**  
   Some jobs still run on the old host runner instead of Docker, leading to missing `/opt/classviz/...` failures.  
   **Workaround**: Fully stop and then start your handler services so they reload the correct runner mapping. For example : 
     ```bash
     sudo systemctl stop galaxy-handler@0.service galaxy-handler@1.service
     sudo systemctl start galaxy-handler@0.service galaxy-handler@1.service
     ```
     This ensures every handler picks up the new mapping of `classviz → docker_env`.

2. **Handlers or Gunicorn processes stubbornly stick around**  
   A simple `restart` can leave old processes running, so fixes aren’t applied uniformly.  
   **Workaround**: Stop the handler units, verify no old `main.py` processes remain, then start them again so every worker picks up the latest code.

3. **Docker “permission denied” errors**  
   If Galaxy can’t spawn containers, you’ll see permissions failures.  
   **Workaround**: Add your Galaxy service user to the Docker group and re-login so it can launch images without sudo.
     ```bash
     sudo usermod -aG docker galaxy_usr
     # Log out and back in, or restart the service
     ```

4. **Job_conf.yml not loaded**  
   If you already have a job_config: inline in your `galaxy.yml`, it won't load job_conf.yml. Comment it, or adjust your job_config.  


## Recapitulative Check List
- [ ] Ensure you have the prerequisites
- [ ] Run the playbook
- [ ] Adapt your configuration files if needed
- [ ] Install dependencies via the Admin interface

 
