## 🔄 **Example Backup Workflow Using VADP**
  - Here’s a **clear example of a backup workflow** using VADP, broken into steps so you can visualize how it works in practice.
---

### 🏗 **Setup Components**

* **vCenter Server** → Manages VMs + snapshots
* **ESXi Host** → Runs the virtual machines
* **Backup Proxy (VADP Client)** → Runs backup software (e.g., Veeam, Commvault)
* **Backup Storage** → Target location (disk, dedup appliance, cloud)

---

### ⚙ **Workflow Steps**

---

### ✅ **1️⃣ Discovery Phase**

* Backup software connects to **vCenter Server** (port 443)
* Discovers VMs, datastores, configuration details

---

### ✅ **2️⃣ Snapshot Request**

* Backup software **sends API call** to vCenter → requests snapshot of the target VM
* vCenter coordinates with ESXi → creates snapshot
  → This ensures a **consistent, point-in-time copy** of the VM’s disks

---

### ✅ **3️⃣ Data Access**

* Backup proxy decides which **transport mode** to use:

  * **NBD / NBDSSL** → Reads VMDK over TCP 902
  * **SAN Mode** → Accesses VM disks directly through SAN storage
  * **HotAdd Mode** → Attaches VM disks to backup proxy (if it’s a VM)

---

### ✅ **4️⃣ Data Transfer**

* Backup proxy reads the VMDK snapshot
* Uses **Changed Block Tracking (CBT)** to detect only changed data since last backup
* Copies data to **backup storage** (local, cloud, or deduplicated target)

---

### ✅ **5️⃣ Snapshot Removal**

* Once backup completes, backup software instructs vCenter to **delete snapshot**
* Snapshot data merges back into the main disk (consolidation)

---

### ✅ **6️⃣ Catalog & Reporting**

* Backup software updates its **catalog/index**
* Generates reports, verifies integrity, updates retention policies

---

### 📊 **Example Summary Table**

| **Step**            | **Action**                                   |
| ------------------- | -------------------------------------------- |
| 1. Discover VMs     | Proxy queries vCenter                        |
| 2. Create Snapshot  | vCenter + ESXi freeze VM state               |
| 3. Choose Transport | NBD / SAN / HotAdd selected                  |
| 4. Backup Data      | Proxy reads data, stores on backup target    |
| 5. Delete Snapshot  | Proxy instructs vCenter to clean up snapshot |
| 6. Update Catalog   | Proxy logs job, verifies, updates retention  |

---

If you want, I can:
🔹 Draw a **visual diagram** of this workflow
🔹 Write a **real-life script** or CLI example using VADP-backed tools
🔹 Provide **common errors & troubleshooting tips** for each step

Alright!

Here’s an **example CLI-based backup script** using **Veeam Backup & Replication** (one of the most popular VADP-backed tools) via **PowerShell**, which is commonly used to automate VMware VM backups.

This script assumes you have Veeam installed and its PowerShell Snap-in or Veeam PowerShell module loaded.

---

### 💻 **Real-Life Script: Backing Up a VM Using VADP (Veeam Example)**

```powershell
# Load the Veeam PowerShell module
Import-Module Veeam.Backup.PowerShell

# Connect to Veeam Backup server
Connect-VBRServer -Server "your-veeam-backup-server"

# Define the VM and backup job names
$vmName = "Production-VM01"
$backupJobName = "Daily VM Backup"

# Find the VM object in Veeam’s inventory
$vm = Get-VBRViEntity -Name $vmName

if ($vm) {
    Write-Host "Found VM: $($vm.Name)"

    # Check if the backup job exists
    $job = Get-VBRJob -Name $backupJobName

    if (-not $job) {
        # Create a new backup job if it doesn't exist
        $job = Add-VBRViBackupJob -Name $backupJobName -Entity $vm -BackupRepository "Default Backup Repository"
        Write-Host "Created new backup job: $backupJobName"
    } else {
        Write-Host "Using existing backup job: $backupJobName"
    }

    # Start the backup job
    $session = Start-VBRJob -Job $job
    Write-Host "Started backup job: $($job.Name)"

    # Wait for completion
    while ($session.IsRunning) {
        Write-Host "Backup running... Status: $($session.State)"
        Start-Sleep -Seconds 10
        $session = Get-VBRBackupSession | Where-Object { $_.JobId -eq $job.Id } | Sort-Object CreationTime -Descending | Select-Object -First 1
    }

    Write-Host "Backup job completed. Final status: $($session.Result)"
} else {
    Write-Host "VM not found in Veeam inventory."
}

# Disconnect from Veeam server
Disconnect-VBRServer
```

---

### ⚙ **What This Script Does**

✅ Connects to the Veeam Backup server
✅ Searches for a specific VM (e.g., `Production-VM01`)
✅ Checks if a backup job already exists; if not, it creates one
✅ Starts the backup job (which uses VADP under the hood to interact with VMware)
✅ Monitors the backup session status and prints updates
✅ Disconnects cleanly from the Veeam server

---

### 🔒 **Where VADP Comes In**

Even though you’re scripting at the Veeam layer, behind the scenes Veeam:

* Calls VMware APIs to create VM snapshots
* Uses VADP to read VM disk data (via NBD, SAN, or HotAdd)
* Leverages CBT (Changed Block Tracking) to optimize what’s backed up

---

If you’re interested, I can also:
🔹 Write an **example script for Commvault or Rubrik CLI**
🔹 Show **API-level calls using VMware’s vSphere API directly** (if you want raw VADP-level control)
🔹 Help you build **error-handling + reporting features** into this script

---
 Here’s an **example of API-level calls** using VMware’s vSphere API directly to control snapshot operations, which underlie VADP-based backups.
 - This gives you raw control if you want to build or script your own workflows outside of commercial backup tools.

---

### 💻 **Example: Python Script Using pyVmomi (vSphere API SDK)**

This script connects to vCenter, takes a snapshot of a VM, lists snapshots, and deletes the snapshot — directly calling VMware APIs.

```python
from pyVim.connect import SmartConnect, Disconnect
from pyVmomi import vim
import ssl

# Ignore SSL warnings
s = ssl.SSLContext(ssl.PROTOCOL_TLS_CLIENT)
s.verify_mode = ssl.CERT_NONE

# Connect to vCenter
si = SmartConnect(host="vcenter.yourdomain.com", user="administrator@vsphere.local", pwd="yourpassword", sslContext=s)
content = si.RetrieveContent()

# Find VM
vm_name = "Production-VM01"
vm = None
for datacenter in content.rootFolder.childEntity:
    for vm_obj in datacenter.vmFolder.childEntity:
        if vm_obj.name == vm_name:
            vm = vm_obj
            break

if not vm:
    print(f"VM {vm_name} not found")
    Disconnect(si)
    exit()

# Create snapshot
task = vm.CreateSnapshot_Task(name="BackupSnapshot", description="Snapshot for backup", memory=False, quiesce=True)
print("Creating snapshot...")

while task.info.state not in ["success", "error"]:
    continue

if task.info.state == "success":
    print("Snapshot created successfully.")
else:
    print(f"Snapshot creation failed: {task.info.error}")

# List snapshots
def list_snapshots(snapshots):
    for snap in snapshots:
        print(f"Snapshot name: {snap.name}")
        list_snapshots(snap.childSnapshotList)

if vm.snapshot:
    list_snapshots(vm.snapshot.rootSnapshotList)

# Remove snapshot
for snap in vm.snapshot.rootSnapshotList:
    if snap.name == "BackupSnapshot":
        remove_task = snap.snapshot.RemoveSnapshot_Task(removeChildren=False)
        print("Removing snapshot...")
        while remove_task.info.state not in ["success", "error"]:
            continue
        print("Snapshot removed.")

# Disconnect
Disconnect(si)
```

---

### ⚙ **What This Does**
✅ Connects directly to vCenter using `pyVmomi`
✅ Locates the target VM
✅ Creates a quiesced snapshot (using VADP’s underlying hooks)
✅ Lists current snapshots
✅ Removes the specific backup snapshot

---

### 🔧 **Why Use This?**
- Custom integrations with non-standard backup pipelines
- Specialized snapshot control outside packaged tools
- Learning or testing raw vSphere API interactions

If you want, I can also:
🔹 Write this in **PowerCLI** (PowerShell)
🔹 Provide **REST API calls (vSphere Automation API)**
🔹 Add **logging + error handling** to the script

Want me to build one of those next? 🚀


