# Kubernetes Platform Components

This repository provides a production-ready reference stack of essential Kubernetes platform components. It serves as both a personal reference and a starting point for building secure, resilient infrastructure in real-world Kubernetes environments.

## 🔧 Components Included

- **[OpenBao](https://github.com/openbao/openbao)** – Open-source secrets management (Vault fork)
- **Vault Secrets Operator** – Automates syncing secrets from OpenBao to Kubernetes workloads
- **[cert-manager](https://cert-manager.io/)** – Automates TLS certificate management
- **MariaDB Galera Cluster** – Highly available MySQL-compatible database setup for Kubernetes

## 🧩 Purpose

This stack is designed for:

- **Production environments** where operational excellence, security, and high availability are priorities
- **Local environments** where test and playing with the tool is required
- **Reference architecture** for DevOps engineers building secure cloud-native platforms

## 🚀 Usage

> *Each component has its own folder with deployment manifests and instructions.*
