# Raspberry Pi 3B+ Docker Swarm with Ansible Automation

## Overview

This project demonstrates the process of building a lightweight container orchestration environment using two Raspberry Pi 3B+ boards running Ubuntu Server 24.04. The deployment process is fully automated with Ansible, enabling consistent, repeatable provisioning of Docker and its dependencies, and seamless joining of nodes into a Docker Swarm cluster.

---

## Objectives

1. Set up Raspberry Pi 3B+ devices with Ubuntu Server 24.04.
2. Automate the installation of Docker and its dependencies using Ansible.
3. Configure one Pi as a **Docker Swarm manager** and others as a **worker node**.
4. Demonstrate cluster functionality by deploying a sample containerized application.

---

## Hardware and Software Requirements

### Hardware

- 2 x Raspberry Pi 3B+ boards
- 2 x 32 GB (or larger) microSD cards (Class 10 recommended)
- Power supplies for each Pi
- Ethernet cables (recommended for stable networking)
- Network switch or router
- _Optional:_ PoE hats for the Raspberry Pi's to reduce cabling

### Software

- **Ubuntu Server 24.04 ARM64** image
- **Ansible** (installed on a control machine, e.g., laptop or third Pi)
- **Docker Engine & Docker CLI**
- **Python 3** (for Ansible compatibility)

---

## Step 1: Preparing the Raspberry Pis

1. **Flash Ubuntu Server 24.04** onto each microSD card using [Raspberry Pi Imager](https://www.raspberrypi.com/software/).
2. Use the options to set hostname, create a user, and define SSH keys
3. Insert SD cards, connect Ethernet, and power on the Pis.

---

## Step 2: Ansible Inventory Setup

Create an inventory file `hosts.ini`:

```ini
[swarm-manager]
pi-manager ansible_host=<your_manager_ip> ansible_user=<your_manager_user>

[pis]
pi-worker01 ansible_host=<your_raspi_worker01_ip> ansible_user=<your_worker_user> # CHANGE THIS
pi-worker02 ansible_host=<your_raspi_worker02_ip> ansible_user=<your_worker_user> # CHANGE THIS
# Add more hosts as needed. Make sure to change your IP and Usernames as needed
```

---

## Step 3: Writing the Ansible Playbook

Create `docker-swarm.yml`:

```yaml
---
- name: Setup Docker and join Docker Swarm
  hosts: pis
  become: yes
  tasks:
    - name: Update apt cache and upgrade packages
      apt:
        update_cache: yes
        upgrade: dist

    - name: Install prerequisite packages
      apt: 
        name: 
          - apt-transport-https
          - ca-certificates
          - curl
          - software-properties-common
        state: present
    
    - name: Add Docker's official GPG key
      ansible.builtin.apt_key:
        url: https://download.docker.com/linux/ubuntu/GPG
        state: present
    
    - name: Add Docker apt repository
      apt_repository:
        repo: "deb [arch=arm64] https://download.docker.com/linux/ubuntu {{ ansible_distribution_release }} stable"
        state: present
        filename: docker

    - name: Install Docker packages
      apt: 
        name: 
          - docker-ce
          - docker-ce-cli
          - containerd.io
          - docker-buildx-plugin
          - docker-compose-plugin
        state: present
        update_cache: yes

    - name: Enable and start Docker service
      systemd: 
        name: docker
        enabled: yes
        state: started

- name: Initialize Docker Swarm
  hosts: manager
  become: yes
  vars:
    swarm_manager_ip: "your_swarm_manager_ip" # CHANGE THIS
  tasks:
    - name: Initialize Swarm
      command: docker swarm init --advertise-addr {{ swarm_manager_ip }}
      register: swarm_init
      failed_when: >
        swarm_init.rc !=0 and
        ('This node is already part of a swarm' not in swarm_init.stderr)
      changed_when: "'Swarm initialized' in (swarm_init.stdout | default(''))"

- name: Join Docker Swarm as Worker
  hosts: pis
  become: yes
  vars: 
    manager_host: "{{ groups['swarm-manager'][0] }}"
    manager_ip: "{{ hostvars[manager_host].ansible_host }}"
  tasks: 
    - name: Get token from manager
      command: docker swarm join-token -q worker
      register: worker_token
      delegate_to: "{{ manager_host }}"
      run_once: true

    - name: Join swarm (safe re-run)
      command: "docker swarm join --token {{ worker_token.stdout }} {{ manager_ip }}:2377"
      register: join_result
      failed_when: >
        join_result.rc != 0 and
        ('This node is already part of a swarm' not in (join_result.stderr | default('')))
```

---

## Step 5: Running the Playbook

```bash
ansible-playbook -i hosts.ini docker-swarm.yml
```

---

## Step 6: Testing the Swarm

Verify nodes:

```bash
ssh <your_manager_username>@<your_manager_ip>
docker node ls
```

Deploy a test service:

```bash
docker service create --name hello-world --replicas 2 nginx:alpine
docker service ls
```

---

## Results

- **Fully automated provisioning** of Raspberry Pi nodes with Docker.
- Working **three-node Docker Swarm** (1 manager, 2 worker).
- Scalability - adding more Pis only requires adding them to the Ansible inventory and re-running the playbook.
- Lightweight orchestration suitable for IoT, edge computing, or development testbeds.

---

## Potential Improvements

- Implement **Ansible roles** for cleaner playbook organization.
- Use **dynamic inventory** scripts for larger clusters.
- Automate application deployment pipelines with **CI/CD integration**.
- Add **monitoring stack** (Prometheus, Grafana) to observe swarm health.

---
