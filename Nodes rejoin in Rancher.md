Absolutely — here is a **generalized, clean, and professional README.md** that covers **both master and worker node re‑add procedures** for any RKE2 cluster managed via Rancher.

***

# Re‑Adding a Server (Master or Worker) to an RKE2 Cluster Managed by Rancher

This document provides a **general procedure** to safely remove and re‑add **any node**—master (control-plane + etcd) or worker—into an RKE2 cluster using Rancher.

***

## **1. Cordon and Drain the Node**

Before removing a node, it must be gracefully removed from the cluster.

1.  Log in to **any active master node** (commonly master‑1).
2.  Open **K9s**.
3.  Select the node you want to re‑add.
4.  Press:
    *   `c` → **Cordon**
    *   `r` → **Drain**

This ensures workloads safely move to other nodes.

***

## **2. Delete the Node From the Cluster**

Once drained:

### Using K9s

*   Press `Ctrl + d` to delete the node.

### Verify in Rancher UI

1.  Open Rancher UI.
2.  Check the cluster.
3.  If the node still appears, delete it from Rancher UI.
4.  If it doesn't disappear:
    *   Click the **Cluster Name**
    *   Delete from the **Nodes** view again.

***

## **3. Cleanup Steps on the Node**

Log in directly to the server you want to re‑add (master or worker) and run:

```bash
./usr/local/bin/rke2-killall.sh
./usr/local/bin/rke2-uninstall.sh

systemctl stop rancher-system-agent.service
systemctl disable rancher-system-agent.service

rm -f /etc/systemd/system/rancher-system-agent.service
rm -f /etc/systemd/system/rancher-system-agent.env

systemctl daemon-reload

rm -f /usr/local/bin/rancher-system-agent
rm -rf /etc/rancher/
rm -rf /var/lib/rancher/
rm -rf /usr/local/bin/rke2*
```

This ensures the node is **clean** and ready for fresh registration.

***

## **4. Re‑Registration from Rancher UI**

### Steps:

1.  Open Rancher UI.
2.  Click the **Cluster Name**.
3.  Go to the **Registration** tab.

### Choose the correct role:

*   For **Master Node** → Select:
    *   `etcd`
    *   `control-plane`

*   For **Worker Node** → Select only:
    *   `worker`

### Copy the registration command provided by Rancher.

***

## **5. Run Registration Command on the Node**

Paste and execute the copied command on the server you are re‑adding.

This will:

*   Install the required RKE2 components
*   Reconnect the node to Rancher
*   Begin provisioning

***

## **6. Verification**

Return to **Rancher UI**:

*   The node should appear as **Provisioning**
*   Then **Active**
*   For master nodes: etcd, control-plane roles should become healthy
*   For worker nodes: it should join Ready state and start receiving workloads

***

## ✔️ Summary Table

| Node Type  | Required Rancher Registration Selection | Expected Result                                   |
| ---------- | --------------------------------------- | ------------------------------------------------- |
| **Master** | etcd + control-plane                    | Joins as control-plane node, participates in etcd |
| **Worker** | worker only                             | Joins cluster workload pool                       |

***

