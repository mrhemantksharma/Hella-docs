# RKE2 Windows Worker Node Recovery & Rejoin in Rancher

## Overview

This document describes the **end-to-end troubleshooting and recovery procedure** for a **Windows Server worker node running RKE2** that became **Unavailable / Unknown** in Rancher and failed to rejoin the cluster.

The issue was ultimately resolved by **restoring the server from backup and performing a clean rejoin to Rancher**. This page consolidates **all root causes, validations, failed paths, and the final working solution**, so the same issue can be resolved faster in the future.

---

## Symptoms Observed

* Node not visible or stuck in **Reconcilling / Unavailable** state in Rancher UI
* Node conditions showing:

  * `MemoryPressure = Unknown`
  * `DiskPressure = Unknown`
  * `PIDPressure = Unknown`
  * `Ready = Unknown`
* Rancher message:

  ```
  waiting for probes: kubelet
  ```
  
<img width="1592" height="360" alt="image" src="https://github.com/user-attachments/assets/143faafd-196d-48ab-90e5-77f51aa46873" />
  
* On Windows server:

  * `rke2` service stopping repeatedly
  * `rancher-wins` service failing to stay running
  * CSI Proxy (`csiproxy`) blocking file deletion
  * Node never appearing in Rancher after rejoin

---

## Environment

* OS: **Windows Server (RKE2 Worker Node)**
* Kubernetes Distribution: **RKE2**
* Cluster Management: **Rancher**
* Network: Corporate proxy + internal Rancher LB

---

## Root Causes Identified

Multiple contributing issues were identified during troubleshooting:

1. **Expired / broken node identity & certificates**
2. **Windows time drift impacting TLS**
3. **Partial / failed Rancher agent registration**
4. **WinHTTP proxy misconfiguration**
5. **CSI Proxy locking files during cleanup**

Once the node entered a **rejoin-failed state**, it could no longer recover cleanly without a reset.

---

## Key Learnings (Important)

* On **RKE2 Windows**, there is **NO separate kubelet/containerd service**
* Everything runs under the **`rke2` Windows service**
* If kubelet bootstrap fails, **RKE2 stops itself**
* Rancher will **not show the node** unless registration fully completes
* Proxy issues can silently break Rancher system-agent bootstrap

---

## Initial Diagnostics Performed

### Check RKE2 Service

```powershell
Get-Service rke2
```

### Check CSI Proxy

```powershell
Get-Service csiproxy
```

### Check DNS Resolution

```powershell
nslookup rancher.<domain>
```

### Test Connectivity to Rancher

```powershell
Test-NetConnection rancher.<domain> -Port 443
```

---

## Failed Rejoin Attempts (For Reference)

### Service Stop & Cleanup

```powershell
Stop-Service rke2 -Force
Stop-Service rancher-wins -Force
```

### Attempted Cleanup

```powershell
Remove-Item -Recurse -Force C:\etc\rancher
Remove-Item -Recurse -Force C:\var\lib\rancher\agent
```

Blocked due to:

```
csi-proxy.log is being used by another process
```

### CSI Proxy Stop

```powershell
Stop-Service csiproxy -Force
```

### Forced Process Cleanup (when required)

```powershell
taskkill /IM csi-proxy.exe /F
taskkill /IM wins.exe /F
taskkill /IM rke2.exe /F
```

---

## Proxy Issue Identified (Critical)

Repeated join failures showed:

```
Error writing proxy settings. (87) The parameter is incorrect.
```

WinHTTP proxy configuration was blocking Rancher agent bootstrap.

### Proxy Validation

```powershell
netsh winhttp show proxy
```

### DNS & Network Were Valid

```powershell
nslookup rancher.<domain>
Test-NetConnection rancher.<domain> -Port 443
```

Despite network being reachable, Rancher agent could not complete registration.

---

## Final Resolution (Successful Path)

### ✅ What Actually Worked

> **The Windows server was restored from a clean backup and rejoined to Rancher using a fresh registration command.**

This fully resolved:

* Corrupted Rancher agent state
* Broken node identity
* Proxy misconfiguration residue
* CSI Proxy file locks

---

## Final Working Procedure (Recommended for Future)

### Step 1: Restore Windows Server from Backup

* Restore to a **known-good state** (before Rancher/RKE2 corruption)
* Verify system boots cleanly

---

### Step 2: Validate Network & DNS

```powershell
nslookup rancher.<domain>
Test-NetConnection rancher.<domain> -Port 443
```

---

### Step 3: (Optional but Recommended) Reset WinHTTP Proxy

```powershell
netsh winhttp reset proxy
```

---

### Step 4: Rejoin Node via Rancher

In Rancher UI:

```
Cluster → Add Node → Windows → Worker
```

* Generate a **fresh registration command**
* Open **PowerShell as Administrator**
* Run the command completely (do not interrupt)

---

### Step 5: Verify Node Status

In Rancher UI:

* Node appears as **Registering → Active → Ready**

On Windows:

```powershell
Get-Service rke2
```

Expected:

```
Status : Running
```

Optional health check:

```powershell
curl -k https://127.0.0.1:10250/healthz
```

Expected:

```
ok
```

---

## Final Validation

```bash
kubectl get nodes
```

Node should show:

```
STATUS: Ready
```

---

## When to Use Backup Restore (Rule of Thumb)

Use **restore + rejoin** immediately if:

* RKE2 repeatedly stops on Windows
* Node does not appear after multiple rejoin attempts
* Proxy/cert errors persist
* CSI Proxy blocks cleanup repeatedly

This approach is **faster, safer, and more reliable** than prolonged manual repair.

---

## Summary

* Issue was **not Kubernetes or Rancher bug**
* Root cause was **Windows-specific agent + proxy + cert state corruption**
* Clean OS restore + fresh Rancher rejoin is the **most reliable fix**

---


