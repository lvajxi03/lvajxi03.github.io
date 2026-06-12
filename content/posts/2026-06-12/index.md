---
title: "2026-06-12"
description: "My Home Lab"
date: 2026-06-12T19:15:00+02:00
# image: "cover.jpg"          # Nazwa pliku graficznego z tego samego folderu
math: false
license: "CC BY-NC-SA 4.0"
hidden: false
comments: true
draft: false
tags:
    - Hardware
    - Kubernetes
categories:
    - HomeLab
---

# Home Kubernetes Lab

I've always been a fan of ARM-based single-board computers (SBCs). My journey started with a Raspberry Pi 4, which served as my primary workstation for nearly two years, followed by an Odroid N2. These are simple, reliable devices that are surprisingly versatile—even capable retro-gaming machines.

While the Raspberry Pi 4 has since been replaced by a Raspberry Pi 5, the Odroid N2 remains in active service. It hosts several lightweight databases and web applications and continues to perform flawlessly.

Over time, my collection has expanded to include:

* **Odroid M2** — the next generation of the Odroid product line,
* **Orange Pi 5 Plus**,
* **Raspberry Pi 500** — purchased as an impulse bargain.

## Hardware Overview

| Hostname         | palladium                     | vanadium         | iridium                       | ruthenium                     | tantalum         |
| ---------------- | ----------------------------- | ---------------- | ----------------------------- | ----------------------------- | ---------------- |
| SBC Model        | ODROID N2                     | Raspberry Pi 5   | ODROID M2                     | Orange Pi 5 Plus              | Raspberry Pi 500 |
| SoC              | Amlogic S922X                 | Broadcom BCM2712 | Rockchip RK3588S2             | Rockchip RK3588               | Broadcom BCM2712 |
| CPU Architecture | 4× Cortex-A73 + 2× Cortex-A53 | 4× Cortex-A76    | 4× Cortex-A76 + 4× Cortex-A55 | 4× Cortex-A76 + 4× Cortex-A55 | 4× Cortex-A76    |
| Memory           | 4 GB                          | 8 GB             | 16 GB                         | 16 GB                         | 16 GB            |
| Storage Type     | microSD                       | NVMe             | NVMe + eMMC                   | NVMe                          | NVMe             |
| Storage Capacity | 128 GB                        | 512 GB           | 512 GB + 64 GB                | 512 GB                        | 256 GB           |
| Operating System | Armbian                       | Ubuntu Server    | Armbian                       | Armbian                       | Ubuntu Server    |

## Retired Systems

| Hostname         | ferrum                  | thorium                 |
| ---------------- | ----------------------- | ----------------------- |
| SBC Model        | Radxa Rock Pi S         | FriendlyElec NanoPi Neo |
| SoC              | Rockchip RK3308         | Allwinner H3            |
| Architecture     | 64-bit (ARMv8-A)        | 32-bit (ARMv7-A)        |
| CPU              | 4× Cortex-A35 @ 1.3 GHz | 4× Cortex-A7 @ 1.2 GHz  |
| Memory           | 512 MB DDR3             | 512 MB DDR3             |
| Storage Type     | microSD                 | microSD                 |
| Storage Capacity | 64 GB                   | 64 GB                   |
| Operating System | DietPi                  | DietPi                  |

## Objectives

The goal of this project is to build a robust Kubernetes cluster capable of hosting both personal and widely used applications and databases. The cluster should also include a complete platform engineering stack, featuring:

* comprehensive observability,
* CI/CD pipelines,
* GitOps-driven deployments.

## Proposed Node Roles

* **iridium** — Kubernetes control plane,
* **palladium** — lightweight worker node and test environment,
* **vanadium** — CI/CD workloads,
* **tantalum** — observability stack,
* **ruthenium** — heavy worker node for resource-intensive web applications.

## Tooling

* **k3s** — lightweight Kubernetes distribution,
* **Ansible** — infrastructure automation and configuration management.
