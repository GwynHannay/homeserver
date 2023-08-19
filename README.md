# homesys
*My home server!*

## Motivation

- Manage the digital lives of my household in an effective way
- Privacy and security - we own our data
- Career development - to extend my skills in data engineering

## Goals

- Replace third party apps with self-hosted, wherever possible
- Automation of household things
- Our own data platform

## Infrastructure

### Hardware

My server:
- PC: HP EliteDesk 800 G3 Micro
- RAM: 32GB
- Storage: 250GB SSD + 1TB NVMe SSD

Other devices:
- Fileserver: 4 * 4TB hard drives
- HP Prodesk: Husband's server, houses Home Assistant
- RPi: Handles media stored in DropBox

## Learnings

One of the big things to come out of this project is just how inefficient apps are these days. Emphasis for infrastructure is on "high availability", which means allocating resources for worst cases and building in massive amounts of redundancy. This does not suit my use case: I am running everything on a single HP EliteDesk PC and I am more concerned with not wasting resources than I am about making sure everything runs without interruption.
