# Terraform for Deployment

<!-- @import "[TOC]" {cmd="toc" depthFrom=2 depthTo=4 orderedList=false} -->
<!-- code_chunk_output -->

- [Objective](#objective)
- [Pre-requisite](#pre-requisite)
- [Deployment Workflow](#deployment-workflow)
    - [Operating System](#operating-system)
    - [Upfront Nginx Web Server](#upfront-nginx-web-server)
    - [Jumpbox](#jumpbox)
    - [Block Disks](#block-disks)
    - [Docker and Kubernetes](#docker-and-kubernetes)
    - [Nginx and Ingress Network Controller](#nginx-and-ingress-network-controller)

<!-- /code_chunk_output -->

## Objective

- Use Terraform to simplify to deploy a Kubernetes platform and a bunch of common applications, such as Nginx, MongoDB, Prometheus, etc
- Standardize Terraform script to deploy on whatever virtual_machines

## Pre-requisite

- An existing virtual_machine platform. Doesn't matter they're from OpenStack, or VMware
- The virtual_machine are networked functionally
- Need several block-disks for MongoDB and others
- A DNS that can resolve service.domain.com

## Deployment Workflow

#### Operating System
- Create `nonroot` user and add into `sudoer`
- System hardening
  - `firewalld`
  - `sshd`
- Modify `/etc/hosts`
- Setup `sshd` and `ssh-copy-id` to all worker-node

#### Upfront Nginx Web Server

#### Jumpbox 

#### Block Disks
- Format disks into `xfs` for MongoDB and `ext4` filesystem format for other application
- Mount disks to respective virtual_machines

#### Docker and Kubernetes
- Install docker and kubernetes
- Grant appropriate user access for both
- Create persistentVolume

#### Nginx and Ingress Network Controller
- Apply custom SSL
- Apply VANTIQ license key
