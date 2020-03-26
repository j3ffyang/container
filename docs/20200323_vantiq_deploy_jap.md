
# VANTIQ Deployment High-level Architecture

<!-- TOC depthFrom:2 depthTo:6 withLinks:1 updateOnSave:1 orderedList:0 -->

- [Basic Skill Required](#basic-skill-required)
- [Operating System](#operating-system)
- [Security and System-Hardening](#security-and-system-hardening)
- [Network pre-requisite for public service](#network-pre-requisite-for-public-service)
- [Container](#container)
- [Images (installed through Helm)](#images-installed-through-helm)
- [VANTIQ](#vantiq)

<!-- /TOC -->

## Basic Skill Required

- Linux
- Container and Kubernetes
- Network
- Open Source software

## Operating System

- Ubuntu 18.04 LTS Server * 6 (single master)
- Disk for persistent volume and persistent volume claim
  - MongoDB * 2 (primary and secondary)
  - Grafana * 2 (grafana and grafanaDB)

## Security and System-Hardening
- Software firewall, eg, `ufw` on Ubuntu
- SSHd hardening, to ban rootlogin, PasswordAuthentication, keyonly

## Network pre-requisite for public service

- Upfront Nginx for web service
- Nginx's HA & Load-Balancing

## Container

- Docker latest 19.x
- Kuberentes 1.15.11 the latest supported by VANTIQ
- Flannel, SDN within intranet

## Images (installed through Helm)
- Public for all public service, such as MySQL, Nginx, etc
- Private from dockerhub.io

## VANTIQ
- Deployment_tool and VANTIQ_version
- Sealed secrets feature
- Access to private code at Github (2FA)
- SSL (self-signed or CA signed) for Nginx & VANTIQ. Replaceable if the future
- Keycloak, if yes, needs PV for PostgreSQL to store Keycloak identify
