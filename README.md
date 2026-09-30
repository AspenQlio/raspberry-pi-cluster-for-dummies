# Raspberry Pi Homelab Cluster

> **[Español] → La documentación completa, con instalación paso a paso y los errores que me encontré, está en [README.es.md](README.es.md).**

A practical baseline for deploying a minimal, two-node Raspberry Pi homelab cluster. 

## Architecture
The setup relies on decoupling services across two nodes to maintain DNS resolution if the primary application node goes offline.

- **Node 1 (Gateway):** Handles network-wide DNS routing, ad-blocking, and secure split-tunnel remote access.
- **Node 2 (Storage & Media):** Manages containerized applications, network-attached storage, and local media streaming.

## Services Deployed
- **Networking:** Tailscale (Mesh VPN) for secure remote access without exposing inbound router ports.
- **Security:** UFW configuration and SSH hardening (key-based authentication only).
- **Applications:** Docker, Portainer, Samba (SMB), Jellyfin/Plex.

This repository contains the configuration steps and scripts necessary to bootstrap a secure, bare-metal local cloud.
