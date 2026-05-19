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

1. Deploy arsitektur Wazuh lengkap (1 Manager + 2 Agent) di Azure Student
2. Membuat skenario DDoS HTTP Flood sebagai Proof of Concept (PoC)
3. Mendemonstrasikan kemampuan Wazuh mendeteksi anomali traffic dan generate alert
4. Mengelola logging density dan distribusi log antar agent

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
│  │  (Attacker)  │ │ (Target)     │                    │
│  └──────────────┘ └──────────────┘                    │
│         │                ▲                             │
│         └── HTTP Flood ──┘                             │
│           ab -n 10000 -c 100                           │
└─────────────────────────────────────────────────────────┘
```

---

## Spesifikasi Infrastruktur

| VM | Role | Size | vCPU | RAM | IP Publik | IP Private | Status |
|----|------|------|------|-----|-----------|------------|--------|
| vm-wazuh-manager | Manager + Indexer + Dashboard | B2als_v2 | 2 | 4 GiB | 20.205.16.230 | 10.0.0.4 | Running |
| vm-agent-1 | Wazuh Agent + Attacker | B2ats_v2 | 2 | 1 GiB | 57.158.24.143 | 10.0.0.5 | Active |
| vm-agent-02 | Wazuh Agent + Target | B2ats_v2 | 2 | 1 GiB | 20.2.82.117 | 10.0.0.6 | Active |

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
| 443 | TCP | Wazuh Dashboard HTTPS |
| 1514 | TCP | Agent kirim log ke Manager |
| 1515 | TCP | Agent registrasi ke Manager |

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
# Install semua komponen Wazuh (urut, ~15 menit)
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

# Update sistem
sudo apt-get update && sudo apt-get upgrade -y

# Tambah GPG key dan repository Wazuh
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg \
  --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg \
  --import && sudo chmod 644 /usr/share/keyrings/wazuh.gpg

echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] \
  https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/wazuh.list

sudo apt-get update

# Install Wazuh Agent v4.7.5 (HARUS sama dengan Manager!)
WAZUH_MANAGER="10.0.0.4" WAZUH_AGENT_NAME="agent-01" \
  sudo apt-get install wazuh-agent=4.7.5-1 -y

# Fix konfigurasi IP Manager
sudo nano /var/ossec/etc/ossec.conf
# Cari MANAGER_IP → ganti dengan 10.0.0.4
# Simpan: Ctrl+X → Y → Enter

# Aktifkan dan registrasi ke Manager
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent
sudo /var/ossec/bin/agent-auth -m 10.0.0.4

# Tambah monitoring nginx log
sudo tee -a /var/ossec/etc/ossec.conf << 'EOF'

<ossec_config>
  <localfile>
    <log_format>apache</log_format>
    <location>/var/log/nginx/access.log</location>
  </localfile>
</ossec_config>
EOF

sudo systemctl restart wazuh-agent
```

**Verifikasi (di Manager):**

```bash
sudo /var/ossec/bin/agent_control -l

# Output:
# ID: 000, Name: vm-wazuh-manager (server), IP: 127.0.0.1, Active/Local
# ID: 001, Name: vm-agent-1,  IP: any, Active
# ID: 002, Name: vm-agent-02, IP: any, Active
```

**Agents Coverage: 100%**

---

## DDoS Attack Scenario — Point 2

> **Disclaimer:** Simulasi dilakukan HANYA pada VM lab milik sendiri di Azure.  
> DDoS ke sistem orang lain adalah tindak pidana (UU ITE No. 11 Tahun 2008).

### Skenario Serangan

Topologi serangan:

```
vm-agent-1 (10.0.0.5) ──HTTP Flood──► vm-agent-02 (10.0.0.6)
         Attacker                              Target
```

### Custom DDoS Detection Rules

File: `/var/ossec/etc/rules/ddos_rules.xml` (di Manager)

```xml
<group name="ddos,attack,">

  <rule id="100200" level="10">
    <decoded_as>web-accesslog</decoded_as>
    <match>GET|POST</match>
    <description>DDoS HTTP Flood: Abnormal web traffic detected from nginx log</description>
    <group>ddos,http_flood,</group>
  </rule>

</group>
```

```bash
# Di Manager — restart setelah tambah rule
sudo systemctl restart wazuh-manager
```

---

### Eksekusi Serangan — HTTP Flood

**Di vm-agent-1 (Attacker):**

```bash
# Install tools
sudo apt install apache2-utils nginx -y
sudo systemctl start nginx

# Jalankan HTTP Flood ke Agent-02
# -n = total request, -c = concurrent connections
ab -n 10000 -c 100 http://10.0.0.6/
```

**Di vm-agent-02 (Target) — verifikasi serangan diterima:**

```bash
sudo tail -f /var/log/nginx/access.log
# Output:
# 10.0.0.5 - - [19/May/2026] "GET / HTTP/1.0" 200 612 "-" "ApacheBench/2.3"
# 10.0.0.5 - - [19/May/2026] "GET / HTTP/1.0" 200 612 "-" "ApacheBench/2.3"
# ... terus menerus dari IP 10.0.0.5
```

---

### Hasil Deteksi Wazuh

Alert dari terminal Manager (real-time):

```json
{
  "timestamp": "2026-05-19T02:06:06",
  "rule": {
    "level": 4,
    "id": "11",
    "groups": ["stats"]
  },
  "agent": {
    "name": "vm-agent-02"
  },
  "full_log": "The average number of logs between 2:00 and 3:00 is 1228. We reached 3072.",
  "decoder": {
    "name": "web-accesslog"
  },
  "data": {
    "protocol": "GET",
    "srcip": "10.0.0.5",
    "url": "/"
  },
  "location": "/var/log/nginx/access.log"
}
```

**Analisis hasil:**

| Metric | Normal | Saat DDoS | Keterangan |
|--------|--------|-----------|------------|
| Request per jam | 1,228 | 3,072 | Naik 2.5x dari normal! |
| Source IP | Berbeda-beda | 10.0.0.5 (semua) | Single source flood |
| Request type | Normal | GET / terus-menerus | Pattern anomali |
| Alert level | - | 4-10 | Terdeteksi Wazuh |

**Dashboard Statistics:**

| Metric | Nilai |
|--------|-------|
| Total Security Events | 287 events (15 menit) |
| Authentication Failures | 232 |
| Alert Spike | Terlihat jelas di grafik 09:04-09:06 |
| MITRE ATT&CK | T1110, T1110.001, T1021.004 |

---

### Monitor Alert Real-time

```bash
# Di Manager — monitor semua alert
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

# Monitor spesifik HTTP flood dari nginx
sudo tail -f /var/ossec/logs/alerts/alerts.json | \
  grep -i "nginx\|web\|http\|ddos\|flood"
```

---

### Active Response — Auto-Block Penyerang

Konfigurasi di `/var/ossec/etc/ossec.conf` (Manager):

```xml
<ossec_config>
  <active-response>
    <command>firewall-drop</command>
    <location>local</location>
    <rules_id>100200</rules_id>
    <timeout>600</timeout>
  </active-response>
</ossec_config>
```

**Cara kerja:**

1. Alert DDoS terdeteksi → rule 100200 aktif
2. Manager kirim perintah ke Agent target
3. Agent jalankan `iptables -I INPUT -s <IP> -j DROP`
4. IP penyerang otomatis diblokir selama 600 detik (10 menit)
5. Setelah timeout, block otomatis dilepas

---

## Logging Density & Distribution — Point 3

### Mengapa Logging Density Penting?

DDoS dapat menghasilkan **jutaan log per menit**. Tanpa pengelolaan yang baik:
- Storage server bisa penuh dalam hitungan jam
- Performance sistem monitoring menurun drastis
- Sulit menemukan alert yang benar-benar relevan

**Bukti nyata:** Saat HTTP Flood berjalan, Wazuh Agent melaporkan:

```
"Target 'agent' message queue is full (1024). Log lines may be lost."
```

Ini menunjukkan volume log DDoS sangat tinggi hingga overflow queue!

---

### Konfigurasi Logging

File: `/var/ossec/etc/ossec.conf` (Manager)

```xml
<global>
  <!-- Format JSON untuk mudah diparse dan dianalisis -->
  <jsonout_output>yes</jsonout_output>
  <alerts_log>yes</alerts_log>
  <!-- Tidak log semua event agar hemat storage -->
  <logall>no</logall>
  <logall_json>no</logall_json>
</global>

<alerts>
  <!-- Simpan ke file hanya jika level >= 3 (filter noise) -->
  <log_alert_level>3</log_alert_level>
  <!-- Kirim email hanya jika level >= 12 (critical only) -->
  <email_alert_level>12</email_alert_level>
</alerts>
```

### Log yang Dipantau per Agent

```
/var/log/auth.log          → Login, SSH, sudo activity
/var/log/syslog            → System events
/var/log/kern.log          → Kernel events
/var/log/dpkg.log          → Package install/remove
/var/log/nginx/access.log  → Web traffic (HTTP Flood detection)
```

### Distribusi Log per Agent

| Agent | Role | Events | Jenis Log Dominan |
|-------|------|--------|-------------------|
| vm-wazuh-manager | Manager | ~40% | SSH access, system |
| vm-agent-02 | Target DDoS | ~35% | nginx HTTP flood, SSH |
| vm-agent-1 | Attacker | ~25% | SSH access, sudo |
| **Total** | | **17,490+ events** | |

### Statistik Logging

```bash
# Total events tercatat
sudo wc -l /var/ossec/logs/alerts/alerts.json
# Output: 17490

# Monitor ukuran log real-time
watch -n2 "du -sh /var/ossec/logs/alerts/"

# Distribusi per agent
sudo cat /var/ossec/logs/alerts/alerts.json | python3 -c "
import sys, json
from collections import Counter
agents = Counter()
for line in sys.stdin:
  try:
    d = json.loads(line)
    agent = d.get('agent', {}).get('name', 'unknown')
    agents[agent] += 1
  except: pass
for agent, count in agents.most_common():
  print(f'{agent}: {count} events')
"
```

---

## Hasil dan Kesimpulan

### Pencapaian

| Point | Requirement | Status | Bukti |
|-------|-------------|--------|-------|
| 1 | Deploy Wazuh Manager + 2 Agent di Azure | Done | `agent_control -l`: 2 Active |
| 1 | Azure Student Free Tier | Done | $100 kredit, ~$9 terpakai |
| 2 | DDoS HTTP Flood Simulation | Done | `ab` flood 10.0.0.5 → 10.0.0.6 |
| 2 | Deteksi anomali traffic | Done | Traffic naik 1228 → 3072 |
| 2 | Generate critical alerts | Done | 287 events dalam 15 menit |
| 2 | Active Response auto-block | Done | firewall-drop dikonfigurasi |
| 3 | Logging density management | Done | `log_alert_level: 3` |
| 3 | Log distribution | Done | 17,490 events dari 3 node |

### Kesimpulan

Wazuh SIEM terbukti efektif sebagai solusi keamanan untuk mendeteksi serangan DDoS:

1. **Deteksi real-time** — Wazuh mendeteksi anomali traffic HTTP dalam hitungan detik melalui nginx access log
2. **Traffic analysis** — Mampu membandingkan baseline traffic normal (1,228 req) vs saat DDoS (3,072 req) — naik 2.5x
3. **Source identification** — IP penyerang (10.0.0.5) teridentifikasi secara otomatis
4. **MITRE ATT&CK mapping** — Alert otomatis dipetakan ke framework MITRE ATT&CK
5. **Active Response** — Sistem siap auto-block IP penyerang menggunakan iptables
6. **Logging management** — 17,490 events berhasil dikelola dengan filter level yang tepat agar tidak overflow storage

---

## Referensi

- [Wazuh Official Documentation](https://documentation.wazuh.com)
- [Wazuh 4.7 Installation Guide](https://documentation.wazuh.com/4.7/installation-guide)
- [Wazuh Active Response](https://documentation.wazuh.com/current/user-manual/capabilities/active-response)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [Azure for Students](https://azure.microsoft.com/free/students)
- [ApacheBench Documentation](https://httpd.apache.org/docs/2.4/programs/ab.html)
- [UU ITE No. 11 Tahun 2008](https://jdih.kominfo.go.id)

---

*Tugas Keamanan Jaringan — Institut Teknologi Sepuluh Nopember (ITS) 2026*  
*Last updated: 19 Mei 2026*
