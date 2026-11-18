# Zero-Cost Self-Hosted n8n + Local AI Agent Infrastructure
## [n8n Self Hosting Guide](n8n-self-hosting-guide.md)
## Overview &amp; Purpose

This repository provides an end-to-end, zero-cost guide to self-hosting a modular **n8n workflow automation platform** integrated with **Ollama** for local AI inference. Built using **Docker Compose**, it incurs zero subscription fees (only local hardware electricity) and provides a clear path from a Windows WSL2 Proof of Concept (PoC) to a permanent edge deployment on a Raspberry Pi 5 or home server.

>[!Important]
>This guide is mainly for personal use and likely is missing information for standard practice. If you want to add contributions to this project, please follow standard contribution guidelines in [Contributing](CONTRIBUTING.md).
>
>Example contributions:
>- Review/edit.
>- Add clarification to a section
>- Update a section
>- Add a new portion to the guide
>- etc...

## Key Architecture &amp; Features

* **Zero-Cost Local AI Stack**: Runs n8n, PostgreSQL, and Ollama (`llama3.2`) in isolated Docker containers with zero external API dependencies.
* **Internal Container Networking**: Eliminates `ECONNREFUSED` errors by connecting n8n directly to Ollama via internal container aliases (`http://ollama:11434`).
* **Secure Zero-Trust Ingress**: Integrates Cloudflare Tunnels (`cloudflared`) to expose webhooks over HTTPS without port forwarding or exposing local IP addresses.
* **Complete PoC Tutorial**: Step-by-step instructions to build a local **Email Inbox Analysis Agent** that parses IMAP feeds, categorizes messages, and generates draft replies.
* **Edge &amp; Hardware Migration**: Production tuning for ARM64 / Raspberry Pi 5 (kernel cgroup memory fixes, NVMe/SSD storage optimization, and MQTT/WebSocket hardware trigger hooks).

## How to Use This Repo

Use this blueprint as a step-by-step reference or automation template whenever you need to deploy, rebuild, or migrate your local AI agent network across Windows, macOS, or Linux/ARM devices.
