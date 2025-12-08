# Cephadm-Ansible

This repository provides an Ansible-based automation framework for deploying and managing a Ceph cluster using **cephadm**.  
It includes modular roles and playbooks for installation, bootstrap, daemon management, monitoring, logging, and load balancing.

## Repository Diagram

![Cluster Diagram](img/cluster_diagram.svg)

## Getting Started

### 1. Prepare the Inventory

Edit the inventory file:

```
[mons]
mon1 ansible_host=192.168.10.10
mon2 ansible_host=192.168.10.11
mon3 ansible_host=192.168.10.12

[mgrs]
mgr1 ansible_host=192.168.10.40
mgr2 ansible_host=192.168.10.41
mgr3 ansible_host=192.168.10.42


[osds]
osd1 ansible_host=192.168.10.20
osd2 ansible_host=192.168.10.21
osd3 ansible_host=192.168.10.22
osd4 ansible_host=192.168.10.23

[rgws]
rgw1 ansible_host=192.168.10.30
rgw2 ansible_host=192.168.10.31
rgw3 ansible_host=192.168.10.32

[bootstrap]
mon1 ansible_host=192.168.10.10

[ceph_nodes:children]
mons
mgrs
osds
rgws

[mdss]


[haproxy]
hap1 ansible_host=192.168.10.50
hap2 ansible_host=192.168.10.51
hap3 ansible_host=192.168.10.52


[monitoring]
log1 ansible_host=192.168.10.60
log2 ansible_host=192.168.10.61
````

Define global variables in:

```
group_vars/all.yml
```

### 2. Install Prerequisites and Docker

```
ansible-playbook -i inventory/hosts.ini playbooks/run_install_docker_prereqs.yml
```

### 3. Bootstrap the Ceph Cluster

```
ansible-playbook -i inventory/hosts.ini playbooks/all.yaml
```

This playbook may include:

* Prerequisite installation
* Docker installation
* cephadm installation
* Ceph bootstrap
* Adding MON/MGR/OSD and other daemons

Edit `all.yaml` to match your deployment workflow.

## Roles Overview

| Role                                                  | Description                                |
| ----------------------------------------------------- | ------------------------------------------ |
| `install_prereqs`                                     | Installs required system packages          |
| `install_docker`                                      | Installs and configures Docker             |
| `install_cephadm`                                     | Installs cephadm and required repositories |
| `bootstrap_ceph`                                      | Bootstraps the initial Ceph MON and MGR    |
| `add_mon`, `add_mgr`, `add_mds`, `add_osd`, `add_rgw` | Adds Ceph daemons                          |
| `remove_*` roles                                      | Removes Ceph daemons                       |
| `monitoring`                                          | Deploys Prometheus/Grafana monitoring      |
| `elk`                                                 | Deploys ELK logging stack                  |
| `fluent`                                              | Deploys Fluent Bit / Fluentd stack         |
| `haproxy`                                             | Deploys HAProxy load balancer              |

## Templates

Included Jinja2 templates allow customization of:

* `ceph.conf`
* Docker Compose files
* Fluent-bit and Logstash configuration
* Prometheus configuration

## Requirements

* Ansible 2.12+
* Python 3.x
* SSH access to nodes
* Root or passwordless sudo privileges
* Internet access or internal mirrors

## Repository Structure

```
.
├── ansible.cfg
├── inventory/
│   └── hosts.ini
├── group_vars/
│   └── all.yml
├── playbooks/
│   ├── all.yaml
│   └── run_install_docker_prereqs.yml
└── roles/
├── install_prereqs/
├── install_docker/
├── install_cephadm/
├── bootstrap_ceph/
├── add_mon/ add_mgr/ add_mds/ add_osd/ add_rgw/
├── remove_mon/ remove_mgr/ remove_mds/ remove_osd/ remove_rgw/
├── monitoring/
├── elk/
├── fluent/
├── haproxy/
└── others...

```
