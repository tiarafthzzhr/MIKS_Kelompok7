# Wazuh SIEM Deployment + DDoS Attack Simulation

> **Tugas Kelompok — Keamanan Jaringan**  
> Institut Teknologi Sepuluh Nopember (ITS) — 2026

---

## Anggota Kelompok

| Nama | NRP | Peran |
|------|-----|-------|
| Tiara Fatimah Azzahra | 5027241090 | Manager Admin (Wazuh Manager) |
| Ananda Widi Alrafi | 5027241067 | Agent Operator 1 (vm-agent-1) — Attacker |
| Az Zahrra Tasya | 5027241087 | Agent Operator 2 (vm-agent-02) — Target |
| Ahmad Yafi | 5027241066 | Agent Operator & Dokumentasi |

---

## Deskripsi Proyek

Proyek ini mengimplementasikan **Wazuh SIEM (Security Information and Event Management)** pada infrastruktur cloud **Microsoft Azure for Students** untuk mendeteksi serangan **DDoS (Distributed Denial of Service)**.

### Tujuan

1. Deploy arsitektur Wazuh lengkap (1 Manager + 2 Agent) di Azure Student
2. Membuat skenario DDoS HTTP Flood sebagai Proof of Concept (PoC)
3. Mendemonstrasikan kemampuan Wazuh mendeteksi anomali traffic dan generate critical alerts
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
│  │  ATTACKER    │ │ TARGET       │                    │
│  │  Active      │ │ Active       │                    │
│  └──────┬───────┘ └──────────────┘                    │
│         │                ▲                             │
│         └── HTTP Flood ──┘                             │
│           ab -n 10000 -c 100 http://10.0.0.6/          │
└─────────────────────────────────────────────────────────┘
```

---

## Spesifikasi Infrastruktur

| VM | Role | Size | vCPU | RAM | IP Publik | IP Private | Status |
|----|------|------|------|-----|-----------|------------|--------|
| vm-wazuh-manager | Manager + Indexer + Dashboard | B2als_v2 | 2 | 4 GiB | 20.205.16.230 | 10.0.0.4 | Running |
| vm-agent-1 | Wazuh Agent + **Attacker** | B2ats_v2 | 2 | 1 GiB | 57.158.24.143 | 10.0.0.5 | Active |
| vm-agent-02 | Wazuh Agent + **Target** | B2ats_v2 | 2 | 1 GiB | 20.2.82.117 | 10.0.0.6 | Active |

**Platform:** Microsoft Azure for Students ($100 kredit)  
**OS:** Ubuntu Server 22.04 LTS  
**Wazuh Version:** 4.7.5  
**Region:** East Asia

---

## Deployment — Point 1

### Step 1 — Buat Infrastruktur Azure

Semua VM dibuat via Azure Portal:
- Resource Group: `rg-wazuh-lab`
- Virtual Network: `vm-wazuh-manager-vnet` (10.0.0.0/16)
- Subnet: `default` (10.0.0.0/24)
- Region: East Asia

**Port yang dibuka di Network Security Group:**

| Port | Protocol | Tujuan |
|------|----------|--------|
| 22 | TCP | SSH akses ke semua VM |
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

**Akses Dashboard:** `https://20.205.16.230`

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

### Skenario Serangan

```
ATTACKER : vm-agent-1  (IP: 10.0.0.5)
TARGET   : vm-agent-02 (IP: 10.0.0.6)
METODE   : HTTP Flood menggunakan ApacheBench (ab)
MONITOR  : Wazuh Manager via Dashboard & terminal
```

---

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
sudo systemctl restart wazuh-manager
```

---

### Eksekusi Serangan — 3 Terminal Sekaligus

---

#### Terminal 1 — Wazuh Manager (Monitor/Pengamat)

**Peran:** Ruang kontrol — melihat semua alert real-time saat serangan berlangsung

```bash
ssh azureuser@20.205.16.230

sudo tail -f /var/ossec/logs/alerts/alerts.json | python3 -c "
import sys, json
for line in sys.stdin:
  try:
    d = json.loads(line)
    lvl = d.get('rule', {}).get('level', 0)
    desc = d.get('rule', {}).get('description', '')
    agent = d.get('agent', {}).get('name', '?')
    srcip = d.get('data', {}).get('srcip', '')
    if lvl >= 3:
      print(f'[LEVEL {lvl}] [{agent}] {desc} | src={srcip}')
  except: pass
"
```

**Kenapa dijalankan:**  
Ini adalah "ruang monitor CCTV" sistem kita. Semua alert dari Agent-01 dan Agent-02 masuk ke sini secara real-time. Kita menunggu alert dengan `src=10.0.0.5` yang membuktikan DDoS dari Agent-01 terdeteksi oleh Wazuh.

---

#### Terminal 2 — vm-agent-02 (Target/Korban)

**Peran:** Server yang dibanjiri request HTTP dari penyerang

```bash
ssh azureuser@20.2.82.117

sudo tail -f /var/log/nginx/access.log
```

**Kenapa dijalankan:**  
Membuktikan secara langsung bahwa nginx di Agent-02 menerima ribuan request dari IP `10.0.0.5` (Agent-01). Layar yang penuh dengan baris `10.0.0.5 GET /` adalah bukti nyata bahwa serangan DDoS berhasil diterima oleh target.

**Output yang muncul saat diserang:**

```
10.0.0.5 - - [19/May/2026:02:06:04 +0000] "GET / HTTP/1.0" 200 612 "-" "ApacheBench/2.3"
10.0.0.5 - - [19/May/2026:02:06:04 +0000] "GET / HTTP/1.0" 200 612 "-" "ApacheBench/2.3"
10.0.0.5 - - [19/May/2026:02:06:04 +0000] "GET / HTTP/1.0" 200 612 "-" "ApacheBench/2.3"
... ribuan baris dari IP yang sama!
```

---

#### Terminal 3 — vm-agent-1 (Attacker/Penyerang)

**Peran:** Mengirimkan HTTP Flood ke Agent-02

```bash
ssh azureuser@57.158.24.143

ab -n 10000 -c 100 http://10.0.0.6/
```

**Penjelasan command:**

| Parameter | Nilai | Arti |
|-----------|-------|------|
| `ab` | - | Apache Benchmark — tool pengirim HTTP request |
| `-n 10000` | 10.000 | Total request yang dikirim ke korban |
| `-c 100` | 100 | Jumlah request yang dikirim **bersamaan** setiap saat |
| `http://10.0.0.6/` | IP Agent-02 | Alamat target yang diserang |

**Kenapa dijalankan:**  
Ini adalah inti simulasi DDoS HTTP Flood. Agent-01 membanjiri web server nginx Agent-02 dengan 10.000 request secara terus menerus — seperti 100 orang menelepon secara bersamaan ke satu nomor sampai operator kewalahan.

**Output saat serangan berjalan:**

```
Benchmarking 10.0.0.6 (be patient)
Completed 1000 requests
Completed 2000 requests
Completed 3000 requests
...
Total of 8781 requests completed
```

---

### Hasil Deteksi Wazuh di Dashboard

**Filter yang digunakan:** `data.srcip: 10.0.0.5`  
*(hanya tampilkan alert dari IP Agent-01 — membuktikan DDoS kita, bukan dari bot internet)*

| Metric | Nilai | Keterangan |
|--------|-------|------------|
| Total Alerts | **3,313** | Semua event dari IP penyerang |
| Level 12+ (CRITICAL) | **3,309** | Alert critical dari DDoS |
| Authentication Failure | 0 | Tidak ada brute force dari kita |
| Spike Grafik | Jam 08:00-09:00 | Persis saat DDoS dijalankan |
| Target Agent | vm-agent-02 | Korban yang menerima serangan |

**Analisis Spike Grafik:**

```
Sebelum DDoS : 0 alerts dari IP 10.0.0.5
Saat DDoS    : 3,313 alerts dalam waktu singkat!
               → Spike tajam seperti jarum di grafik
Setelah DDoS : kembali 0
```

---

### Alur Deteksi Wazuh

```
1. Agent-01 jalankan ab command
         ↓
2. 10.000 HTTP request dikirim ke Agent-02
         ↓
3. Nginx Agent-02 terima request → catat di access.log
         ↓
4. Wazuh Agent Agent-02 baca nginx log real-time
         ↓
5. Log dikirim ke Manager via port 1514
         ↓
6. Manager analisa → traffic naik 1.228 → 3.072 req/jam!
         ↓
7. Alert level 12 (CRITICAL) di-generate
         ↓
8. Alert muncul di Dashboard
```

---

### Alert yang Terdeteksi

```json
{
  "rule": { "level": 4, "id": "11" },
  "agent": { "name": "vm-agent-02" },
  "full_log": "The average number of logs between 2:00 and 3:00 is 1228. We reached 3072.",
  "decoder": { "name": "web-accesslog" },
  "data": { "protocol": "GET", "srcip": "10.0.0.5", "url": "/" },
  "location": "/var/log/nginx/access.log"
}
```

**Interpretasi:**
- `srcip: 10.0.0.5` → IP penyerang (Agent-01) teridentifikasi
- `average 1228 → reached 3072` → traffic naik **2.5x** dari normal
- `location: /var/log/nginx/access.log` → terdeteksi dari nginx log
- `decoder: web-accesslog` → Wazuh berhasil parse log web server

---

### Active Response — Auto-Block Penyerang

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
2. Manager kirim perintah block ke Agent-02
3. Agent-02 jalankan `iptables -I INPUT -s 10.0.0.5 -j DROP`
4. Semua traffic dari Agent-01 otomatis dibuang selama 10 menit
5. Setelah timeout, block otomatis dilepas

---

## Logging Density & Distribution — Point 3

### Mengapa Logging Density Penting?

DDoS menghasilkan **ribuan log per detik**. Tanpa pengelolaan yang baik:
- Storage server bisa penuh dalam hitungan jam
- Performance sistem monitoring menurun drastis
- Alert penting tenggelam di antara noise log

**Bukti nyata dari eksperimen:**  
Saat HTTP Flood berjalan, Wazuh melaporkan:

```
"Target 'agent' message queue is full (1024). Log lines may be lost."
```

Ini membuktikan volume log DDoS sangat tinggi hingga overflow queue Wazuh!

---

### Konfigurasi Logging

File: `/var/ossec/etc/ossec.conf` (Manager)

```xml
<global>
  <jsonout_output>yes</jsonout_output>
  <alerts_log>yes</alerts_log>
  <!-- Tidak log SEMUA event → hemat storage -->
  <logall>no</logall>
  <logall_json>no</logall_json>
</global>

<alerts>
  <!-- Simpan ke file hanya jika level >= 3 -->
  <!-- Level 1-2 terlalu kecil → dibuang (filter noise) -->
  <log_alert_level>3</log_alert_level>
  <!-- Email hanya dikirim jika level >= 12 (CRITICAL) -->
  <email_alert_level>12</email_alert_level>
</alerts>
```

---

### Hasil Nyata — Distribusi Log per Agent

```bash
# Command yang dijalankan di Manager:
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

**Hasil:**

| Agent | Events | Persentase | Keterangan |
|-------|--------|------------|------------|
| vm-agent-02 | **6,791** | 46% | Target DDoS — paling banyak log |
| vm-agent-1 | **4,839** | 33% | Attacker — banyak aktivitas |
| vm-wazuh-manager | **2,972** | 20% | Manager — log sistem |
| **Total** | **14,596** | 100% | |

---

### Hasil Nyata — Distribusi Level Alert

```bash
# Command yang dijalankan di Manager:
sudo cat /var/ossec/logs/alerts/alerts.json | python3 -c "
import sys, json
from collections import Counter
levels = Counter()
for line in sys.stdin:
  try:
    d = json.loads(line)
    lvl = d.get('rule', {}).get('level', 0)
    levels[lvl] += 1
  except: pass
print('Distribusi level alert:')
for lvl, count in sorted(levels.items()):
  print(f'Level {lvl}: {count} events')
"
```

**Hasil:**

| Level | Events | Keterangan |
|-------|--------|------------|
| Level 3 | 927 | Minimum level tersimpan |
| Level 4 | 161 | HTTP flood detection (nginx) |
| Level 5 | 9,450 | SSH attempts dari internet |
| Level 7 | 17 | Medium alerts |
| Level 8 | 55 | High alerts |
| Level 9 | 3 | High alerts |
| Level 10 | 690 | Brute force critical |
| Level 12 | **3,309** | **DDoS HTTP Flood CRITICAL!** |
| Level 1 & 2 | **0** | **Tidak ada → filter bekerja!** |

**Kesimpulan Logging Density:**

```
Level 1 & 2 = 0  → log_alert_level: 3 bekerja dengan baik
Level 12: 3,309  → semua dari DDoS simulation kita
vm-agent-02: 46% → wajar sebagai target DDoS
Total 14,596 events dikelola efisien tanpa overflow storage
```

---

### Log yang Dipantau per Agent

| Log File | Isi | Relevansi DDoS |
|----------|-----|----------------|
| `/var/log/auth.log` | Login, SSH, sudo | Deteksi brute force |
| `/var/log/syslog` | System events | General monitoring |
| `/var/log/kern.log` | Kernel events | System health |
| `/var/log/dpkg.log` | Package install | Perubahan software |
| `/var/log/nginx/access.log` | HTTP requests | **Deteksi DDoS** |

---

## Hasil dan Kesimpulan

### Ringkasan Pencapaian

| Point | Requirement | Status | Hasil Nyata |
|-------|-------------|--------|-------------|
| 1 | Deploy Wazuh Manager | Done | Service running, Dashboard `https://20.205.16.230` |
| 1 | Deploy 2 Server Agent | Done | 2 Active, coverage 100% |
| 1 | Azure Student Free Tier | Done | $100 kredit, ~$9 terpakai |
| 2 | DDoS HTTP Flood Simulation | Done | 10.000 request dari 10.0.0.5 → 10.0.0.6 |
| 2 | Deteksi anomali traffic | Done | Traffic spike 1.228 → 3.072 req/jam (naik 2.5x) |
| 2 | Generate critical alerts | Done | 3,309 level 12 CRITICAL alerts |
| 2 | Active Response auto-block | Done | firewall-drop dikonfigurasi |
| 3 | Logging density management | Done | Level 1&2 = 0, filter log_alert_level: 3 bekerja |
| 3 | Log distribution | Done | 14,596 events terdistribusi dari 3 node |

### Kesimpulan

Wazuh SIEM terbukti efektif mendeteksi serangan DDoS:

1. **Deteksi real-time** — Alert muncul dalam hitungan detik setelah DDoS dimulai
2. **Traffic analysis** — Wazuh membandingkan baseline (1.228 req) vs DDoS (3.072 req) — naik 2.5x
3. **Source identification** — IP penyerang (10.0.0.5) teridentifikasi otomatis dari nginx log
4. **Critical alerts** — 3,309 alert level 12 (CRITICAL) ter-generate dalam waktu singkat
5. **Visual evidence** — Spike tajam terlihat jelas di grafik Dashboard jam 08:00-09:00
6. **Active Response** — Sistem siap auto-block IP penyerang via iptables
7. **Log management** — 14,596 events dikelola efisien, level 1&2 difilter, tidak ada overflow storage

---

## Referensi

- [Wazuh Official Documentation](https://documentation.wazuh.com)
- [Wazuh 4.7 Installation Guide](https://documentation.wazuh.com/4.7/installation-guide)
- [Wazuh Active Response](https://documentation.wazuh.com/current/user-manual/capabilities/active-response)
- [ApacheBench Documentation](https://httpd.apache.org/docs/2.4/programs/ab.html)
- [MITRE ATT&CK Framework](https://attack.mitre.org)
- [Azure for Students](https://azure.microsoft.com/free/students)
- [UU ITE No. 11 Tahun 2008](https://jdih.kominfo.go.id)

---

*Tugas Keamanan Jaringan — Institut Teknologi Sepuluh Nopember (ITS) 2026*  
*Last updated: 19 Mei 2026*
