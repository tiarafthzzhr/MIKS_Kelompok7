# Wazuh SIEM Deployment + DDoS Attack Simulation

> **Tugas Kelompok — Keamanan Jaringan**  
> Institut Teknologi Sepuluh Nopember (ITS)

---

## Anggota Kelompok

| Nama | NRP | Peran |
|------|-----|-------|
| Tiara Fatimah Azzahra | 5027241090 | Manager Admin (Wazuh Manager) |
| Ananda Widi Alrafi | 5027241067 | Agent Operator 1 (vm-agent-1) |
| Az Zahrra Tasya | 5027241087 | Agent Operator 2 (vm-agent-02) |
| Ahmad Yafi | 5027241066 | Agent Operator 3 (vm-agent-03) |

---

## Deskripsi Proyek

Proyek ini mengimplementasikan **Wazuh SIEM (Security Information and Event Management)** pada infrastruktur cloud **Microsoft Azure for Students** untuk mendeteksi serangan **DDoS (Distributed Denial of Service)**.

Sistem terdiri dari:
- **1 Wazuh Manager** — pusat analisis, indexer, dan dashboard monitoring
- **3 Wazuh Agent** — server target yang dipantau secara real-time

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
│  │  Services:                           │               │
│  │  wazuh-manager  (port 1514/1515)     │               │
│  │  wazuh-indexer  (port 9200)          │               │
│  │  wazuh-dashboard (port 443)          │               │
│  └──────────────┬───────────────────────┘               │
│                 │ Port 1514/1515 TCP                     │
│        ┌────────┼────────┐                              │
│        ▼        ▼        ▼                              │
│  ┌──────────┐ ┌──────────┐ ┌──────────┐               │
│  │ Agent-1  │ │ Agent-02 │ │ Agent-03 │               │
│  │10.0.0.5  │ │10.0.0.6  │ │[IP Teman]│               │
│  │B2ats_v2  │ │B2ats_v2  │ │B2ats_v2  │               │
│  │ Active   │ │ Active   │ │ Setup    │               │
│  └──────────┘ └──────────┘ └──────────┘               │
└─────────────────────────────────────────────────────────┘
```

---

## Spesifikasi Infrastruktur

| VM | Role | Size | vCPU | RAM | IP Publik | IP Private | Status |
|----|------|------|------|-----|-----------|------------|--------|
| vm-wazuh-manager | Manager + Indexer + Dashboard | B2als_v2 | 2 | 4 GiB | 20.205.16.230 | 10.0.0.4 | Running |
| vm-agent-1 | Wazuh Agent 1 | B2ats_v2 | 2 | 1 GiB | 57.158.24.143 | 10.0.0.5 | Active |
| vm-agent-02 | Wazuh Agent 2 | B2ats_v2 | 2 | 1 GiB | 20.2.82.117 | 10.0.0.6 | Active |
| vm-agent-03 | Wazuh Agent 3 | B2ats_v2 | 2 | 1 GiB | [IP Teman] | - | Setup |

**Platform:** Microsoft Azure for Students ($100 kredit)  
**OS:** Ubuntu Server 22.04 LTS  
**Wazuh Version:** 4.7.5  
**Region:** East Asia

---

## Langkah Deployment

### Step 1 — Buat Infrastruktur Azure

Buat semua resource via **Azure Portal** (portal.azure.com):

```
Resource Group : rg-wazuh-lab
Region         : East Asia
Virtual Network: vm-wazuh-manager-vnet (10.0.0.0/16)
Subnet         : default (10.0.0.0/24)
```

> Semua VM HARUS pakai VNet yang sama agar bisa komunikasi via IP private!

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

# Edit config.yml
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
# Install (jalankan urut, tunggu tiap step selesai ~15 menit)
sudo bash wazuh-install.sh --generate-config-files
sudo bash wazuh-install.sh --wazuh-indexer node-1
sudo bash wazuh-install.sh --start-cluster
sudo bash wazuh-install.sh --wazuh-server wazuh-1
sudo bash wazuh-install.sh --wazuh-dashboard dashboard
```

> **PENTING:** Simpan username dan password yang muncul di akhir instalasi!

**Akses Dashboard:** `https://20.205.16.230`

---

### Step 3 — Install Wazuh Agent

> Lakukan di setiap VM Agent. Ganti nama agent sesuai urutannya.

```bash
# SSH ke Agent VM
ssh azureuser@<IP-PUBLIK-AGENT>

# Update sistem
sudo apt-get update && sudo apt-get upgrade -y

# Tambah GPG key Wazuh
curl -s https://packages.wazuh.com/key/GPG-KEY-WAZUH | sudo gpg \
  --no-default-keyring \
  --keyring gnupg-ring:/usr/share/keyrings/wazuh.gpg \
  --import && sudo chmod 644 /usr/share/keyrings/wazuh.gpg

# Tambah repository
echo "deb [signed-by=/usr/share/keyrings/wazuh.gpg] \
  https://packages.wazuh.com/4.x/apt/ stable main" | \
  sudo tee /etc/apt/sources.list.d/wazuh.list

sudo apt-get update

# Install Wazuh Agent versi 4.7.5 (HARUS sama dengan Manager!)
# Jika 1 akun Azure (pakai IP private Manager):
WAZUH_MANAGER="10.0.0.4" WAZUH_AGENT_NAME="agent-01" \
  sudo apt-get install wazuh-agent=4.7.5-1 -y

# Jika beda akun Azure (pakai IP publik Manager):
WAZUH_MANAGER="20.205.16.230" WAZUH_AGENT_NAME="agent-03" \
  sudo apt-get install wazuh-agent=4.7.5-1 -y

# Fix konfigurasi IP Manager di ossec.conf
sudo nano /var/ossec/etc/ossec.conf
# Cari: MANAGER_IP → ganti dengan IP Manager yang sesuai
# Simpan: Ctrl+X → Y → Enter

# Aktifkan dan jalankan Agent
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent

# Registrasi ke Manager
sudo /var/ossec/bin/agent-auth -m 10.0.0.4
# atau jika beda akun:
sudo /var/ossec/bin/agent-auth -m 20.205.16.230
```

> Versi Agent HARUS sama atau lebih rendah dari Manager (4.7.5).

**Verifikasi (jalankan di Manager):**

```bash
sudo /var/ossec/bin/agent_control -l

# Output yang diharapkan:
# ID: 000, Name: vm-wazuh-manager (server), IP: 127.0.0.1, Active/Local
# ID: 001, Name: vm-agent-1,  IP: any, Active
# ID: 002, Name: vm-agent-02, IP: any, Active
# ID: 003, Name: vm-agent-03, IP: any, Active
```

---

### Step 4 — Panduan Agent-03 (Akun Azure Sendiri)

```
1. Daftar Azure Student: azure.microsoft.com/free/students
   Pakai email kampus (NRP@student.its.ac.id)

2. Buat VM di Azure Portal:
   - VM name : vm-agent-03
   - Region  : East Asia
   - Image   : Ubuntu Server 22.04 LTS - Gen2
   - Size    : Standard B2ats_v2
   - Username: azureuser
   - Port    : SSH (22)

3. SSH masuk dan ikuti Step 3 di atas
   Gunakan IP PUBLIK Manager: 20.205.16.230
```

---

## Skenario DDoS Attack Simulation

> **Disclaimer:** Simulasi HANYA pada VM lab milik sendiri.  
> DDoS ke sistem orang lain = tindak pidana (UU ITE No. 11 Tahun 2008).

### Pasang Custom DDoS Detection Rules (Manager)

```bash
sudo nano /var/ossec/etc/rules/ddos_rules.xml
```

```xml
<group name="ddos,attack,">

  <!-- HTTP Flood Detection -->
  <rule id="100200" level="12">
    <if_group>web|firewall</if_group>
    <same_source_ip />
    <frequency>500</frequency>
    <timeframe>10</timeframe>
    <description>HTTP Flood DDoS: lebih 500 req/10 detik dari 1 IP</description>
    <group>ddos,http_flood,</group>
  </rule>

  <!-- SYN Flood Detection (CRITICAL) -->
  <rule id="100201" level="14">
    <if_group>firewall</if_group>
    <match>SYN</match>
    <same_source_ip />
    <frequency>1000</frequency>
    <timeframe>5</timeframe>
    <description>CRITICAL: SYN Flood attack detected</description>
    <group>ddos,syn_flood,</group>
  </rule>

  <!-- Server Overload -->
  <rule id="100202" level="10">
    <if_group>syslog</if_group>
    <match>Too many connections</match>
    <description>Server overload - possible DDoS</description>
    <group>ddos,overload,</group>
  </rule>

</group>
```

```bash
sudo systemctl restart wazuh-manager
```

---

### Skenario 1 — HTTP Flood (Agent-01)

```bash
ssh azureuser@57.158.24.143

sudo apt install apache2-utils nginx -y
sudo systemctl start nginx

# Flood 50.000 request, 200 concurrent
ab -n 50000 -c 200 http://localhost/
```

### Skenario 2 — SYN Flood (Agent-02)

```bash
ssh azureuser@20.2.82.117

sudo apt install hping3 nginx -y
sudo systemctl start nginx

# SYN Flood 30 detik (Ctrl+C untuk stop)
sudo hping3 -S -p 80 --flood 127.0.0.1
```

### Skenario 3 — Slowloris (Agent-03)

```bash
ssh azureuser@<IP-AGENT-03>

sudo apt install python3-pip -y
pip3 install slowloris

slowloris localhost --port 80 --socket 200
```

### Monitor Real-time (Manager)

```bash
sudo tail -f /var/ossec/logs/alerts/alerts.json | \
  python3 -c "
import sys, json
for line in sys.stdin:
  try:
    d = json.loads(line)
    lvl = d.get('rule', {}).get('level', 0)
    desc = d.get('rule', {}).get('description', '')
    agent = d.get('agent', {}).get('name', '?')
    if lvl >= 8:
      print(f'[LEVEL {lvl}] [{agent}] {desc}')
  except: pass
"

# Dashboard: https://20.205.16.230
# Menu: Security Events → filter: rule.groups: ddos
```

---

## Active Response — Auto-Block Penyerang

Tambahkan di `/var/ossec/etc/ossec.conf` pada Manager:

```xml
<active-response>
  <command>firewall-drop</command>
  <location>local</location>
  <rules_id>100200,100201</rules_id>
  <timeout>600</timeout>
</active-response>
```

```bash
sudo systemctl restart wazuh-manager

# Verifikasi saat DDoS berlangsung:
sudo iptables -L INPUT -n | grep DROP
sudo tail -f /var/ossec/logs/active-responses.log
```

---

## Pengelolaan Logging Density

```xml
<!-- Di ossec.conf Manager -->
<global>
  <jsonout_output>yes</jsonout_output>
  <alerts_log>yes</alerts_log>
  <logall>no</logall>
</global>

<alerts>
  <log_alert_level>3</log_alert_level>
  <email_alert_level>10</email_alert_level>
</alerts>
```

```bash
# Monitor storage saat DDoS
watch -n2 "du -sh /var/ossec/logs/alerts/"
```

---

## Checklist Progress

- [x] Buat Resource Group `rg-wazuh-lab` di Azure
- [x] Buat Virtual Network `vm-wazuh-manager-vnet`
- [x] Buat & install VM Manager — Wazuh v4.7.5
- [x] Akses Wazuh Dashboard via browser
- [x] Install & registrasi Agent-01 → Active
- [x] Install & registrasi Agent-02 → Active
- [ ] Setup Agent-03 (akun Azure sendiri)
- [ ] Reset password Wazuh Dashboard
- [ ] Pasang custom DDoS detection rules
- [ ] Install nginx & tools di semua Agent
- [ ] Jalankan simulasi HTTP Flood (Agent-01)
- [ ] Jalankan simulasi SYN Flood (Agent-02)
- [ ] Jalankan simulasi Slowloris (Agent-03)
- [ ] Screenshot alert di Wazuh Dashboard
- [ ] Konfigurasi Active Response auto-block
- [ ] Dokumentasi hasil dan kesimpulan

---

## Yang Harus Dilanjutkan

### 1. Fix Password Wazuh Dashboard

```bash
ssh azureuser@20.205.16.230
sudo systemctl restart wazuh-indexer
# Tunggu 1 menit
sudo /usr/share/wazuh-indexer/plugins/opensearch-security/tools/wazuh-passwords-tool.sh \
  -u admin -p WazuhLab2025.*
sudo systemctl restart wazuh-dashboard
```

### 2. Setup Agent-03

Kirim panduan Step 4 ke teman untuk daftar Azure Student dan install agent.

### 3. Jalankan Simulasi DDoS

Koordinasi semua anggota — serang bersamaan, ketua monitor di Dashboard.

### 4. Screenshot untuk Laporan

- Dashboard saat alert muncul (level 12-14)
- Output `agent_control -l` semua agent Active
- Log alert terminal real-time

---

## Tips Hemat Kredit Azure

```bash
# Matikan VM saat tidak digunakan!
# portal.azure.com → Virtual machines → Stop (Deallocate)

# Estimasi biaya:
# vm-wazuh-manager (B2als_v2) : ~$38/bulan
# vm-agent-1       (B2ats_v2) : ~$9.56/bulan
# vm-agent-02      (B2ats_v2) : ~$9.56/bulan
# Total                        : ~$57/bulan dari $100 kredit
```

> Gunakan **Deallocate** bukan Stop biasa — Deallocate benar-benar menghentikan billing compute.

---

## Referensi

- [Wazuh Documentation](https://documentation.wazuh.com)
- [Wazuh 4.7 Installation Guide](https://documentation.wazuh.com/4.7/installation-guide)
- [Wazuh Active Response](https://documentation.wazuh.com/current/user-manual/capabilities/active-response)
- [Azure for Students](https://azure.microsoft.com/free/students)

---

*Tugas Keamanan Jaringan — ITS 2026 | Last updated: 16 Mei 2026*
