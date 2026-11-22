<div align="center">

# 🛡️ EC2 Sudoers Corruption Recovery Runbook

### Disaster Recovery Simulation for AWS EC2 Ubuntu 24.04

[![AWS](https://img.shields.io/badge/AWS-EC2-FF9900?style=for-the-badge&logo=amazonaws&logoColor=white)](https://aws.amazon.com/ec2/)
[![Ubuntu](https://img.shields.io/badge/Ubuntu-24.04_LTS-E95420?style=for-the-badge&logo=ubuntu&logoColor=white)](https://ubuntu.com/)
[![Shell](https://img.shields.io/badge/Shell-Bash-4EAA25?style=for-the-badge&logo=gnubash&logoColor=white)](https://www.gnu.org/software/bash/)
[![Status](https://img.shields.io/badge/Status-Verified-brightgreen?style=for-the-badge)](#)
[![License](https://img.shields.io/badge/License-MIT-blue?style=for-the-badge)](#)

**A battle-tested, production-grade recovery workflow for when `sudo` breaks and you can't fix it from inside.**

[📖 Overview](#-overview) • [🏗️ Architecture](#️-architecture) • [🚀 Steps](#-step-1-manually-corrupt-the-sudoers-file-on-prod-simulation) • [🔧 Troubleshooting](#-troubleshooting) • [⚡ Quick Reference](#-quick-command-reference)

</div>

---

## 📖 Overview

> **Scenario**: A kernel patch, bad config push, or fat-finger edit corrupts `/etc/sudoers` on a production EC2 instance. `sudo` no longer works. You are locked out of your own machine.

This runbook walks through the **complete offline recovery workflow**:

```text
┌──────────────────────────────────────────────────────────────────────────┐
│                                                                          │
│   🔴 BROKEN PROD          🔧 RESCUE DEV           ✅ HEALTHY PROD        │
│                                                                          │
│   ┌──────────┐            ┌──────────┐            ┌──────────┐           │
│   │  prod    │            │   dev    │            │  prod    │           │
│   │  💥 sudo │  ──────►   │  🩺 fix  │  ──────►   │  ✨ sudo │           │
│   │  broken  │  detach/   │  mounted │  re-attach │  works  │           │
│   └──────────┘  attach    └──────────┘            └──────────┘           │
│                                                                          │
│     Step 1-2               Step 3-4                 Step 5-6             │
│                                                                          │
└──────────────────────────────────────────────────────────────────────────┘
```

**Use Case**: Kernel patch failures, sudoers corruption, `/etc/fstab` damage, GRUB config issues — any boot-critical file corruption that requires offline repair.

---

## 🏗️ Architecture

| 🖥️ Instance | 🎭 Role | 🎯 Purpose |
|:-----------:|:-------:|:-----------|
| **`prod`** | 🎯 Target | Instance with corrupted `/etc/sudoers` |
| **`dev`**  | 🩺 Rescue | Healthy instance used to mount and repair prod's volume |

> ⚠️ **CRITICAL**: Both instances **MUST** be in the **same Availability Zone (AZ)** for direct volume attachment. See [Alternative Solution](#-alternative-solution-different-az) if not.

<details>
<summary><b>📋 Prerequisites (click to expand)</b></summary>

- ✅ Two AWS EC2 Ubuntu 24.04 instances tagged `dev` and `prod`
- ✅ Both instances in the **same Availability Zone**
- ✅ SSH access to both instances
- ✅ AWS Console access (or AWS CLI configured)
- ✅ **Recommended**: Snapshot of prod root volume taken before starting

</details>

---

## 🧭 Recovery Flow

```text
  ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐   ┌─────────┐
  │  1️⃣   │──►│  2️⃣   │──►│  3️⃣   │──►│  4️⃣   │──►│  5️⃣   │──►│  6️⃣   │
  │ Corrupt │   │  Stop & │   │ Attach  │   │  Fix    │   │Re-attach│   │ Verify  │
  │  prod   │   │ Detach  │   │ to dev  │   │ sudoers │   │ to prod │   │  login  │
  └─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘   └─────────┘
```

---

## 1️⃣ Step 1: Manually Corrupt the sudoers File on `prod` (Simulation)

SSH into the **prod** instance and intentionally break the sudoers file to simulate the failure.

```bash
# SSH into prod
ssh ubuntu@<prod-public-ip>

# Corrupt the sudoers file by appending invalid syntax
echo "broken syntax here" | sudo tee -a /etc/sudoers
```

<details>
<summary><b>🔍 Verify the Failure (click to expand)</b></summary>

```bash
sudo ls
```

**Expected Error Output:**
```
sudo: /etc/sudoers is world writable
sudo: no valid sudoers sources found, quitting
sudo: error initializing audit plugin sudoers_audit
```

At this point, `prod` can no longer self-repair because `sudo` is broken.

</details>

---

## 2️⃣ Step 2: Stop `prod` and Detach the Root Volume

### 🖱️ Via AWS Console

1. **Stop the instance**:
   - EC2 → Instances → Select **prod** → **Instance State** → **Stop Instance**
   - Wait until state = `stopped`

2. **Note the Volume ID**:
   - Go to prod instance → **Storage** tab → Copy the **Volume ID** of the root device (e.g., `vol-0abc123...`)

3. **Detach the volume**:
   - EC2 → Volumes → Select the volume → **Actions** → **Detach Volume** → Confirm
   - Wait until state = `available`

### ⌨️ Via AWS CLI

```bash
# Stop the prod instance
aws ec2 stop-instances --instance-ids <prod-instance-id>

# Wait for stopped state
aws ec2 wait instance-stopped --instance-ids <prod-instance-id>

# Get root volume ID
aws ec2 describe-instances \
  --instance-ids <prod-instance-id> \
  --query "Reservations[0].Instances[0].BlockDeviceMappings[0].Ebs.VolumeId" \
  --output text

# Detach the volume
aws ec2 detach-volume --volume-id <volume-id>

# Wait for detachment
aws ec2 wait volume-available --volume-ids <volume-id>
```

---

## 3️⃣ Step 3: Attach Volume to `dev` and Mount It

### 🖱️ Attach via AWS Console

1. EC2 → Volumes → Select the detached volume → **Actions** → **Attach Volume**
   - **Instance**: Select **dev**
   - **Device name**: `/dev/sdf` (AWS will map to NVMe automatically)

### ⌨️ Attach via AWS CLI

```bash
aws ec2 attach-volume \
  --volume-id <volume-id> \
  --instance-id <dev-instance-id> \
  --device /dev/sdf
```

### 🖥️ SSH into `dev` and Mount

```bash
# SSH into dev
ssh ubuntu@<dev-public-ip>

# List block devices to identify the newly attached disk
lsblk
```

You will typically see a new disk like `nvme1n1` with a partition `nvme1n1p1`.

```bash
# Create mount point
sudo mkdir -p /mnt/rescue

# Mount the partition (adjust device name based on lsblk output)
sudo mount /dev/nvme1n1p1 /mnt/rescue

# Verify mount
ls /mnt/rescue/etc/sudoers
```

> 💡 **Note**: If the disk has no partition (e.g., just `nvme1n1`), mount the whole disk:
> ```bash
> sudo mount /dev/nvme1n1 /mnt/rescue
> ```

---

## 4️⃣ Step 4: Fix the sudoers File from `dev`

The prod filesystem is now mounted at `/mnt/rescue` on `dev`. You have root access here.

```bash
# Edit the sudoers file
sudo nano /mnt/rescue/etc/sudoers
```

**Remove the line** added in Step 1:
```
broken syntax here
```

<details>
<summary><b>✅ Validate Syntax (click to expand)</b></summary>

```bash
sudo visudo -c -f /mnt/rescue/etc/sudoers
```

**Expected Output:**
```
/etc/sudoers: parsed OK
```

</details>

<details>
<summary><b>🔐 Fix Permissions — CRITICAL (click to expand)</b></summary>

The sudoers file **MUST** have `0440` permissions and be owned by `root:root`.

```bash
sudo chmod 0440 /mnt/rescue/etc/sudoers
sudo chown root:root /mnt/rescue/etc/sudoers

# Verify
ls -l /mnt/rescue/etc/sudoers
```

**Expected Output:**
```
-r--r----- 1 root root ... /mnt/rescue/etc/sudoers
```

</details>

### 📤 Unmount

```bash
sudo umount /mnt/rescue

# Verify unmount
lsblk
```

---

## 5️⃣ Step 5: Detach from `dev` and Re-attach to `prod`

### 📤 Detach from `dev`

**Via Console**: EC2 → Volumes → Select volume → **Actions** → **Detach Volume**

**Via CLI**:
```bash
aws ec2 detach-volume --volume-id <volume-id>
aws ec2 wait volume-available --volume-ids <volume-id>
```

### 📥 Re-attach to `prod`

**Via Console**: EC2 → Volumes → Select volume → **Actions** → **Attach Volume**
- **Instance**: Select **prod**
- **Device name**: Use the **original root device name** (typically `/dev/sda1` or `/dev/xvda`)

**Via CLI**:
```bash
aws ec2 attach-volume \
  --volume-id <volume-id> \
  --instance-id <prod-instance-id> \
  --device /dev/sda1
```

### ▶️ Start `prod`

**Via Console**: EC2 → Instances → Select **prod** → **Instance State** → **Start Instance**

**Via CLI**:
```bash
aws ec2 start-instances --instance-ids <prod-instance-id>
aws ec2 wait instance-running --instance-ids <prod-instance-id>
```

---

## 6️⃣ Step 6: Verify Recovery

SSH into `prod` and test sudo:

```bash
ssh ubuntu@<prod-public-ip>
sudo ls
```

> ✅ **Expected**: Command executes successfully with no errors — sudo access is restored.

Optional sanity check:

```bash
sudo visudo -c
```

**Expected Output:**
```
/etc/sudoers: parsed OK
```

---

## 🔀 Alternative Solution (Different AZ)

If `dev` and `prod` are in **different Availability Zones**, direct volume attachment will fail. Use the snapshot approach:

```text
┌──────────────┐    ┌──────────────┐    ┌──────────────┐    ┌──────────────┐
│  Snapshot    │──►│  New Volume  │──►│  Repair on   │──►│  Snapshot    │
│  of prod     │    │  in dev AZ   │    │  dev         │    │  repaired    │
└──────────────┘    └──────────────┘    └──────────────┘    └──────────────┘
                                                                   │
                                                                   ▼
                                                            ┌──────────────┐
                                                            │  New Volume  │
                                                            │  in prod AZ  │
                                                            │  → attach    │
                                                            └──────────────┘
```

1. **Create snapshot** of the corrupted prod root volume:
   ```bash
   aws ec2 create-snapshot --volume-id <prod-volume-id> --description "Corrupted prod volume"
   ```

2. **Create new volume from snapshot** in `dev`'s AZ:
   ```bash
   aws ec2 create-volume \
     --snapshot-id <snapshot-id> \
     --availability-zone <dev-az> \
     --volume-type gp3
   ```

3. **Attach to `dev`**, mount, and repair (same as Steps 3–4).

4. **Create snapshot** of the repaired volume:
   ```bash
   aws ec2 create-snapshot --volume-id <repaired-volume-id> --description "Fixed prod volume"
   ```

5. **Create new volume from repaired snapshot** in `prod`'s AZ:
   ```bash
   aws ec2 create-volume \
     --snapshot-id <repaired-snapshot-id> \
     --availability-zone <prod-az> \
     --volume-type gp3
   ```

6. **Detach old volume, attach repaired volume** to `prod`, and start.

---

## ⚡ Quick Command Reference

| 🎯 Action | 💻 Command |
|-----------|-----------|
| List block devices | `lsblk` |
| Mount volume | `sudo mount /dev/nvme1n1p1 /mnt/rescue` |
| Unmount volume | `sudo umount /mnt/rescue` |
| Validate sudoers | `sudo visudo -c -f /mnt/rescue/etc/sudoers` |
| Fix permissions | `sudo chmod 0440 /mnt/rescue/etc/sudoers` |
| Fix ownership | `sudo chown root:root /mnt/rescue/etc/sudoers` |
| Verify file | `ls -l /mnt/rescue/etc/sudoers` |

---

## 🔧 Troubleshooting

| 🚨 Issue | ✅ Resolution |
|---------|--------------|
| `sudo: /etc/sudoers is world writable` | Permissions wrong. Must be `0440` with `root:root` ownership. |
| Volume won't attach — AZ mismatch | Use the snapshot method in the [Alternative Solution](#-alternative-solution-different-az). |
| Device name changed after re-attach | Ensure you use the **original root device name** (`/dev/sda1` or `/dev/xvda`). |
| Mount fails — wrong partition | Re-run `lsblk` and check if partition is `nvme1n1p1` or whole disk `nvme1n1`. |
| `visudo -c` still fails | Check for other syntax errors — edit the file line-by-line. |

---

## ⚠️ Important Reminders

> [!CAUTION]
> **Always take a snapshot** of the volume before making any changes. Never edit `/etc/sudoers` directly on a live system — always use `visudo`. Test in a non-production environment first.

- ✅ Both instances **MUST be in the same AZ** for direct attachment
- ✅ Use the **original root device name** when re-attaching to `prod`
- ✅ Verify `0440` permissions and `root:root` ownership on sudoers
- ✅ Confirm unmount on `dev` before detaching the volume

---

## 🎯 Conclusion

This runbook covers a **complete disaster recovery workflow** for a corrupted `/etc/sudoers` file on AWS EC2. The same procedure applies to **any boot-critical file corruption** — kernel patches, `/etc/fstab`, GRUB configs, SSH configs — where the instance cannot self-recover due to lost sudo or boot failures.

<div align="center">

### 🔑 Key Takeaway

```text
Detach  ➜  Attach to healthy instance  ➜  Mount  ➜  Repair  ➜  Re-attach  ➜  Verify
```

**⭐ If this runbook saved you, star the repo!**

</div>

---

<div align="center">

**Made with 🛠️ for SREs, DevOps engineers, and anyone who's ever typed `sudo` and got a heart attack.**

[![Back to Top](https://img.shields.io/badge/⬆️_Back_to_Top-blue?style=for-the-badge)](#️-ec2-sudoers-corruption-recovery-runbook)

*Last Updated: 2025*

</div>
