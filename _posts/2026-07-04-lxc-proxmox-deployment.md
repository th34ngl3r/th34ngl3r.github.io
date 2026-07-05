---
layout: post
title: "Automation in the Home Lab: Part 1: Provisioning Proxmox LXC Containers with Ansible"
date: 2026-07-04 06:00:00 -0500
categories: [Linux,Proxmox]
tags: [Ansible,Proxmox]
author: Th3_4ngl3r
toc: true
comments: false
---

If you love tinkering with your home lab, you know that spinning up fresh environments is half the fun—and doing it manually is the tedious half. Today, we’re going to eliminate the click-ops and look at how to fully automate the creation of a Linux Container (LXC) on Proxmox VE using Ansible.

By the end of this post, you'll see how a single playbook can handle authentication, hardware allocation, networking, and security configurations in seconds.

Here is the exact Ansible playbook we will be breaking down today:

``` yaml
---
- name: Create Proxmox LXC Container
  hosts: pve2.home.lab   # Matches your current test host
  gather_facts: false
  
  vars_files:
    - vars.yml

  tasks:
    - name: Create LXC container 202 (ca.home.lab)
      community.proxmox.proxmox:
        # --- API Authentication Parameters ---
        api_host: "localhost"
        api_user: "{{ vars_pve_api_user }}"
        api_password: "{{ vars_pve_api_password }}"
        validate_certs: false  # <--- BYPASS SELF-SIGNED SSL ERRORS
        
        # --- Container Configuration ---
        vmid: 202
        hostname: ca
        password: "{{ vars_lxc_password }}"
        node: "{{ ansible_hostname | default('pve2') }}"
        ostemplate: "local:vztmpl/rocky9.tar.xz"
        unprivileged: true
        features:
          - nesting=1
        tags: rocky
        cores: 2
        memory: 4096
        swap: 2048
        disk: "DS01:20"
        netif:
          net0: "name=eth0,bridge=vmbr0,firewall=0,ip=10.10.10.7/24,gw=10.10.10.1"
        searchdomain: home.lab
        nameserver: 10.10.10.4
        
        # --- State Configuration ---
        state: present
```

## Why Automate Proxmox with Ansible?

Before diving into the code, let’s answer the why.

1) **Consistency**: Avoid human error. Every time you run this playbook, you get the exact same configuration.

2) **Speed**: Going through the Proxmox GUI web interface takes a few minutes. This playbook takes a few seconds.

3) **Documentation as Code**: Your infrastructure configuration lives in a text file. If your Proxmox node dies, rebuilding your specific containers is a breeze.

## Playbook Breakdown: How It Works

Let's dissect this playbook section by section.

### 1. The Play Header & Secrets Management

``` yaml
- name: Create Proxmox LXC Container
  hosts: pve2.home.lab
  gather_facts: false
  
  vars_files:
    - vars.yml
```

- `hosts`: pve2.home.lab: This tells Ansible exactly which machine to connect to. In this case, it targets our Proxmox node directly.

- `gather_facts`: false: Since we are just sending an API request to Proxmox, we don't need to waste time gathering setup data about the host machine beforehand.

- `vars_files`: - vars.yml: Security matters, even in a home lab! We are pulling encrypted sensitive data (like passwords and API tokens) from an Ansible Vault file (vars.yml) to keep them out of plaintext.

### 2. The Core Task & Module

``` yaml
tasks:
    - name: Create LXC container 202 (ca.home.lab)
      community.proxmox.proxmox:
```

We use the official `community.proxmox.proxmox` module. This powerful module interfaces directly with the Proxmox VE API, meaning we don't have to manually write complex curl commands or SSH scripts.

### 3. API Authentication

- `api_host`: "localhost": Because the playbook runs on the Proxmox host itself, it can safely talk to the API locally.

- **Variables** (`{{ ... }}`): This pulls the credentials securely out of our `vars.yml` file.

- `validate_certs: false:` Most home labs use self-signed SSL certificates. Setting this to `false` prevents Ansible from throwing an error over an untrusted certificate. (Note: Don't do this in production!)

### 4. Container Resource & OS Configuration

This is where the magic happens. We define exactly what this container looks like.

| Parameter | Value | Description |
| :--- | :--- | :--- |
| **vmid** | 202 | The unique ID of the container inside Proxmox. |
| **hostname** | ca | The hostname given to the container (e.g., Certificate Authority). |
| **ostemplate** | local:vztmpl/rocky9.tar.xz | Uses a pre-downloaded Rocky Linux 9 template stored locally on Proxmox. |
| **unprivileged** | true | Enhances security by ensuring the root user inside the container doesn't have root privileges on the host. |
| **features** | nesting=1 | Enables Docker/systemd to run smoothly inside the container. |

We also define the virtual hardware limits:

- cores: 2: Allocates 2 CPU cores.

- memory: 4096: Allocates 4GB of RAM.

- swap: 2048: Allocates 2GB of swap space.

- disk: "DS01:20": Provisions a 20GB root disk on the storage pool named DS01.

**NOTE**: Hardware limits can also be defined in the vars.yml

### 5. Network & DNS Configuration

``` yaml
netif:
          net0: "name=eth0,bridge=vmbr0,firewall=0,ip=10.10.10.7/24,gw=10.10.10.1"
        searchdomain: home.lab
        nameserver: 10.10.10.4
```

Instead of relying on DHCP, this container is provisioned with a predictable, static network profile:

- Interface (`net0`): Creates a virtual interface named `eth0`, binds it to the primary network bridge (`vmbr0`), disables the built-in Proxmox firewall for this interface, assigns the static IP `10.10.10.7`, and points to `10.10.10.1` as the gateway.

- DNS: Points to a local nameserver (`10.10.10.4`) and configures the search domain as `home.lab` so local devices can resolve each other by name easily.

### 6. The Desired State

```yaml
    state: present
```

Ansible is **idempotent**. By setting `state: present`, we are telling Ansible: "Make sure this container exists with these exact specs."

If the container doesn't exist, Ansible will build it. If it does exist and matches the specs, Ansible will do absolutely nothing. This ensures you can run the playbook repeatedly without breaking your active lab.

### Wrapping Up

With this playbook, setting up a new server in your home lab goes from a multi-step click routine to a 5-second automated task. You can easily tweak the `vmid`, `hostname`, and `ip` fields to roll out an entire fleet of Linux containers for your home lab projects.

## What's Next?

Stay tuned! Up next, I will post a follow-up guide showing how Ansible can automate connecting to this newly created container and, as an example, install the EBCA (Enterprise Business Class Authority) application for creating and managing your own custom certificates!

*Check out [Part 2: Automation in the Home Lab: Provisioning EJBCA Inside an LXC Containers with Ansible](https://thelotusbar.com/posts/ejbca-deployment/) before diving in.)*

This will be followed by part 3: The configuration of ECBA and certificates with a bonus announcement at then end.
