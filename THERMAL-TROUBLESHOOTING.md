# Isı Kaynakli Kapanma Sorunlarını Tespit Etme / Thermal Shutdown Troubleshooting

## Türkçe

### Sisteminizin Neden Kapandığını Tespit Etme

Laptopunuz sunucu olarak kullanıldığında ısınma nedeniyle kapanıyorsa, aşağıdaki adımları izleyerek bunu doğrulayabilirsiniz.

#### 1. Sistem Loglarını Kontrol Etme

**Son kapanma öncesi sistem loglarını görüntüleme:**
```bash
# Systemd kullanan sistemlerde (Debian 8+)
sudo journalctl -b -1
# En son önceki boot'u gösterir

# Son 100 sistem mesajını göster
sudo journalctl -n 100

# Belirli bir zaman aralığındaki logları göster
sudo journalctl --since "2026-02-15 10:00:00" --until "2026-02-15 15:00:00"
```

#### 2. Kernel Loglarını ve dmesg'i Kontrol Etme

```bash
# Kernel loglarını kontrol et
sudo dmesg | grep -i "thermal\|temperature\|overheat\|critical\|shutdown"

# Kernel ring buffer'daki tüm mesajları göster
sudo dmesg -T

# Geçmiş boot loglarını kontrol et (systemd)
sudo journalctl -k -b -1 | grep -i "thermal\|temperature\|critical"
```

#### 3. Sıcaklık İzleme Araçlarını Kullanma

**lm-sensors kurulumu ve kullanımı:**
```bash
# lm-sensors kurulumu
sudo apt-get update
sudo apt-get install lm-sensors

# Sensörleri tespit et
sudo sensors-detect
# Sorulan sorulara "yes" cevabı verin

# Sıcaklıkları görüntüle
sensors

# Sürekli izleme (her 2 saniyede bir güncelleme)
watch -n 2 sensors
```

**s-tui ile gerçek zamanlı CPU izleme:**
```bash
# s-tui kurulumu
sudo apt-get install s-tui

# Çalıştır
sudo s-tui
```

#### 4. Önemli Log Dosyalarının Konumları

```bash
# Sistem mesajları
sudo less /var/log/syslog
sudo less /var/log/messages  # Bazı dağıtımlarda

# Kernel mesajları
sudo less /var/log/kern.log

# Donanım hataları
sudo less /var/log/dmesg

# Systemd journal
sudo journalctl --list-boots  # Tüm boot'ları listele
```

#### 5. BIOS/Firmware Tarafından Yapılan Kapanmalar

**BIOS tarafından yapılan acil kapanmalar genellikle:**
- Linux loglarına tam olarak yazılmaz (çünkü ani kesinti olur)
- Kernel panic mesajı olmadan ani kapanma olarak görünür
- Son log girişi genellikle yüksek sıcaklık uyarısı içerir

**ACPI event loglarını kontrol edin:**
```bash
# ACPI event logları
sudo cat /var/log/acpid

# ACPI termal zone bilgileri
cat /sys/class/thermal/thermal_zone*/temp
cat /sys/class/thermal/thermal_zone*/type

# Her termal zone için sıcaklık (Celsius cinsinden, 1000'e bölün)
for i in /sys/class/thermal/thermal_zone*/temp; do 
    echo "$i: $(($(cat $i) / 1000))°C"
done
```

#### 6. Kapanma Sebebini Tespit Etme Komutları

```bash
# Son kapatma/yeniden başlatma olaylarını göster
last -x shutdown reboot

# Systemd ile son kapanma sebebini göster
sudo journalctl -b -1 -n 50

# ACPI event'lerini izleme
acpi_listen
# Bu komutu çalıştırın ve sistemi izleyin

# CPU sıcaklığını sürekli loglama (bir dosyaya kaydet)
while true; do 
    echo "$(date): $(sensors | grep 'Core ')" >> /tmp/temp_log.txt
    sleep 60
done
```

#### 7. Önleyici Tedbirler

**Performans ayarları:**
```bash
# CPU frekans ölçeklendirmesini kontrol et
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Power-save moduna geç
echo powersave | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# TLP kurulumu (dizüstü güç yönetimi)
sudo apt-get install tlp tlp-rdw
sudo systemctl enable tlp
sudo systemctl start tlp
```

**Fiziksel bakım:**
- Havalandırma deliklerini ve fanları temizleyin
- Termal macun değişimini düşünün
- Laptop altında hava akışı sağlayacak yükseltici kullanın
- Oda sıcaklığını kontrol edin

#### 8. Thermal Throttling'i İzleme

```bash
# Intel CPU için turbostat (daha detaylı bilgi)
sudo apt-get install linux-tools-common linux-tools-generic
sudo turbostat --interval 5

# CPU throttling event'lerini kontrol et
sudo journalctl | grep -i "thermal\|throttl"
```

### Örnek Analiz Senaryosu

1. **Kapanma sonrası ilk kontrol:**
   ```bash
   sudo journalctl -b -1 | tail -100
   ```
   Son 100 mesajı inceleyin, özellikle "thermal", "temperature", "critical" kelimelerini arayın.

2. **Sıcaklık geçmişi:**
   ```bash
   sudo dmesg -T | grep -i thermal
   ```

3. **Gelecek kapanmaları önleme:**
   - lm-sensors ile sürekli izleme
   - CPU governor'ı powersave'e ayarlama
   - TLP kurulumu
   - Fiziksel temizlik

### Önemli Notlar

- ⚠️ BIOS seviyesinde bir güvenlik kapanması olduğunda, işletim sistemi bunu loglayamayabilir
- 🌡️ Intel i5-1135G7 için maksimum sıcaklık ~100°C'dir
- 🔧 Eğer CPU sürekli 90°C+ sıcaklıklara ulaşıyorsa termal macun değişimi gerekebilir
- 💻 Sunucu olarak kullanım daha fazla ısınmaya neden olur

---

## English

### Identifying Why Your System is Shutting Down

If your laptop is shutting down due to overheating while being used as a server, you can verify this by following these steps.

#### 1. Check System Logs

**View system logs before the last shutdown:**
```bash
# On systems using systemd (Debian 8+)
sudo journalctl -b -1
# Shows the previous boot

# Show last 100 system messages
sudo journalctl -n 100

# Show logs for a specific time range
sudo journalctl --since "2026-02-15 10:00:00" --until "2026-02-15 15:00:00"
```

#### 2. Check Kernel Logs and dmesg

```bash
# Check kernel logs
sudo dmesg | grep -i "thermal\|temperature\|overheat\|critical\|shutdown"

# Show all messages in kernel ring buffer
sudo dmesg -T

# Check previous boot logs (systemd)
sudo journalctl -k -b -1 | grep -i "thermal\|temperature\|critical"
```

#### 3. Use Temperature Monitoring Tools

**lm-sensors installation and usage:**
```bash
# Install lm-sensors
sudo apt-get update
sudo apt-get install lm-sensors

# Detect sensors
sudo sensors-detect
# Answer "yes" to the questions asked

# Display temperatures
sensors

# Continuous monitoring (updates every 2 seconds)
watch -n 2 sensors
```

**Real-time CPU monitoring with s-tui:**
```bash
# Install s-tui
sudo apt-get install s-tui

# Run
sudo s-tui
```

#### 4. Important Log File Locations

```bash
# System messages
sudo less /var/log/syslog
sudo less /var/log/messages  # On some distributions

# Kernel messages
sudo less /var/log/kern.log

# Hardware errors
sudo less /var/log/dmesg

# Systemd journal
sudo journalctl --list-boots  # List all boots
```

#### 5. BIOS/Firmware-Initiated Shutdowns

**Emergency shutdowns by BIOS typically:**
- Are not fully logged to Linux logs (because of sudden power cut)
- Appear as sudden shutdowns without kernel panic messages
- Last log entry usually contains high temperature warnings

**Check ACPI event logs:**
```bash
# ACPI event logs
sudo cat /var/log/acpid

# ACPI thermal zone information
cat /sys/class/thermal/thermal_zone*/temp
cat /sys/class/thermal/thermal_zone*/type

# Temperature for each thermal zone (in Celsius, divide by 1000)
for i in /sys/class/thermal/thermal_zone*/temp; do 
    echo "$i: $(($(cat $i) / 1000))°C"
done
```

#### 6. Commands to Identify Shutdown Cause

```bash
# Show last shutdown/reboot events
last -x shutdown reboot

# Show last shutdown reason with systemd
sudo journalctl -b -1 -n 50

# Monitor ACPI events
acpi_listen
# Run this command and monitor your system

# Continuously log CPU temperature (save to file)
while true; do 
    echo "$(date): $(sensors | grep 'Core ')" >> /tmp/temp_log.txt
    sleep 60
done
```

#### 7. Preventive Measures

**Performance settings:**
```bash
# Check CPU frequency scaling
cat /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Switch to power-save mode
echo powersave | sudo tee /sys/devices/system/cpu/cpu*/cpufreq/scaling_governor

# Install TLP (laptop power management)
sudo apt-get install tlp tlp-rdw
sudo systemctl enable tlp
sudo systemctl start tlp
```

**Physical maintenance:**
- Clean ventilation holes and fans
- Consider thermal paste replacement
- Use a laptop stand for better airflow
- Control room temperature

#### 8. Monitor Thermal Throttling

```bash
# turbostat for Intel CPU (more detailed info)
sudo apt-get install linux-tools-common linux-tools-generic
sudo turbostat --interval 5

# Check CPU throttling events
sudo journalctl | grep -i "thermal\|throttl"
```

### Example Analysis Scenario

1. **First check after shutdown:**
   ```bash
   sudo journalctl -b -1 | tail -100
   ```
   Review the last 100 messages, especially looking for "thermal", "temperature", "critical" keywords.

2. **Temperature history:**
   ```bash
   sudo dmesg -T | grep -i thermal
   ```

3. **Prevent future shutdowns:**
   - Continuous monitoring with lm-sensors
   - Set CPU governor to powersave
   - Install TLP
   - Physical cleaning

### Important Notes

- ⚠️ When a BIOS-level safety shutdown occurs, the OS may not be able to log it
- 🌡️ Maximum temperature for Intel i5-1135G7 is ~100°C
- 🔧 If CPU consistently reaches 90°C+ temperatures, thermal paste replacement may be needed
- 💻 Server usage causes more heat generation

---

## Additional Commands for Detailed Analysis

### Check Hardware Information
```bash
# CPU information
lscpu

# Hardware information
sudo dmidecode -t processor
sudo dmidecode -t memory

# PCI devices (including temperature sensors)
lspci -v
```

### Create a Thermal Monitoring Script

Create a file `/usr/local/bin/thermal-monitor.sh`:
```bash
#!/bin/bash
LOG_FILE="/var/log/thermal-monitor.log"

while true; do
    TIMESTAMP=$(date '+%Y-%m-%d %H:%M:%S')
    TEMPS=$(sensors | grep -E 'Core|temp' || echo "Sensors not available")
    LOAD=$(uptime | awk -F'load average:' '{print $2}')
    
    echo "$TIMESTAMP | Load: $LOAD | $TEMPS" >> $LOG_FILE
    
    # Check if temperature is critical (above 85°C)
    CRITICAL=$(sensors | grep -oP '(?<=\+)\d+(?=\.\d°C)' | awk '$1 > 85 {print $1}')
    if [ ! -z "$CRITICAL" ]; then
        echo "$TIMESTAMP | WARNING: Critical temperature detected: $CRITICAL°C" >> $LOG_FILE
        # You can add notification here
    fi
    
    sleep 60
done
```

Make it executable and run:
```bash
sudo chmod +x /usr/local/bin/thermal-monitor.sh
sudo /usr/local/bin/thermal-monitor.sh &
```

Or create a systemd service for it.

### Useful Aliases

Add to your `~/.bashrc`:
```bash
alias temps='watch -n 2 sensors'
alias cpufreq='watch -n 2 "cat /proc/cpuinfo | grep MHz"'
alias thermal='cat /sys/class/thermal/thermal_zone*/temp | while read temp; do echo "scale=2; $temp/1000" | bc; done'
```
