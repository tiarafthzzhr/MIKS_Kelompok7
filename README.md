# Wazuh SIEM Deployment + DDoS Attack Simulation

> **Tugas Kelompok — Keamanan Jaringan**  
> Institut Teknologi Sepuluh Nopember (ITS) — 2026

---

## Anggota Kelompok

| Nama | NRP | Peran |
|------|-----|-------|
| Tiara Fatimah Azzahra | 5027241090 | Manager Admin (Wazuh Manager) |
| Ananda Widi Alrafi | 5027241067 | Agent Operator 1 (vm-agent-1) |
| Az Zahrra Tasya | 5027241087 | Agent Operator 2 (vm-agent-02) |
| Ahmad Yafi | 5027241066 | Agent Operator & Dokumentasi |

---

## Deskripsi Proyek

Proyek ini mengimplementasikan **Wazuh SIEM (Security Information and Event Management)** pada infrastruktur cloud **Microsoft Azure for Students** untuk mendeteksi serangan **DDoS (Distributed Denial of Service)**.

### Tujuan
1. Deploy arsitektur Wazuh lengkap (Manager + 2 Agent) di Azure
2. Membuat skenario DDoS sebagai Proof of Concept (PoC)
3. Mendemonstrasikan kemampuan Wazuh mendeteksi anomali traffic dan generate alert
4. Mengelola logging density dan distribusi log

---

## Arsitektur Sistem

```
┌─────────────────────────────────────────────────────────┐
│                    Microsoft Azure                       │
│              Resource Group: rg-wazuh-lab                │
│           Virtual Network: vm-wazuh-manager-vnet         │
│                  Subnet: 10.0.0.0/24                     │
│                                                         │
│  ┌──────────────────────────────────────┐               │
│  │         Wazuh Manager                │               │
│  │  vm-wazuh-manager                    │               │
│  │  Public IP : 20.205.16.230           │               │
│  │  Private IP: 10.0.0.4               │               │
│  │  OS        : Ubuntu 22.04 LTS       │               │
│  │  Size      : B2als_v2 (2vCPU, 4GB)  │               │
│  │  Wazuh ver : 4.7.5                  │               │
│  │                                      │               │
│  │  wazuh-manager  (port 1514/1515)     │               │
│  │  wazuh-indexer  (port 9200)          │               │
│  │  wazuh-dashboard (port 443)          │               │
│  └──────────────┬───────────────────────┘               │
│                 │ Port 1514/1515 TCP                     │
│           ┌─────┴──────┐                               │
│           ▼            ▼                               │
│  ┌──────────────┐ ┌──────────────┐                    │
│  │  vm-agent-1  │ │ vm-agent-02  │                    │
│  │  10.0.0.5    │ │ 10.0.0.6     │                    │
│  │  B2ats_v2    │ │ B2ats_v2     │                    │
│  │  Active      │ │ Active       │                    │
│  └──────────────┘ └──────────────┘                    │
└─────────────────────────────────────────────────────────┘
```

---

## Spesifikasi Infrastruktur

| VM | Role | Size | vCPU | RAM | IP Publik | IP Private | Status |
|----|------|------|------|-----|-----------|------------|--------|
| vm-wazuh-manager | Manager + Indexer + Dashboard | B2als_v2 | 2 | 4 GiB | 20.205.16.230 | 10.0.0.4 | Running |
| vm-agent-1 | Wazuh Agent 1 (HTTP Flood target) | B2ats_v2 | 2 | 1 GiB | 57.158.24.143 | 10.0.0.5 | Active |
| vm-agent-02 | Wazuh Agent 2 (SYN Flood target) | B2ats_v2 | 2 | 1 GiB | 20.2.82.117 | 10.0.0.6 | Active |

**Platform:** Microsoft Azure for Students ($100 kredit)  
**OS:** Ubuntu Server 22.04 LTS  
**Wazuh Version:** 4.7.5  
**Region:** East Asia

---

## Deployment — Point 1

### Step 1 — Buat Infrastruktur Azure

Semua VM dibuat via Azure Portal dengan konfigurasi:
- Resource Group: `rg-wazuh-lab`
- Virtual Network: `vm-wazuh-manager-vnet` (10.0.0.0/16)
- Subnet: `default` (10.0.0.0/24)
- Region: East Asia

**Port yang dibuka di Network Security Group:**

| Port | Protocol | Tujuan |
|------|----------|--------|
| 22 | TCP | SSH akses |
| 443 | TCP | Wazuh Dashboard |
| 1514 | TCP | Agent kirim log |
| 1515 | TCP | Agent registrasi |

---

### Step 2 — Install Wazuh Manager

```bash
# SSH ke Manager
ssh azureuser@20.205.16.230

# Update sistem
sudo apt-get update && sudo apt-get upgrade -y

# Download installer
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.7/config.yml

# Edit config.yml dengan IP private Manager
nano config.yml
```

Isi `config.yml`:

```yaml
nodes:
  indexer:
    - name: node-1
      ip: "10.0.0.4"
  server:
    - name: wazuh-1
      ip: "10.0.0.4"
  dashboard:
    - name: dashboard
      ip: "10.0.0.4"
```

```bash
# Install semua komponen Wazuh
sudo bash wazuh-install.sh --generate-config-files
sudo bash wazuh-install.sh --wazuh-indexer node-1
sudo bash wazuh-install.sh --start-cluster
sudo bash wazuh-install.sh --wazuh-server wazuh-1
sudo bash wazuh-install.sh --wazuh-dashboard dashboard
```

**Hasil:** Wazuh Dashboard dapat diakses di `https://20.205.16.230`

---

### Step 3 — Install Wazuh Agent

```bash
# SSH ke Agent VM
ssh azureuser@<IP-AGENT>

# Tambah repository Wazuh
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg \
  --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg \
  --import && sudo chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] \
  https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/wazuh.list

sudo apt-get update

# Install Wazuh Agent v4.7.5 (harus sama dengan Manager)
WAZUH_MANAGER="10.0.0.4" WAZUH_AGENT_NAME="agent-01" \
  sudo apt-get install wazuh-agent=4.7.5-1 -y

# Aktifkan dan registrasi
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo /var/ossec/bin/agent-auth -m 10.0.0.4
```

**Hasil verifikasi:**

```
ID: 000, Name: vm-wazuh-manager (server), IP: 127.0.0.1, Active/Local
ID: 001, Name: vm-agent-1,  IP: any, Active
ID: 002, Name: vm-agent-02, IP: any, Active
```

**Agents Coverage: 100%**

---

## DDoS Attack Scenario — Point 2

> **Disclaimer:** Simulasi dilakukan HANYA pada VM lab milik sendiri.  
> DDoS ke sistem orang lain adalah tindak pidana (UU ITE No. 11 Tahun 2008).

### Custom DDoS Detection Rules

File: `/var/ossec/etc/rules/ddos_rules.xml`

```xml
<group name="ddos,attack,">

  <!-- HTTP Flood Detection -->
  <rule id="100200" level="12">
    <if_group>web</if_group>
    <description>HTTP Flood DDoS detected</description>
    <group>ddos,http_flood,</group>
  </rule>

  <!-- SYN Flood Detection -->
  <rule id="100201" level="14">
    <match>SYN</match>
    <description>CRITICAL: SYN Flood attack detected</description>
    <group>ddos,syn_flood,</group>
  </rule>

  <!-- Server Overload -->
  <rule id="100202" level="10">
    <match>Too many connections</match>
    <description>Server overload - possible DDoS</description>
    <group>ddos,overload,</group>
  </rule>

</group>
```

---

### Skenario 1 — HTTP Flood (vm-agent-1)

```bash
# Install tools
sudo apt install apache2-utils nginx -y
sudo systemctl start nginx

# Jalankan HTTP Flood — 50.000 request, 200 concurrent
ab -n 50000 -c 200 http://localhost/
```

**Hasil:** Traffic anomali terdeteksi, alert muncul di Dashboard
![Uploading Screenshot 2026-05-17 235606.png…]()


---

### Skenario 2 — SYN Flood (vm-agent-02)

```bash
# Install tools
sudo apt install hping3 nginx -y
sudo systemctl start nginx

# Jalankan SYN Flood
sudo hping3 -S -p 80 --flood 127.0.0.1
```

**Hasil:** SYN packet flood terdeteksi, brute force alert level 10 triggered

---

### Hasil Deteksi Wazuh (PoC)

| Metric | Nilai |
|--------|-------|
| Total Security Events | **12,406 events** |
| Authentication Failures | **11,655** |
| Level 12+ Critical Alerts | Terdeteksi |
| MITRE ATT&CK Techniques | T1110, T1110.001, T1021.004 |
| Tactic | Credential Access, Lateral Movement |

**Alert yang terdeteksi:**

| Rule ID | Level | Deskripsi |
|---------|-------|-----------|
| 5712 | 10 | sshd: brute force trying to get access |
| 5551 | 10 | PAM: Multiple failed logins in a small period of time |
| 5710 | 5 | sshd: Attempt to login using a non-existent user |
| 5760 | 5 | sshd: authentication failed |
| 5503 | 5 | PAM: User login failed |

---

### Monitor Alert Real-time

```bash
# Di Manager — monitor alert real-time
sudo tail -f /var/ossec/logs/alerts/alerts.json | python3 -c "
import sys, json
for line in sys.stdin:
  try:
    d = json.loads(line)
    lvl = d.get('rule', {}).get('level', 0)
    desc = d.get('rule', {}).get('description', '')
    agent = d.get('agent', {}).get('name', '?')
    if lvl >= 3:
      print(f'[LEVEL {lvl}] [{agent}] {desc}')
  except: pass
"
```

---

### Active Response — Auto-Block Penyerang

Konfigurasi di `/var/ossec/etc/ossec.conf`:

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>100200,100201</rules_id>
  <timeout>600</timeout>
</active-response>
```

**Cara kerja:**
1. Alert DDoS terdeteksi (rule 100200 atau 100201)
2. Manager kirim perintah ke Agent
3. Agent jalankan `iptables DROP` untuk IP penyerang
4. Setelah 600 detik (10 menit), block otomatis dilepas

---

## Logging Density & Distribution — Point 3

### Konfigurasi Logging

File: `/var/ossec/etc/ossec.conf`

```xml
<global>
  <!-- Output dalam format JSON untuk mudah diparse -->
  <jsonout_output>yes</jsonout_output>
  <alerts_log>yes</alerts_log>
  <!-- Tidak log semua event agar hemat storage -->
  <logall>no</logall>
  <logall_json>no</logall_json>
</global>

<alerts>
  <!-- Simpan ke file hanya jika level >= 3 -->
  <log_alert_level>3</log_alert_level>
  <!-- Kirim email hanya jika level >= 12 (critical) -->
  <email_alert_level>12</email_alert_level>
</alerts>
```

### Distribusi Log per Agent

| Agent | Events | Persentase |
|-------|--------|------------|
| vm-wazuh-manager | ~60% | Server utama, banyak SSH attempts |
| vm-agent-02 | ~25% | Target SYN Flood |
| vm-agent-1 | ~15% | Target HTTP Flood |
| **Total** | **12,406** | **100%** |

### Pengelolaan Log

```bash
# Cek ukuran log
du -sh /var/ossec/logs/alerts/

# Cek jumlah total events
sudo wc -l /var/ossec/logs/alerts/alerts.json

# Monitor storage real-time saat DDoS
watch -n2 "du -sh /var/ossec/logs/alerts/"
```

### Mengapa Logging Density Penting?

DDoS dapat menghasilkan **jutaan log per menit**. Tanpa pengelolaan yang baik:
- Storage server bisa penuh dalam hitungan jam
- Performance sistem menurun
- Sulit menemukan alert yang relevan

Solusi yang diterapkan:
- `logall: no` — tidak simpan semua event, hanya yang relevan
- `log_alert_level: 3` — filter event di bawah level 3
- `email_alert_level: 12` — notifikasi hanya untuk event critical
- Format JSON untuk memudahkan parsing dan analisis

---

## Hasil dan Kesimpulan

### Yang Berhasil Dicapai

1. **Wazuh SIEM berhasil di-deploy** di Azure dengan 100% agent coverage
2. **DDoS terdeteksi** dalam hitungan detik setelah serangan dimulai
3. **12,406 security events** berhasil dicatat dan dianalisis
4. **MITRE ATT&CK mapping** otomatis oleh Wazuh
5. **Active Response** siap auto-block penyerang
6. **Logging density** berhasil dikelola tanpa overflow storage

### Kesimpulan

Wazuh SIEM terbukti efektif sebagai solusi keamanan untuk mendeteksi serangan DDoS. Dengan konfigurasi yang tepat, Wazuh dapat:
- Mendeteksi anomali traffic secara real-time
- Generate alert dengan level severity yang jelas
- Memetakan serangan ke framework MITRE ATT&CK
- Merespons otomatis dengan memblokir IP penyerang
- Mengelola volume log yang besar secara efisien

---

## Referensi

- [Wazuh Official Documentation](https://documentation.wazuh.com)
- [Wazuh 4.7 Installation Guide](https://documentation.wazuh.com/4.7/installation-guide)
- [Wazuh Active Response](https://documentation.wazuh.com/current/user-manual/capabilities/active-response)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [Azure for Students](https://azure.microsoft.com/free/students)
- [UU ITE No. 11 Tahun 2008](https://jdih.kominfo.go.id)

---

*Tugas Keamanan Jaringan — Institut Teknologi Sepuluh Nopember (ITS) 2026*
