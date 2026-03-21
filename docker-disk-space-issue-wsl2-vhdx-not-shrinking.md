
# Docker Disk Space Issue (WSL2 VHDX Not Shrinking)


* [Overview](#overview)
* [Identifying the Root Cause](#identifying-the-root-cause)
* [Root Cause](#root-cause)
* [Resolution Steps (Low Risk Approach)](#resolution-steps-low-risk-approach)
* [Preventive Practices](#preventive-practices)

## Overview

During the investigation of C drive space exhaustion, an additional storage issue was identified related to Docker running on WSL2.


## Identifying the Root Cause

Using disk analysis tools such as:

* TreeSize
* WinDirStat


I found Large disk usage under the following directory:

```
C:\Users\<User>\AppData\Local\Docker\wsl\disk\docker_data
```
Even after removing images and containers, disk space on Windows did not decrease.

## Root Cause

When using Docker with WSL2:

* All images, containers, volumes, and build cache are stored inside a **VHDX virtual disk file**
* Deleting Docker resources frees space *inside* the VHDX
* But the VHDX file size itself does **not shrink automatically**
* Windows does not reclaim that free space unless:

  * WSL is shut down
  * The VHDX file is optimized

Frequent use of:

```bash
docker compose -f docker-compose.yml -f docker-compose.dev.yml up -d --build
```

caused:

* Build cache accumulation
* Image layer duplication
* Gradual VHDX growth


## Resolution Steps (Low Risk Approach)

### 1️⃣ Clean Docker Resources

```bash
docker system prune --volumes
docker builder prune --all --force
```

This removes:

* Stopped containers
* Unused images
* Build cache
* Unattached volumes

⚠️ Active containers and images are not removed.

---

### 2️⃣ Shutdown WSL

```bash
wsl --shutdown
```

This step is critical for Windows to recognize freed space.

---

### 3️⃣ Compact / Optimize the VHDX (Critical Step)

Run PowerShell as Administrator:

```powershell
Optimize-VHD -Path "C:\Users\<User>\AppData\Local\Docker\wsl\disk\docker_data.vhdx" -Mode Full
```

✔️ This is the **key step** that actually reduces the file size on disk.


### 4️⃣ Restart Docker Desktop

After optimization, restart Docker Desktop (or Windows if needed).

Windows should now reflect the reduced disk usage.

## Preventive Practices

* Run `docker system prune` periodically
* Avoid excessive `--build` unless necessary
* Monitor Docker disk usage
* Consider moving Docker disk to another drive in heavy development environments

