Absolutely, Hemant! Here’s a **final `README.md`** that includes **both manual cleanup steps** and an **automation script** for the node cleanup, while keeping the full master/worker re‑add procedure.

You can copy‑paste this directly into your repo as `README.md`.

***

# Re‑Adding a Server (Master or Worker) to an RKE2 Cluster via Rancher

This guide explains how to safely remove and re‑add **any node**—**master** (control‑plane + etcd) or **worker**—to an **RKE2** cluster managed by **Rancher**. It includes both **manual cleanup steps** and an **automation script** you can run on the node.

***

## Prerequisites

*   Access to a healthy master node (to cordon/drain via K9s).
*   Access to **Rancher UI**.
*   SSH access to the target server (master or worker) being re‑added.
*   `sudo` privileges on the target server.

***

## 1) Gracefully Remove the Node (Cordon & Drain)

1.  Log in to **any healthy master node** (e.g., `master-1`).
2.  Open **K9s**.
3.  Select the **target node** you want to re‑add.
4.  Press:
    *   `c` → **Cordon** (prevent new pods from scheduling)
    *   `r` → **Drain** (evict workloads safely)

***

## 2) Delete the Node From the Cluster

### Using K9s

*   Press `Ctrl + d` on the selected node to delete it.

### Verify in Rancher UI

1.  Open **Rancher UI** → your **Cluster**.
2.  Confirm the node is removed.
3.  If still visible:
    *   Go to **Cluster → Nodes** view and delete it again.

***

## 3) Cleanup on the Target Server (Manual & Automated)

> **Run these on the server you want to re‑add** (master or worker).

### Option A — **Automated Cleanup Script (Recommended)**

Create the script `rke2_node_cleanup.sh` on the target server:

```bash
#!/bin/bash

# ---------------------------------------
# RKE2 Cleanup Script
# Removes all RKE2 + Rancher System Agent components
# ---------------------------------------

RED="\033[0;31m"
GREEN="\033[0;32m"
YELLOW="\033[1;33m"
NC="\033[0m" # No Color

echo -e "${YELLOW}=== RKE2 Node Cleanup Script ===${NC}"

# Confirm action
read -p "This will REMOVE RKE2 and Rancher agent from this server. Continue? (y/n): " choice
if [[ "$choice" != "y" ]]; then
    echo -e "${RED}Aborted.${NC}"
    exit 1
fi

echo -e "${GREEN}Stopping and uninstalling RKE2...${NC}"

# Run kill and uninstall if scripts exist
if [[ -f /usr/local/bin/rke2-killall.sh ]]; then
    /usr/local/bin/rke2-killall.sh
fi

if [[ -f /usr/local/bin/rke2-uninstall.sh ]]; then
    /usr/local/bin/rke2-uninstall.sh
fi

echo -e "${GREEN}Stopping Rancher System Agent...${NC}"

# Stop rancher-system-agent
systemctl stop rancher-system-agent.service 2>/dev/null
systemctl disable rancher-system-agent.service 2>/dev/null

# Remove service files
rm -f /etc/systemd/system/rancher-system-agent.service
rm -f /etc/systemd/system/rancher-system-agent.env

# Reload systemd
systemctl daemon-reload

echo -e "${GREEN}Removing Rancher System Agent binary...${NC}"
rm -f /usr/local/bin/rancher-system-agent

echo -e "${GREEN}Removing Rancher and RKE2 directories...${NC}"
rm -rf /etc/rancher/
rm -rf /var/lib/rancher/
rm -rf /usr/local/bin/rke2*

echo -e "${GREEN}Cleanup Completed Successfully!${NC}"
echo -e "${YELLOW}You can now re-register this node using Rancher UI.${NC}"
```

**Save & run:**

```bash
# Save the script
sudo tee /usr/local/sbin/rke2_node_cleanup.sh >/dev/null <<'EOF'
# (paste the script content above)
EOF

# Make it executable
sudo chmod +x /usr/local/sbin/rke2_node_cleanup.sh

# Run (interactive)
sudo /usr/local/sbin/rke2_node_cleanup.sh

# Or non-interactive
sudo /usr/local/sbin/rke2_node_cleanup.sh -y
```

***

### Option B — **Manual Cleanup (Exact commands)**

If you prefer to run commands manually, execute:

```bash
# Kill & uninstall RKE2
./usr/local/bin/rke2-killall.sh
./usr/local/bin/rke2-uninstall.sh

# Stop/disable Rancher System Agent
systemctl stop rancher-system-agent.service
systemctl disable rancher-system-agent.service

# Remove service files
rm -f /etc/systemd/system/rancher-system-agent.service
rm -f /etc/systemd/system/rancher-system-agent.env

# Reload systemd
systemctl daemon-reload

# Remove agent binary
rm -f /usr/local/bin/rancher-system-agent

# Remove Rancher & RKE2 data/binaries
rm -rf /etc/rancher/
rm -rf /var/lib/rancher/
rm -rf /usr/local/bin/rke2*
```

> **Tip:** On some systems the scripts are at `/usr/local/bin/...` (absolute path). If the relative `./usr/local/bin/...` doesn’t exist, use the absolute path.

***

## 4) Re‑Register the Node from Rancher UI

1.  Open **Rancher UI** → click your **Cluster** → **Registration** tab.
2.  Choose the appropriate role:
    *   **Master**: select ✅ **etcd** and ✅ **control‑plane**
    *   **Worker**: select ✅ **worker** only
3.  **Copy** the registration command shown.
4.  **Run** it on the target server you just cleaned.

***

## 5) Verify

*   Watch **Rancher UI**: node should move **Provisioning → Active/Ready**.
*   For **masters**, ensure **etcd** and **control‑plane** show **Healthy**.
*   For **workers**, node should be **Ready** and start taking workloads.

***

## Troubleshooting (Optional)

*   If the node **still appears** in Rancher after deletion, delete from **Cluster → Nodes** and wait a minute.
*   If the registration fails, check:
    *   `/var/lib/rancher/rke2/agent/logs` and `/var/log/syslog`
    *   Firewall/ports (9345 for RKE2 server, 6443 for kube‑api)
    *   Time sync (`chrony`/`ntp`) across cluster nodes

***

## Summary Table

| Node Type  | Registration selection (Rancher) | Expected result                              |
| ---------- | -------------------------------- | -------------------------------------------- |
| **Master** | etcd + control‑plane             | Joins as control‑plane; participates in etcd |
| **Worker** | worker                           | Joins as workload node                       |

***
