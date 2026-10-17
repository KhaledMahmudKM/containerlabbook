# Concise Book on ContainerLab

A practical, hands-on guide to building network labs with containers.

<img src="cover.svg" alt="Cover Image" width="100%" >

---

## Overview

ContainerLab is a tool for building network topologies from containers. Instead of spinning up heavy virtual machines in GNS3 or EVE-NG, you describe your lab in a short YAML file, run one command, and ContainerLab wires everything together in seconds.

This book is a task-focused introduction to ContainerLab for students and practitioners. It favors runnable examples over theory: every lab in these pages uses **free, downloadable images** — FRR for routing and plain Linux containers for hosts — so you can deploy each one without licenses or vendor accounts.

---

## What This Book Covers

- What ContainerLab is and when to use it
- Installing ContainerLab and its Docker prerequisite
- The topology file: nodes, kinds, links, and the management network
- Deploying, inspecting, and destroying labs
- Running FRR routers and Linux hosts as containers
- Wiring interfaces and building multi-node topologies
- Pre-loading device configuration at startup
- Accessing and operating nodes (vtysh, shells, logs)
- Building and verifying a leaf–spine (CLOS) fabric
- Automating labs, drawing topology graphs, and lab-as-code workflows
- Troubleshooting and day-to-day best practices

---

## Who This Book Is For

- Students in networking, cybersecurity, cloud, and DevOps
- Network engineers who want fast, disposable labs
- Anyone moving from GNS3/EVE-NG to a lighter, file-driven workflow

You should be comfortable with the Linux command line and basic networking (IP addressing, routing, interfaces). No prior container experience is required — the essentials are covered along the way.

---

## How to Read It

The chapters build on each other, so first-time readers will get the most from reading Part I in order before experimenting. If you already run ContainerLab, jump straight to the topic you need from the sidebar — each chapter stands on its own, and the labs are self-contained.

> **Lab convention.** Topology files in this book are named `something.clab.yml`. Commands are shown for a Linux host with Docker installed. Where a command needs elevated privileges it is shown with `sudo`.

---

## Notes

This is a living reference. ContainerLab is developed actively, so command output and image tags may shift between releases; always check the official documentation at **containerlab.dev** for the current details.
