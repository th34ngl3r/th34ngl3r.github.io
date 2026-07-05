---
layout: post
title: "Automation in the Home Lab: Provisioning EJBCA Inside an LXC Containers with Ansible"
date: 2026-07-05 06:00:00 -0500
categories: [Linux,Proxmox]
tags: [Ansible,Proxmox]
author: Th3_4ngl3r
toc: true
comments: false
---
## Introduction

If you spend enough time managing complex environments or deep-diving into network security, you quickly realize that manual deployments are the enemy of a resilient architecture. Spinning up fresh, secure services should be seamless, repeatable, and entirely free of tedious "click-ops."

A robust Public Key Infrastructure (PKI) is a foundational element for securing internal services, managing access, and ensuring proper encryption across your network. EJBCA is an industry heavyweight for this exact purpose, but its traditional installation process can be heavy and complex. By pairing it with Docker and Ansible, we can transform a cumbersome deployment into a clean, automated, and documented process.

In Part 2 of this series, we are taking our infrastructure-as-code (IaC) journey a step further by fully automating the deployment of EJBCA inside a Docker environment on Rocky Linux using Ansible.

*(If you missed the foundational setup, be sure to check out [Part 1: Automation in the Home Lab: Provisioning Proxmox LXC Containers with Ansible](https://thelotusbar.com/posts/lxc-proxmox-deployment/) before diving in.)*

Here is the exact Ansible playbook we will be breaking down today:

``` yaml
---
- name: Deploy EJBCA on Docker (Rocky Linux)
  hosts: ca.home.lab
  become: yes
  vars:
    # Docker configuration
    docker_packages:
      - docker-ce
      - docker-ce-cli
      - containerd.io
      - docker-buildx-plugin
      - docker-compose-plugin
    
    # EJBCA configuration
    ejbca_container_name: ejbca-server
    ejbca_image: keyfactor/ejbca-ce:latest
    ejbca_http_port: 8080
    ejbca_https_port: 8443

  tasks:
    ## 1. Install System Dependencies
    - name: Install prerequisite system packages
      dnf:
        name:
          - curl
          - python3-pip
        state: present
        update_cache: yes

    - name: Install Python Docker SDK via pip
      pip:
        name: docker
        state: present

    ## 2. Set up Docker Repository & Install Docker
    - name: Install dnf-plugins-core
      dnf:
        name: dnf-plugins-core
        state: present

    - name: Add Docker stable repository
      ansible.builtin.get_url:
        url: https://download.docker.com/linux/centos/docker-ce.repo
        dest: /etc/yum.repos.d/docker-ce.repo
        mode: '0644'

    - name: Install Docker Engine and plugins
      dnf:
        name: "{{ docker_packages }}"
        state: present
        update_cache: yes

    - name: Ensure Docker service is started and enabled
      service:
        name: docker
        state: started
        enabled: yes

    ## 3. Configure and Run EJBCA Container
    - name: Pull EJBCA Community Edition image
      community.docker.docker_image:
        name: "{{ ejbca_image }}"
        source: pull

    - name: Create and start EJBCA container
      community.docker.docker_container:
        name: "{{ ejbca_container_name }}"
        image: "{{ ejbca_image }}"
        state: started
        restart_policy: unless-stopped
        ports:
          - "{{ ejbca_http_port }}:8080"
          - "{{ ejbca_https_port }}:8443"
        env:
          ACCEPT_DEFAULT_ALLOW: "true" 
        expose:
          - "8080"
          - "8443"
      register: ejbca_container_status

    ## 4. Post-Deployment Info Retrieval
    - name: Wait for EJBCA initialization
      pause:
        seconds: 20

    - name: Display EJBCA setup details
      debug:
        msg: 
          - "EJBCA is successfully deployed!"
          - "Management URL: https://ca.home.lab:{{ ejbca_https_port }}/ejbca/adminweb/"

```

## Phase 1: Prepping the System and Python Environment

Ansible doesn't just run shell commands; it relies on specialized modules to interact with services. To manage Docker containers, Ansible uses the `community.docker collection`, which in turn requires the Docker Python SDK to be present on the target host.

``` yaml
tasks:
    - name: Install prerequisite system packages
      dnf:
        name:
          - curl
          - python3-pip
        state: present
        update_cache: yes

    - name: Install Python Docker SDK via pip
      pip:
        name: docker
        state: present
```

### What is happening

1. We use the `dnf` package manager (standard for Rocky/RHEL systems) to ensure `curl` and `python3-pip` are installed.

2. We then use the `pip` module to install the `docker` Python package.

**Why**? Without that Python SDK, Ansible's `docker_container` tasks will immediately fail with a missing dependency error. This step bridges the gap between Ansible's Python execution environment and the Docker daemon.

## Phase 2: Installing the Docker Engine

Rocky Linux doesn't ship with the official Docker repositories by default. We need to add them before installing the engine.

``` yaml
- name: Install dnf-plugins-core
      dnf:
        name: dnf-plugins-core
        state: present

    - name: Add Docker stable repository
      ansible.builtin.get_url:
        url: [https://download.docker.com/linux/centos/docker-ce.repo](https://download.docker.com/linux/centos/docker-ce.repo)
        dest: /etc/yum.repos.d/docker-ce.repo
        mode: '0644'

    - name: Install Docker Engine and plugins
      dnf:
        name: "{{ docker_packages }}"
        state: present
        update_cache: yes

    - name: Ensure Docker service is started and enabled
      service:
        name: docker
        state: started
        enabled: yes
```

What is happening

1. We install `dnf-plugins-core`, which provides utilities for managing DNF repositories.

2. We pull the official CentOS Docker repository file directly from Docker's servers (CentOS repos are fully compatible with Rocky Linux) and drop it into `/etc/yum.repos.d/`.

3. We install the suite of Docker packages defined in our `vars` section.

4. Finally, the `service` module ensures the Docker daemon is currently running and is set to launch automatically if the system reboots.

## Phase 3: Deploying the EJBCA Container

With the host prepped, we move to the core objective: getting EJBCA up and running.

``` yaml
- name: Pull EJBCA Community Edition image
      community.docker.docker_image:
        name: "{{ ejbca_image }}"
        source: pull

    - name: Create and start EJBCA container
      community.docker.docker_container:
        name: "{{ ejbca_container_name }}"
        image: "{{ ejbca_image }}"
        state: started
        restart_policy: unless-stopped
        ports:
          - "{{ ejbca_http_port }}:8080"
          - "{{ ejbca_https_port }}:8443"
        env:
          ACCEPT_DEFAULT_ALLOW: "true" 
        expose:
          - "8080"
          - "8443"
      register: ejbca_container_status
```

What is happening

1. We pull the `keyfactor/ejbca-ce:latest` image and spin it up with specific port mappings and environment variables. The `restart_policy: unless-stopped` ensures the CA survives a host reboot or Docker daemon restart.

2. Why is `ACCEPT_DEFAULT_ALLOW`: "true" so important?
EJBCA is highly secure by default. Typically, to access the admin web interface, you are required to authenticate using a client certificate. However, on a fresh deployment, you haven't generated any client certificates yet!

3. Setting `ACCEPT_DEFAULT_ALLOW`: "true" temporarily bypasses this strict mutual TLS (mTLS) requirement, allowing you to access the web UI using standard username/password credentials on initial boot so you can perform the initial setup.

## Phase 4: Validation and Handoff

Enterprise Java applications are notorious for taking a moment to fully initialize their internal web servers (like WildFly/JBoss) and databases.

``` yaml
- name: Wait for EJBCA initialization
      pause:
        seconds: 20

    - name: Display EJBCA setup details
      debug:
        msg: 
          - "EJBCA is successfully deployed!"
          - "Management URL: [https://ca.home.lab](https://ca.home.lab):{{ ejbca_https_port }}/ejbca/adminweb/"
```

### What is happening?

The playbook artificially pauses for 20 seconds. Afterward, it prints a clean, formatted debug message displaying the final Management URL.

### Why?

If you try to hit the web UI the exact millisecond the container starts, you will likely get a 502 Bad Gateway or a connection refused error. Giving it a 20-second buffer ensures that by the time the Ansible run finishes, the EJBCA admin interface is actually ready for you to log in.

## Conclusion

By breaking this down into an Infrastructure-as-Code (IaC) model, setting up a complex Certificate Authority becomes a trivial, single-command operation (`ansible-playbook deploy_ejbca.yml`). This ensures your environment remains reproducible, documented, and easy to rebuild in the event of a failure.
