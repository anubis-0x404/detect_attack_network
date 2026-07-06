#🛡️ Mini SIEM — Hệ thống Giám sát và Phát hiện Tấn công Mạng

![Python](https://img.shields.io/badge/Python-3.10+-blue?logo=python)
![Elasticsearch](https://img.shields.io/badge/Elasticsearch-8.x-005571?logo=elasticsearch)
![Kibana](https://img.shields.io/badge/Kibana-8.x-E8478B?logo=kibana)
![Suricata](https://img.shields.io/badge/Suricata-6.x-orange)
![License](https://img.shields.io/badge/License-MIT-green)

---

#Mô tả

Hệ thống **Mini SIEM** (Security Information and Event Management) được xây dựng nhằm giám sát và phát hiện các hành vi tấn công mạng thông qua phân tích log theo thời gian thực. Hệ thống tích hợp **Suricata IDS**, **ELK Stack** và **Detection Engine tự xây dựng bằng Python**, áp dụng hai kỹ thuật phát hiện chính:

- **Rule-based Detection** — phát hiện tấn công đơn sự kiện theo ngưỡng (threshold + time window)
- **Event Correlation** — phát hiện tấn công có chủ đích đa bước (multi-step attack chain)

---

#Kiến trúc hệ thống
┌─────────────┐    TCP/514    ┌──────────────────────────────────┐
│  Victim Host│ ────────────▶ │         SIEM Server              │
│ (Ubuntu)    │               │                                  │
│             │               │  ┌─────────┐    ┌─────────────┐  │
│ • Suricata  │               │  │ Parser  │───▶│Elasticsearch│  │
│ • Rsyslog   │               │  │Normalizer    └──────┬──────┘  │
│   Forwarder │               │  └─────────┘           │         │
└─────────────┘               │                        ▼         │
│  ┌──────────────────────────┐   _________________________
┌─────────────┐               │  │    Detection Engine      │    │
│  Kali Linux │               │  │  • Rule-based            │    │
│ (Attacker)  │               │  │  • Correlation           │    │
│             │               │  └──────────┬───────────────┘    │
│ • Nmap      │               │             │                    │
│ • Hydra     │               │  ┌──────────▼───────────────┐    │
└─────────────┘               │  │  Alert & Dashboard       │    │
│  │  • Kibana                │  │                          |    |
│  │  • Telegram Bot          │  │                          |    |
│  └──────────────────────────┘  │__________________________|    |
└-----------------------------------------------------------------

---

#Hướng dẫn cài đặt

#Yêu cầu môi trường

- Ubuntu 22.04 LTS (SIEM Server + Victim Host)
- Kali Linux (Attacker)
- RAM: SIEM Server tối thiểu 4GB (Elasticsearch cần ≥ 2GB)
- Python 3.10+

#Bước 1 — Clone repository

```bash
git clone https://github.com/anubis-0x404/Siem-project.git
cd Siem-project
```

#Bước 2 — Cài thư viện Python

```bash
pip install -r requirements.txt
```

#Bước 3 — Cài đặt Elasticsearch và Kibana

```bash
wget -qO- https://artifacts.elastic.co/GPG-KEY-elasticsearch \
  | sudo gpg --dearmor -o /usr/share/keyrings/elasticsearch-keyring.gpg

echo "deb [signed-by=/usr/share/keyrings/elasticsearch-keyring.gpg] \
https://artifacts.elastic.co/packages/8.x/apt stable main" \
  | sudo tee /etc/apt/sources.list.d/elastic-8.x.list

sudo apt update && sudo apt install elasticsearch kibana -y
```

Cấu hình Elasticsearch (`/etc/elasticsearch/elasticsearch.yml`):
```yaml
cluster.name: siem-cluster
node.name: node-1
network.host: localhost
http.port: 9200
discovery.type: single-node
xpack.security.enabled: false
path.data: /var/lib/elasticsearch
path.logs: /var/log/elasticsearch
```

```bash
sudo systemctl enable elasticsearch kibana
sudo systemctl start elasticsearch kibana
```

#Bước 4 — Cấu hình file `.env`

```bash
cp .env.example .env
nano .env
```
Điền thông tin:
ES_HOST=http://localhost:9200
TELEGRAM_BOT_TOKEN=your_bot_token_here
TELEGRAM_CHAT_ID=your_chat_id_here

#Bước 5 — Cài đặt Suricata trên Victim Host

```bash
sudo apt install suricata -y
sudo suricata-update
sudo systemctl enable suricata
sudo systemctl start suricata
```
#Bước 6 — Cấu hình Rsyslog
Trên Victim Host (/etc/rsyslog.d/60-forward-siem.conf):
```bash
module(load="imfile")

input(type="imfile"
      File="/var/log/suricata/eve.json"
      Tag="suricata-eve:"
      Severity="info"
      Facility="local0")

auth,authpriv.*    @@<SIEM_SERVER_IP>:514
local0.*           @@<SIEM_SERVER_IP>:514
```
Trên SIEM Server (/etc/rsyslog.d/10-receive-siem.conf):
```bash
module(load="imtcp")
input(type="imtcp" port="514")

template(name="RawMsgOnly" type="string" string="%msg:::drop-last-lf%\n")

if $syslogfacility-text == 'authpriv' or $syslogfacility-text == 'auth' then {
    action(type="omfile" file="/var/log/remote/victim-auth.log"
           template="RSYSLOG_TraditionalFileFormat")
    stop
}

if $syslogfacility-text == 'local0' then {
    action(type="omfile" file="/var/log/remote/victim-eve.json"
           template="RawMsgOnly")
    stop
}
```
```bash
sudo systemctl restart rsyslog
sudo ufw allow 514/tcp
```

#Bước 7 — Khởi động hệ thống
```bash
cd ~/siem_project
python3 main.py
```

#Giao diện điều khiển
_____ _____ _____ __  __             
  / ____|_   _|  ___|  \/  |      (_)    (_)
 | (___   | | | |__ | \  / |      
  \___ \  | | |  __|| |\/| |     
  ____) |_| |_| |___| |  | |     
 |_____/|_____|_____|_|  |_|    

  Mini SIEM — Rule-based Detection & Event Correlation
  Version: 1.0

[1] Kiem tra ket noi Elasticsearch
[2] Tao/cap nhat Index Template
[3] Nap log moi vao Elasticsearch (ingest)
[4] Chay Rule-based Detection (1 lan)
[5] Chay Correlation Detection (1 lan)
[6] Chay 1 chu ky Detection day du
[7] Chay giam sat lien tuc (--schedule)
[8] Test gui canh bao Telegram
[9] Mo Kibana Dashboard
[0] Thoat

Kibana Dashboard: http://<SIEM_SERVER_IP>:5601
Dashboard: SIEM — Overview