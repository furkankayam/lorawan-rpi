# BasicStation Kurulum Rehberi

> 🐳 BasicStation · Docker · ChirpStack v4
> Raspberry Pi + WM1302 üzerinde Docker tabanlı BasicStation kurulumu ve AWS ChirpStack entegrasyonu.

---

## Mimari

```
[LilyGO T3S3]  --RF 868MHz-->  [WM1302 Concentrator]  --SPI-->  [BasicStation Docker]
                                                                          |
                                                                   WebSocket (LNS)
                                                                          |
                                                              [Gateway Bridge :3001]
                                                                          |
                                                                   MQTT :1883
                                                                          |
                                                               [ChirpStack v4]
                                                              /              \
                                                       [PostgreSQL]        [Redis]
                                                                          |
                                                              [Spring Boot Backend]
                                                                     gRPC (API Key)
                                                              [React Frontend]
                                                                  REST/JSON
```

---

## 1. Sistem Güncelleme

```bash
sudo apt update && sudo apt upgrade -y
```

---

## 2. Docker Kurulumu

Resmi convenience script — ARM mimarisi otomatik desteklenir.

```bash
curl -fsSL https://get.docker.com -o get-docker.sh
sudo sh get-docker.sh
```

> ℹ️ Raspberry Pi OS ARM mimarisini destekler. Script doğru paketi otomatik indirir, ekstra adım gerekmez.

---

## 3. Docker Compose Kurulumu

v2 — Plugin olarak kurulum.

```bash
sudo apt install -y docker-compose-plugin
```

Doğrulama:

```bash
docker compose version
```

---

## 4. Kullanıcı & Servis Ayarları

### a) Docker grubuna ekle (sudo'suz kullanım)

```bash
sudo usermod -aG docker $USER
```

> ⚠️ Değişikliğin aktif olması için **yeniden giriş** yap ya da `newgrp docker` çalıştır.

### b) Systemd ile otomatik başlatma

```bash
sudo systemctl enable docker
sudo systemctl enable containerd
sudo systemctl start docker
sudo systemctl status docker
```

---

## 5. BasicStation Kurulumu

Pi'de Docker Compose ile çalıştırma.

### 5.1 Çalışma dizini oluştur

```bash
mkdir lora-gateway && cd lora-gateway
nano docker-compose.yml
```

### 5.2 docker-compose.yml

```yaml
services:
  basicstation:
    image: xoseperez/basicstation:latest
    container_name: basicstation
    restart: unless-stopped
    privileged: true
    network_mode: host
    environment:
      MODEL: "WM1302"
      INTERFACE: "SPI"
      RESET_GPIO: 17
      TC_URI: "ws://192.168.1.37:3001"
      USE_CUPS: 0
      TC_KEY: "none"
      GATEWAY_EUI_SOURCE: "chip"
    devices:
      - "/dev/spidev0.0:/dev/spidev0.0"
```

### 5.3 Gateway EUI Kodunu Öğren

ChirpStack paneline gateway eklemek için gerekli. Cihazı deploy etmeden önce şu komutla al:

```bash
docker run -it --privileged --rm \
  -e GATEWAY_EUI_SOURCE=chip \
  xoseperez/basicstation:latest gateway_eui
```

> 💡 Çıktıdaki **16 haneli kodu** kopyala — sonraki adımda ChirpStack'e kayıt için lazım.

### 5.4 ChirpStack Paneline Kayıt

1. Buluttaki **ChirpStack Web UI**'a git
2. **Gateways → Add Gateway** adımlarını izle
3. **Gateway ID** alanına kopyaladığın EUI kodunu yapıştır
4. **Gateway Profile**'da bölgeyi seç: `EU868`

### 5.5 Başlatma ve İzleme

```bash
docker compose up -d
docker logs -f basicstation
```

Logda görmen gerekenler:

| Log                                   | Anlam                                   |
| ------------------------------------- | --------------------------------------- |
| `[HAL] CoreCell concentrator started` | Modül donanımsal olarak çalıştı ✅      |
| `[LNS] Connected to ws://...`         | Bulut sunucusuna başarıyla bağlandın ✅ |

> 🎉 Her iki satırı da gördüysen gateway aktif ve ChirpStack'e veri akıyor.

<br>
<br>
<br>

---

---

---

<br>
<br>
<br>

# UDP Packet Forwarder Kurulum Rehberi

> 🔧 SX1302 HAL · UDP Packet Forwarder · TTN EU868
> Raspberry Pi 4B + WM1302 üzerinde sx1302_hal derleme, GPIO pin düzeltmeleri ve TTN entegrasyonu.

---

## 1. Sistem Hazırlığı ve Arayüzler

SPI, I2C ve Serial donanım arayüzlerini etkinleştir.

```bash
sudo raspi-config
# Interface Options → SPI        → Yes
# Interface Options → I2C        → Yes
# Interface Options → Serial Port → No (Login Shell), Yes (Hardware)
sudo reboot
```

```bash
sudo apt-get update && sudo apt-get install -y build-essential git
```

---

## 2. SX1302 HAL Derlenmesi

Semtech'in resmi kütüphanesi — Pi üzerinde derleme.

```bash
git clone https://github.com/Lora-net/sx1302_hal
cd /home/pi/sx1302_hal
make clean && make all
```

---

## 3. KRİTİK: reset_lgw.sh Güncellenmesi

> ⚠️ **Neden gerekli?** Standart dokümanlardaki GPIO pin numaraları eski Pi OS offset değerlerine göre yazılmış. `cat /sys/kernel/debug/gpio` ile sistemin gerçek offset değerleri tespit edilip aşağıdaki değerlerle güncellendi.

**Dosya:** `~/sx1302_hal/packet_forwarder/reset_lgw.sh`

| Değişken              | Eski (Default) | Yeni (Pi Sistemi) | Açıklama                    |
| --------------------- | -------------- | ----------------- | --------------------------- |
| `SX1302_RESET_PIN`    | 23             | **529**           | Modülü resetlemek (GPIO17)  |
| `SX1302_POWER_EN_PIN` | 18             | **530**           | Güç vermek (GPIO18)         |
| `SX1261_RESET_PIN`    | 22             | **517**           | LBT / Spectral Scan (GPIO5) |

```bash
cd /home/pi/sx1302_hal/packet_forwarder
cp /home/pi/sx1302_hal/tools/reset_lgw.sh ./
nano reset_lgw.sh
# SX1302_RESET_PIN=529, SX1302_POWER_EN_PIN=530, SX1261_RESET_PIN=517
```

---

## 4. KRİTİK: global_conf.json Güncellenmesi

TTN ile el sıkışma için konfigürasyonu yerelden buluta çek.

**Dosya:** `/home/pi/sx1302_hal/packet_forwarder/global_conf.json.sx1250.EU868`

`gateway_conf` bloğu içinde aşağıdaki 4 parametre güncellendi:

| Parametre          | Eski (Default) | Yeni (TTN)                          |
| ------------------ | -------------- | ----------------------------------- |
| `"gateway_ID"`     | `"AA555A..."`  | **`"0016C001F1540604"`**            |
| `"server_address"` | `"localhost"`  | **`"eu1.cloud.thethings.network"`** |
| `"serv_port_up"`   | `1730`         | **`1700`**                          |
| `"serv_port_down"` | `1730`         | **`1700`**                          |

```bash
nano /home/pi/sx1302_hal/packet_forwarder/global_conf.json.sx1250.EU868
# gateway_ID, server_address, serv_port_up, serv_port_down güncelle
```

---

## 5. Çalıştırma ve Doğrulama

```bash
./lora_pkt_fwd -c global_conf.json.sx1250.EU868
```

> ✅ TTN konsolunda gateway kartı **yeşil (Connected)** görünüyorsa kurulum başarılı.

---

## 6. Systemd Servisi

Reboot sonrası otomatik başlatma.

### 6.1 Servis dosyasını oluştur

```bash
sudo nano /etc/systemd/system/lora-gateway.service
```

```ini
[Unit]
Description=LoRa Packet Forwarder
After=network-online.target
Wants=network-online.target

[Service]
Type=simple
User=root
WorkingDirectory=/home/pi/sx1302_hal/packet_forwarder
ExecStartPre=/home/pi/sx1302_hal/packet_forwarder/reset_lgw.sh start
ExecStart=/home/pi/sx1302_hal/packet_forwarder/lora_pkt_fwd -c global_conf.json.sx1250.EU868
ExecStop=/home/pi/sx1302_hal/packet_forwarder/reset_lgw.sh stop
Restart=on-failure
RestartSec=10

[Install]
WantedBy=multi-user.target
```

### 6.2 Aktif et ve başlat

```bash
sudo systemctl daemon-reload
sudo systemctl enable lora-gateway
sudo systemctl start lora-gateway
```

### 6.3 Durum ve log takibi

```bash
sudo systemctl status lora-gateway
journalctl -u lora-gateway -f
```

> ℹ️ Reboot sonrası servis otomatik olarak başlar. `systemctl enable` bunu garanti eder.

---

## 7. Bağlantı Testleri

### a) Port erişim testi (Windows)

```powershell
Test-NetConnection -ComputerName eu1.cloud.thethings.network -Port 1883
```

### b) MQTT Subscribe — Cihaz uplink dinle

TLS bağlantısı, port 8883:

```bash
mosquitto_sub \
  -h eu1.cloud.thethings.network \
  -p 8883 \
  --capath /etc/ssl/certs \
  -u "furkan-gateway-monitor@ttn" \
  -P "NNSXS.RJ3H3WHD6WN4MWUG4TT7..." \
  -t "v3/furkan-gateway-monitor@ttn/devices/+/up" \
  -v
```

> 🔑 API key'i (`NNSXS....`) TTN konsolundan üretirsin — tam key'i sakla, tekrar görüntülenemez.
