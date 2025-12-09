# Cephadm-Ansible

This repo is all about automating Ceph cluster deployment and management with Ansible and cephadm. It's got roles and playbooks for everything from bootstrapping the cluster to adding/removing daemons, setting up monitoring, logging, and load balancing. Think of it as a flexible toolkit for getting a solid Ceph setup running without too much hassle.

## Repository Diagram

![Cluster Diagram](img/cluster_diagram.svg)

## Getting Started

Before diving in, make sure you've got your environment prepped

### Prerequisites

- Ansible 2.12 or higher
- Python 3.x on your control machine
- SSH access to all nodes (passwordless preferred)
- Root or sudo privileges on nodes
- Internet access for package downloads (or use internal mirrors)
- Nodes should be running a compatible OS (tested on Ubuntu 22.04+)

First things first, set up SSH key-based authentication. This makes everything smoother and more secure. On your control machine (where you'll run Ansible):

1. Generate an SSH key if you don't have one:  
   ```
   ssh-keygen -t ed25519 -C "your_email@example.com"
   ```

2. Copy the public key to all nodes (including the bootstrap node). You can use `ssh-copy-id` for this:  
   ```
   ssh-copy-id user@node-ip
   ```
   Repeat for every node in your inventory. If you're using root, replace `user` with `root`.

Next, install the Python dependencies for Ansible. There's a `requirements.txt` in the root:  
```
pip install -r requirements.txt
```

This pulls in stuff like Ansible 9.0.1, Jinja2, PyYAML, and a few others to keep things running smooth.

### Inventory Setup

Edit `inventory/hosts.ini` to match your setup. Here's a sample:  

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
mds1 ansible_host=192.168.10.70

[haproxy]
hap1 ansible_host=192.168.10.50
hap2 ansible_host=192.168.10.51
hap3 ansible_host=192.168.10.52

[monitoring]
log1 ansible_host=192.168.10.60
log2 ansible_host=192.168.10.61
```

Tweak `group_vars/all.yml` for global settings like Ceph versions, network ranges, or custom paths.

### Installation and Bootstrap

We're using cephadm for the heavy lifting, so the process is modular. Start with the bootstrap node (usually the first MON). 

Because in the current configuration the OSD nodes are connected to both the public network and the cluster network, Ceph needs to “see” that cluster network during bootstrap. Temporarily add a virtual IP on the bootstrap node’s loopback interface: 
```
sudo ip addr add 192.168.20.1/24 dev lo
```
You can remove it after bootstrap with `sudo ip addr del 192.168.20.1/24 dev lo`. This just tricks Ceph into recognizing the network early on.

Now, let's get rolling:

1. Install prerequisites and Docker on the bootstrap node:  
   ```
   ansible-playbook playbooks/all.yaml --tags docker,prereqs --limit bootstrap
   ```

2. Install cephadm on the bootstrap node:  
   ```
   ansible-playbook playbooks/all.yaml --tags installcephadm --limit bootstrap
   ```

3. Bootstrap the Ceph cluster. This uses the config from `roles/bootstrap_ceph/templates/initial-ceph.conf.j2`, so edit that if you need custom settings:  
   ```
   ansible-playbook playbooks/all.yaml --tags bootstrap --limit bootstrap
   ```

Once bootstrapped, you can add more nodes/daemons. Run these from the control machine, specifying the host to add with `-e "ceph_host_to_add=hostname"`. Always limit to bootstrap for these ops since they orchestrate from there.

- Add monitors:  
  ```
  ansible-playbook playbooks/all.yaml --tags add-mon -e "ceph_host_to_add=mon2"
  ansible-playbook playbooks/all.yaml --tags add-mon -e "ceph_host_to_add=mon3"
  ```

- Add managers:  
  ```
  ansible-playbook playbooks/all.yaml --tags add-mgr -e "ceph_host_to_add=mgr1"
  ansible-playbook playbooks/all.yaml --tags add-mgr -e "ceph_host_to_add=mgr2"
  ansible-playbook playbooks/all.yaml --tags add-mgr -e "ceph_host_to_add=mgr3"
  ```

- Add RGWs:  
  ```
  ansible-playbook playbooks/all.yaml --tags add-rgw -e "ceph_host_to_add=rgw1"
  ansible-playbook playbooks/all.yaml --tags add-rgw -e "ceph_host_to_add=rgw2"
  ansible-playbook playbooks/all.yaml --tags add-rgw -e "ceph_host_to_add=rgw3"
  ```

- Add OSDs:  
  ```
  ansible-playbook playbooks/all.yaml --tags add-osd -e "ceph_host_to_add=osd1"
  ansible-playbook playbooks/all.yaml --tags add-osd -e "ceph_host_to_add=osd2"
  # And so on for more OSDs
  ```

- Add MDS (for CephFS if needed):  
  ```
  ansible-playbook playbooks/all.yaml --tags add-mds -e "ceph_host_to_add=mds1"
  ```

### Additional Features

Once the core cluster is up, layer on extras:

- **HAProxy for Cluster Access**: This sets up load balancing directly to RGWs for reliable access. Deploy on your HAProxy nodes:  
  ```
  ansible-playbook playbooks/all.yaml --tags haproxy --limit haproxy
  ```

- **Fluent Bit for Logging**: Runs on HAProxy nodes to forward logs to the monitoring stack. Pair it with HAProxy deployment:  
  ```
  ansible-playbook playbooks/all.yaml --tags fluent --limit haproxy
  ```

- **ELK Stack for Logging**: Deploys Elasticsearch, Logstash, and Kibana on monitoring nodes for centralized logging:  
  ```
  ansible-playbook playbooks/all.yaml --tags elk --limit monitoring
  ```

- **Prometheus/Grafana Monitoring**: Separate from Ceph's built-in monitoring, this deploys a full stack on monitoring nodes:  
  ```
  ansible-playbook playbooks/all.yaml --tags monitoring --limit monitoring
  ```

- **Erasure Coding Profiles**: Set up EC pools for better storage efficiency:  
  ```
  ansible-playbook playbooks/all.yaml --tags ec_pools
  ```

- **Block Storage (RBD)**: Enable dependencies for replicated block devices:  
  ```
  ansible-playbook playbooks/all.yaml --tags dep-rbd
  ```

### Removing Nodes/Daemons

If you need to scale down or remove something, use the remove roles. Specify the host with `-e "ceph_host_to_remove=hostname"` and limit to bootstrap:

- Remove a monitor:  
  ```
  ansible-playbook playbooks/all.yaml --tags rm-mon -e "ceph_host_to_remove=mon3" --limit bootstrap
  ```

- Remove a manager:  
  ```
  ansible-playbook playbooks/all.yaml --tags rm-mgr -e "ceph_host_to_remove=mgr3" --limit bootstrap
  ```

- Remove an RGW:  
  ```
  ansible-playbook playbooks/all.yaml --tags rm-rgw -e "ceph_host_to_remove=rgw3" --limit bootstrap
  ```

- Remove an OSD:  
  ```
  ansible-playbook playbooks/all.yaml --tags rm-osd -e "ceph_host_to_remove=osd3" --limit bootstrap
  ```

- Remove an MDS:  
  ```
  ansible-playbook playbooks/all.yaml --tags rm-mds -e "ceph_host_to_remove=mds1" --limit bootstrap
  ```

Be careful with removals—always check cluster health first with `ceph status`.

## Roles Overview

Here's a quick rundown of the roles:

| Role            | Description                                                                 |
|-----------------|-----------------------------------------------------------------------------|
| `install_prereqs` | Handles system packages and basics                                         |
| `install_docker` | Sets up Docker on nodes                                                    |
| `install_cephadm` | Installs cephadm and repos                                                 |
| `bootstrap_ceph` | Bootstraps the first MON/MGR                                               |
| `add_mon`, `add_mgr`, `add_mds`, `add_osd`, `add_rgw` | Adds daemons to the cluster                                                |
| `remove_mon`, `remove_mgr`, `remove_mds`, `remove_osd`, `remove_rgw` | Safely removes daemons                                                     |
| `monitoring`    | Deploys Prometheus and Grafana for metrics                                 |
| `elk`           | Sets up ELK (Elasticsearch, Logstash, Kibana) for logs                     |
| `fluent`        | Deploys Fluent Bit/Fluentd for log forwarding                              |
| `haproxy`       | Configures HAProxy for load balancing RGWs                                 |
| `ec_pools`      | Creates erasure coding profiles and pools                                  |
| `dep-rbd`       | Enables dependencies for RBD block storage                                 |
| `add_hosts`     | Utility for adding hosts to the cluster                                    |

## Templates

Customize configs with these Jinja2 templates:
- `ceph.conf` for Ceph settings
- Docker Compose files for stacks like ELK, Fluent, HAProxy, Monitoring
- Fluent Bit parsers and configs
- Prometheus scrape jobs
- HAProxy backend/frontend rules
- Logstash pipelines

Find them in the respective role's `templates/` dir.

## Repository Structure

```
.
├── README.md
├── ansible.cfg
├── group_vars/
│   └── all.yml
├── img/
│   └── cluster_diagram.svg
├── inventory/
│   └── hosts.ini
├── playbooks/
│   ├── all.yaml
│   └── run_install_docker_prereqs.yml
├── requirements.txt
└── roles/
    ├── add_hosts/
    ├── add_mds/
    ├── add_mgr/
    ├── add_mon/
    ├── add_osd/
    ├── add_rgw/
    ├── bootstrap_ceph/
    ├── dep-rbd/
    ├── ec_pools/
    ├── elk/
    ├── fluent/
    ├── haproxy/
    ├── install_cephadm/
    ├── install_docker/
    ├── install_prereqs/
    ├── monitoring/
    ├── remove_mds/
    ├── remove_mgr/
    ├── remove_mon/
    ├── remove_osd/
    └── remove_rgw/
```
