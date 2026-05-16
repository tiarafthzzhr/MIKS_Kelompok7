# Wazuh SIEM Deployment + DDoS Attack Simulation

> **Tugas Kelompok — Keamanan Jaringan**  
> Institut Teknologi Sepuluh Nopember (ITS) — 2026

---

## Anggota Kelompok 7

| Nama | NRP | Peran |
|------|-----|-------|
| Tiara Fatimah Azzahra | 5027241090 | Manager Admin (Wazuh Manager) |
| Ananda Widi Alrafi | 5027241067 | Agent Operator 1 (vm-agent-1) |
| Az Zahrra Tasya | 5027241087 | Agent Operator 2 (vm-agent-02) |
| Ahmad Yafi | 5027241066 | Agent Operator 3 (vm-agent-03) |

---

## Deskripsi Proyek

Proyek ini mengimplementasikan **Wazuh SIEM (Security Information and Event Management)** pada infrastruktur cloud **Microsoft Azure** untuk mendeteksi serangan **DDoS (Distributed Denial of Service)**.

Sistem terdiri dari:
- **1 Wazuh Manager** — pusat analisis dan monitoring keamanan
- **3 Wazuh Agent** — server target yang dipantau

---

## Arsitektur Sistem

```
┌─────────────────────────────────────────────────────┐
│                   Microsoft Azure                    │
│                                                     │
│  ┌─────────────────┐    Port 1514/1515 (TCP)        │
│  │  Wazuh Manager  │◄─────────────────────────┐    │
│  │  vm-wazuh-manager│                          │    │
│  │  IP: 20.205.16.230│                         │    │
│  │  Private: 10.0.0.4│                         │    │
│  │  Ubuntu 22.04   │                           │    │
│  │  B2als_v2       │                           │    │
│  │  2 vCPU, 4GB RAM│    ┌──────────────────┐  │    │
│  └─────────────────┘    │   vm-agent-1     │──┘    │
│         │               │   IP: 57.158.24.143│      │
│    Port 443             │   Private: 10.0.0.5│      │
│    (Dashboard)          │   Ubuntu 22.04    │      │
│         │               │   B2ats_v2        │      │
│    ┌────▼────┐          └──────────────────┘       │
│    │ Browser │          ┌──────────────────┐        │
│    │Dashboard│          │   vm-agent-02    │        │
│    └─────────┘          │   IP: 20.2.82.117│        │
│                         │   Private: 10.0.0.6│      │
│                         │   Ubuntu 22.04    │       │
│                         │   B2ats_v2        │       │
│                         └──────────────────┘        │
│                         ┌──────────────────┐        │
│                         │   vm-agent-03    │        │
│                         │   Ubuntu 22.04   │        │
│                         │   B2ats_v2        │       │
│                         └──────────────────┘        │
└─────────────────────────────────────────────────────┘
```

---

## Spesifikasi Infrastruktur

| VM | Role | Size | vCPU | RAM | IP Publik | IP Private |
|----|------|------|------|-----|-----------|------------|
| vm-wazuh-manager | Wazuh Manager + Indexer + Dashboard | B2als_v2 | 2 | 4 GiB | 20.205.16.230 | 10.0.0.4 |
| vm-agent-1 | Wazuh Agent (Target 1) | B2ats_v2 | 2 | 1 GiB | 57.158.24.143 | 10.0.0.5 |
| vm-agent-02 | Wazuh Agent (Target 2) | B2ats_v2 | 2 | 1 GiB | 20.2.82.117 | 10.0.0.6 |
| vm-agent-03 | Wazuh Agent (Target 3) | B2ats_v2 | 2 | 1 GiB | - | - |

**Platform:** Microsoft Azure for Students ($100 kredit)  
**OS:** Ubuntu Server 22.04 LTS  
**Wazuh Version:** 4.7.5

---

## Panduan Deployment

### Prerequisites

- Akun Microsoft Azure for Students (email kampus ITS)
- SSH Client (PowerShell / Terminal)
- Browser untuk akses Wazuh Dashboard

---

### 1. Setup Wazuh Manager

```bash
# SSH ke Manager
ssh azureuser@20.205.16.230

# Update sistem
sudo apt-get update && sudo apt-get upgrade -y

# Download installer Wazuh
curl -sO https://packages.wazuh.com/4.7/wazuh-install.sh
curl -sO https://packages.wazuh.com/4.7/config.yml

# Edit config.yml — isi IP private Manager
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
# Jalankan installer (urut, tunggu tiap step selesai)
sudo bash wazuh-install.sh --generate-config-files
sudo bash wazuh-install.sh --wazuh-indexer node-1
sudo bash wazuh-install.sh --start-cluster
sudo bash wazuh-install.sh --wazuh-server wazuh-1
sudo bash wazuh-install.sh --wazuh-dashboard dashboard
```

> **PENTING:** Simpan username dan password yang muncul di akhir instalasi!  
> Contoh: `User: admin | Password: xxxxxxxxxx`

**Akses Dashboard:** `https://20.205.16.230`

---

### 2. Setup Wazuh Agent

> Lakukan langkah ini di setiap VM Agent (agent-1, agent-02, agent-03)

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

# Install Wazuh Agent versi 4.7.5
# Jika Agent di akun Azure yang sama (pakai IP private):
WAZUH_MANAGER="10.0.0.4" WAZUH_AGENT_NAME="agent-0X" \
  sudo apt-get install wazuh-agent=4.7.5-1 -y

# Jika Agent di akun Azure berbeda (pakai IP publik):
WAZUH_MANAGER="20.205.16.230" WAZUH_AGENT_NAME="agent-0X" \
  sudo apt-get install wazuh-agent=4.7.5-1 -y

# Fix konfigurasi IP Manager
sudo nano /var/ossec/etc/ossec.conf
# Cari MANAGER_IP → ganti dengan 10.0.0.4 atau 20.205.16.230

# Aktifkan dan jalankan Agent
sudo systemctl daemon-reload
sudo systemctl enable wazuh-agent
sudo systemctl start wazuh-agent

# Registrasi ke Manager
sudo /var/ossec/bin/agent-auth -m 10.0.0.4
```

**Verifikasi Agent terdaftar (jalankan di Manager):**

```bash
sudo /var/ossec/bin/agent_control -l
```

Output yang diharapkan:

```
ID: 000, Name: vm-wazuh-manager (server), IP: 127.0.0.1, Active/Local
ID: 001, Name: vm-agent-1,  IP: any, Active
ID: 002, Name: vm-agent-02, IP: any, Active
ID: 003, Name: vm-agent-03, IP: any, Active
```

---

## Skenario DDoS Attack Simulation

> **Disclaimer:** Simulasi ini dilakukan HANYA pada VM lab milik sendiri dalam lingkungan terkontrol. Melakukan DDoS terhadap sistem orang lain adalah tindak pidana sesuai UU ITE.

### Pasang Custom DDoS Detection Rules (di Manager)

```bash
sudo nano /var/ossec/etc/rules/ddos_rules.xml
```

```xml
<group name="ddos,attack,">

  <!-- Rule: HTTP Flood Detection -->
  <rule id="100200" level="12">
    <if_group>web|firewall</if_group>
    <same_source_ip />
    <frequency>500</frequency>
    <timeframe>10</timeframe>
    <description>HTTP Flood DDoS: lebih 500 request/10 detik dari 1 IP</description>
    <group>ddos,http_flood,</group>
  </rule>

  <!-- Rule: SYN Flood Detection (CRITICAL) -->
  <rule id="100201" level="14">
    <if_group>firewall</if_group>
    <match>SYN</match>
    <same_source_ip />
    <frequency>1000</frequency>
    <timeframe>5</timeframe>
    <description>CRITICAL: SYN Flood attack detected</description>
    <group>ddos,syn_flood,</group>
  </rule>

  <!-- Rule: Server Overload -->
  <rule id="100202" level="10">
    <if_group>syslog</if_group>
    <match>Too many connections</match>
    <description>Server connection overload - possible DDoS</description>
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

sudo apt install apache2-utils -y
ab -n 50000 -c 200 http://localhost/
```

### Skenario 2 — SYN Flood (Agent-02)

```bash
ssh azureuser@20.2.82.117

sudo apt install hping3 -y
sudo hping3 -S -p 80 --flood 127.0.0.1
```

### Monitor Alert Real-time (di Manager)

```bash
sudo tail -f /var/ossec/logs/alerts/alerts.json | \
  python3 -c "
import sys, json
for line in sys.stdin:
  try:
    d = json.loads(line)
    lvl = d.get('rule', {}).get('level', 0)
    desc = d.get('rule', {}).get('description', '')
    if lvl >= 8:
      print(f'[LEVEL {lvl}] {desc}')
  except: pass
"
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
```

---

## Hasil dan Temuan

### Alert yang Terdeteksi

| Rule ID | Level | Jenis Serangan | Deskripsi |
|---------|-------|----------------|-----------|
| 100200 | 12 — CRITICAL | HTTP Flood | >500 request/10 detik dari 1 IP |
| 100201 | 14 — CRITICAL | SYN Flood | >1000 SYN packet/5 detik |
| 100202 | 10 — HIGH | Server Overload | Koneksi server penuh |

### Kesimpulan

1. Wazuh berhasil mendeteksi serangan DDoS dalam hitungan detik setelah serangan dimulai
2. Active Response berhasil auto-block IP penyerang menggunakan iptables
3. Logging density dikelola dengan mengatur `log_alert_level` agar storage tidak penuh
4. Dashboard Wazuh memberikan visualisasi real-time yang memudahkan monitoring

---

## Tips Hemat Kredit Azure

```bash
# Matikan VM saat tidak digunakan via Azure CLI
az vm deallocate -g rg-wazuh-lab -n vm-wazuh-manager
az vm deallocate -g rg-wazuh-lab -n vm-agent-1
az vm deallocate -g rg-wazuh-lab -n vm-agent-02
```

> Gunakan **Deallocate** bukan Stop biasa — Deallocate benar-benar menghentikan billing compute.

---

## Referensi

- [Wazuh Official Documentation](https://documentation.wazuh.com)
- [Wazuh Installation Guide](https://documentation.wazuh.com/current/installation-guide)
- [Azure for Students](https://azure.microsoft.com/free/students)
- [Wazuh Active Response](https://documentation.wazuh.com/current/user-manual/capabilities/active-response)

---

*Dibuat untuk memenuhi tugas mata kuliah Keamanan Jaringan — Institut Teknologi Sepuluh Nopember 2026*
