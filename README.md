# EC2_Volume-Instance_DR_Recovery
File corruption disaster recovery
# EC2 Sudoers File Corruption Recovery Simulation

## Overview

This document provides a step-by-step simulation for recovering an AWS EC2 Ubuntu 24.04 instance (`prod`) when the `/etc/sudoers` file is corrupted, causing loss of `sudo` access. The recovery process involves detaching the root volume from the affected instance, attaching it to a healthy instance (`dev`), mounting it, repairing the file, re-attaching it back, and verifying login/sudo access.

**Use Case**: Kernel patch failures, sudoers corruption, or any boot-level file corruption that requires offline repair.

---

## Architecture

| Instance | Role | Purpose |
|----------|------|---------|
| `prod`   | Target | Instance with corrupted `/etc/sudoers` |
| `dev`    | Rescue | Healthy instance used to mount and repair prod's volume |

> ⚠️ **IMPORTANT**: Both instances **MUST** be in the **same Availability Zone (AZ)** for direct volume attachment. See the [Alternative Solution](#alternative-solution-different-az) if they are not.

---

## Prerequisites

- Two AWS EC2 Ubuntu 24.04 instances tagged `dev` and `prod`
- Both instances in the **same Availability Zone**
- SSH access to both instances
- AWS Console access (or AWS CLI configured)
- **Recommended**: Take a snapshot of the prod root volume before starting

---

## Step 1: Manually Corrupt the sudoers File on `prod` (Simulation)

SSH into the **prod** instance and intentionally break the sudoers file to simulate the failure.

```bash
# SSH into prod
ssh ubuntu@<prod-public-ip>

# Corrupt the sudoers file by appending invalid syntax
echo "broken syntax here" | sudo tee -a /etc/sudoers
