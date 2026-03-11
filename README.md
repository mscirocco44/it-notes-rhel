# it-notes-rhel
IT Notes Tailored Around RHEL



Study prompt template:

Create a comprehensive RHEL 9 study note as a .md file for GitHub on the topic: [TOPIC]

Structure it with the following sections:

---

# [TOPIC]

## Overview
- What it is and why it matters in a DevOps/Platform Engineering context
- Where it fits in the Linux/infrastructure ecosystem

## How It Works
- Core concepts and architecture
- Key components and how they interact
- Important terminology

## Setup & Installation
- Prerequisites
- Installation steps on RHEL 9 (use `dnf` where applicable)
- Verification steps

## Configuration
- Main config file(s) and their locations
- Key configuration options explained
- Example config snippets with comments

## Common Commands & Usage
- Day-to-day commands with explanations
- One-off / ad-hoc usage examples
- Useful flags and options

## Making It Persistent
- systemd unit setup (enable/start/status)
- Boot-time configuration
- Any relevant `systemctl` or `firewalld` rules needed

## Troubleshooting
- Common issues and fixes
- Useful log locations (`journalctl`, `/var/log/`, etc.)
- Diagnostic commands

## Practice Labs
3–5 hands-on lab exercises, each with:
- Objective
- Step-by-step instructions
- Expected outcome

## Practice Questions
10 questions mixing conceptual and command-based, ranging easy → hard.
Include a collapsible `<details>` solution block for each answer.

## References
- Relevant `man` pages
- Official Red Hat documentation links

---

Use RHEL 9 specifically (dnf, not apt; firewalld, not ufw; nmcli/NetworkManager, not netplan; SELinux context; systemd-first). Format cleanly for GitHub markdown rendering. Use fenced code blocks with syntax highlighting. Keep commands copy-pasteable.