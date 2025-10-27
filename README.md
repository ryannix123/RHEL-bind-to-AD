# 🧩 Join RHEL to Active Directory with SSSD + Sudo Integration

## Overview

This repository contains an **Ansible playbook** that automates the process of joining a **Red Hat Enterprise Linux (RHEL)** system to a **Microsoft Active Directory (AD)** domain.

It configures **SSSD** (System Security Services Daemon) for centralized authentication, sets up **sudo access for AD groups**, and enables **automatic home directory creation** — all using Red Hat best practices.

---

## 🚀 Features

✅ Automatically joins a RHEL system to an AD domain using `realm` and `adcli`  
✅ Configures `/etc/sssd/sssd.conf` for secure, non-root, short-name logins  
✅ Enables **sudo permissions** for specific AD groups  
✅ Creates home directories automatically for AD users  
✅ Works offline (cached credentials)  
✅ Tested on RHEL 8 and RHEL 9

### ⚠️ RHEL 10 Compatibility

This playbook is compatible with **RHEL 10** (released May 20, 2025), with the following key changes to be aware of:

- **Enumeration Feature Removed**: The enumeration feature for AD and IdM was deprecated in RHEL 9 and removed in RHEL 10. This typically doesn't affect standard domain join operations.

- **Network Interface Naming**: RHEL 10 no longer supports the legacy `net.ifnames=0` kernel parameter. Predictable network interface names (e.g., `ens3`, `ens192`) are now mandatory.

- **Kernel Update**: RHEL 10 uses kernel 6.12.0 (versus 5.14 in RHEL 9), which provides improved hardware support but doesn't impact AD integration functionality.

The core components (`realm`, `adcli`, `SSSD`) remain fully supported and function as expected in RHEL 10.

---

## 🧱 Prerequisites

Before running the playbook:

1. Ensure your RHEL system can resolve and reach your AD domain controllers:
   ```bash
   ping ad.example.com
   nslookup _ldap._tcp.ad.example.com
   ```

2. Confirm that your RHEL host has network time synchronization (Kerberos requires accurate time):
   ```bash
   chronyc tracking
   ```

3. Install **Ansible** (on your control node):
   ```bash
   sudo dnf install ansible -y
   ```

4. Have an AD user account with permissions to join computers to the domain (e.g. `administrator`).

---

## 📂 Files

| File | Description |
|------|-------------|
| `join-ad-and-setup-sssd.yml` | Main playbook that joins the domain, configures SSSD, and sets sudo permissions. |
| `vault.yml` *(optional)* | Encrypted file for securely storing your AD join password via **Ansible Vault**. |

---

## ⚙️ Variables

The main variables (defined in the playbook):

```yaml
ad_domain: "ad.example.com"
ad_realm: "AD.EXAMPLE.COM"
ad_join_user: "administrator"
ad_join_password: "{{ vault_ad_join_password }}"
ad_sudo_groups:
  - "linux-admins"
  - "linux-readonly"
```

You can override these via:
- `--extra-vars` on the CLI  
- Inventory vars (`group_vars/all.yml`)  
- Ansible Vault (`vault.yml`)

---

## 🔐 Secure Password Handling

It's best practice **not to store plaintext AD passwords**.  
Instead, encrypt them with **Ansible Vault**:

```bash
ansible-vault create vault.yml
```

Inside:
```yaml
vault_ad_join_password: "YourStrongPassword!"
```

Then run with:
```bash
ansible-playbook join-ad-and-setup-sssd.yml -i inventory.yml --ask-vault-pass
```

---

## 🧭 How It Works

### Step 1 — Install Required Packages
Installs `realmd`, `sssd`, `adcli`, `oddjob`, and related dependencies.

### Step 2 — Join the AD Domain
Uses `realm join` with credentials to join `ad.example.com` and generate a `/etc/krb5.keytab`.

### Step 3 — Configure SSSD
Writes a hardened `/etc/sssd/sssd.conf` that enables:
- AD user and group lookups
- Short names (no `DOMAIN\username`)
- Cached credentials for offline login
- Automatic home directory creation
- Sudo provider via AD or local `/etc/sudoers.d`

### Step 4 — Enable and Start Services
Restarts and enables the SSSD service.

### Step 5 — Configure Sudo Access
Creates `/etc/sudoers.d/ad_groups` with rules granting sudo rights to specific AD groups.

Example output file:
```bash
# Managed by Ansible
%linux-admins ALL=(ALL) ALL
%linux-readonly ALL=(ALL) NOPASSWD: /usr/bin/journalctl, /usr/bin/systemctl status *
```

### Step 6 — Verification
Performs a test lookup of the AD user:
```bash
id administrator@ad.example.com
```

---

## 🧩 Example Run

```bash
ansible-playbook join-ad-and-setup-sssd.yml   -i inventory.yml   --ask-vault-pass
```

You can also run against localhost:
```bash
ansible-playbook join-ad-and-setup-sssd.yml -i localhost, --connection=local --ask-vault-pass
```

---

## ✅ Post-Run Verification

After the playbook completes successfully:

1. Check that the system is joined:
   ```bash
   realm list
   ```

2. Verify an AD user:
   ```bash
   id aduser
   ```

3. Confirm sudo permissions:
   ```bash
   sudo -l -U aduser
   ```

4. Test login via SSH:
   ```bash
   ssh aduser@hostname
   ```

---

## 💡 Tips & Customization

- **Restrict login to specific AD groups:**  
  Add an access filter to `/etc/sssd/sssd.conf`:
  ```ini
  ad_access_filter = (memberOf=CN=linux-users,OU=Groups,DC=ad,DC=example,DC=com)
  ```

- **Centralize sudo in AD:**  
  Configure `sudo_provider = ad` and store sudo rules in Active Directory's LDAP schema.

- **Multi-Domain Forests:**  
  Extend the playbook by adding multiple `[domain/x.y.z]` sections in `/etc/sssd/sssd.conf`.

---

## 🧾 License

This project is released under the **MIT License**.  
Feel free to modify and use in your environment.

---

## 👨‍💻 Author

**Ryan Nix**  
*Red Hat Enterprise Linux + Ansible Automation Enthusiast*

---

## 📚 References

Red Hat Customer Portal. (n.d.). *How to integrate SSSD with Active Directory*. Retrieved from https://access.redhat.com/solutions/5444941
