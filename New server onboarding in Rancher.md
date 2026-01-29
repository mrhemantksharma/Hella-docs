Got it 👍  
I’ve **cleaned up the format**, fixed **numbering**, **spacing**, **image placement**, and made it **GitHub / Confluence friendly** while keeping **your content exactly the same**.

Below is the **corrected and polished version** 👇

---

# Rancher Node Onboarding & Certificate Trust Setup

This document explains how to **onboard a new server as a Master or Worker node in Rancher** and how to **trust the Rancher TLS certificate** on both **Linux** and **Windows** nodes.

---

## 1. Verify Rancher Certificate (UI)

Before onboarding any node, first verify that the Rancher URL is using a **valid and trusted certificate**.

### Steps

1. Open the Rancher URL in a browser:
   ```
   https://rancher.hella.com/
   ```

2. Click the 🔒 **lock icon** in the address bar.

   <img width="608" height="619" alt="Lock icon – connection details" src="https://github.com/user-attachments/assets/7c3fd54f-12b8-43a4-b14d-67763bab64e0" />

3. Confirm **Connection is secure**.

   <img width="556" height="323" alt="Connection is secure" src="https://github.com/user-attachments/assets/90a7e968-dc29-4f4f-a169-a3592118fc84" />

4. Open **Certificate details** and verify the certificate chain and CN.

   <img width="811" height="857" alt="Certificate hierarchy" src="https://github.com/user-attachments/assets/79c3c32f-6a4d-43c6-9025-b054063197dc" />


5. Click **Export**.

6. Save the certificate  
   *(Base-64 / PEM format recommended)*.

You may name the file:

```
rancher.crt
```

---

## 2. Linux Node Setup (RKE2 / K3s)

### 2.1 Copy Certificate to New Server

You can copy the certificate using **SCP** or **manual copy-paste**.

#### Option A: SCP

```bash
scp rancher.crt root@<NEW_SERVER_IP>:/tmp/
```

#### Option B: Manual Copy

Copy the certificate content and create the file manually:

```bash
vi rancher.crt
```

---

### 2.2 Install Certificate into Trusted Store

Move the certificate to the system trust anchors:

```bash
mv rancher.crt /etc/pki/ca-trust/source/anchors/
```

Update the CA trust:

```bash
update-ca-trust extract
```

Restart the RKE2 agent:

```bash
systemctl restart rke2-agent
```

> ✅ This ensures the node trusts `https://rancher.hella.com`

---

## 3. Register the Node from Rancher UI

### Steps

1. Open **Rancher UI**
2. Select your **Cluster**
3. Go to the **Registration** tab
4. Choose the node role

#### Master Node

- ✅ etcd  
- ✅ control-plane  

#### Worker Node

- ✅ worker  

5. Copy the **registration command** shown
6. Run the command on the target server

---

## 4. Windows Node Setup

### 4.1 Import Rancher Certificate (Windows)

1. Copy `rancher.crt` to the Windows server
2. Open **Run** → `mmc`
3. Add the **Certificates** snap-in
4. Choose **Computer account** → **Local computer**
5. Import the certificate into:

```
Trusted Root Certification Authorities
```

> ✅ Ensure the certificate chain shows **Trusted**

---

### 4.2 Register Windows Node

1. In Rancher UI → Cluster → **Registration**
2. Select **Worker** role
3. Copy the **Windows registration command**
4. Run it in **PowerShell (Admin)** on the Windows node

---

## 5. Validation

After registration, verify:

- Node appears as **Active** in Rancher UI
- No TLS / x509 errors in logs
- Node status is **Ready**

---

## 6. Common Issues & Fixes

| Issue                                        | Cause                      | Fix                              |
|---------------------------------------------|----------------------------|----------------------------------|
| x509 certificate signed by unknown authority | Certificate not trusted    | Re-import cert + update-ca-trust |
| Node stuck in Registering                    | Agent not restarted        | Restart rke2-agent               |
| Windows node not connecting                  | Cert missing in Root store | Import cert correctly            |

---

## 7. Summary

✔ Verified Rancher TLS certificate  
✔ Installed certificate on Linux & Windows nodes  
✔ Registered nodes as Master or Worker  
✔ Ensured secure Rancher communication  

---

If you want next:
- 📘 **Confluence version with panels & callouts**
- 🧾 **README.md strictly optimized for GitHub**
- 🧩 **Architecture / flow diagram**
- 🔁 **Same doc for air-gapped environment**

Just tell me 💪
