# 14. Faz - Linux Bash Scripting

Bu fazda Linux ortamında **Bash scripting** kullanarak disk kullanımını kontrol eden basit bir script geliştirdim.

## 🎯 Amaç

Linux sistemindeki `/` disk bölümünün kullanım oranını kontrol etmek ve disk kullanımı `%80` seviyesini geçtiğinde kullanıcıya uyarı vermek.

## 🛠️ Kullanılan Araçlar

- Linux
- Bash
- `df`
- `awk`
- `tr`
- Git

## 📁 Oluşturulan Dosya

```text
disk_check.sh
```

## 📝 Script

```bash
#!/bin/bash

usage=$(df -h / | awk 'NR==2 {print $5}' | tr -d '%')

echo "Disk usage: $usage%"

if [ "$usage" -gt 80 ]; then
    echo "WARNING: Disk usage is above 80%"
else
    echo "Disk usage is normal."
fi
```

## 🔍 Kullanılan Komutlar

### Disk kullanımını kontrol etme

```bash
df -h /
```

Bu komut `/` disk bölümünün toplam, kullanılan ve boş alanını gösterir.

### Script oluşturma

```bash
nano disk_check.sh
```

### Scripti çalıştırılabilir yapma

```bash
chmod +x disk_check.sh
```

### Scripti çalıştırma

```bash
./disk_check.sh
```

### Dosya izinlerini kontrol etme

```bash
ls -l disk_check.sh
```

Scriptin çalıştırılabilir olduğunu gösteren izin:

```text
-rwxr-xr-x
```

## 🧪 Testler

### Normal kullanım testi

Sistemde gerçek disk kullanımı yaklaşık `%12` iken script çalıştırıldı:

```text
Disk usage: 12%
Disk usage is normal.
```

### Uyarı testi

Script içerisinde geçici olarak:

```bash
usage=90
```

kullanılarak `%80` üzerindeki durum simüle edildi.

Sonuç:

```text
Disk usage: 90%
WARNING: Disk usage is above 80%
```

Test tamamlandıktan sonra script tekrar gerçek disk kullanımını okuyacak şekilde düzenlendi.

## 📦 Git İşlemleri

Script Git repository'sine eklendi:

```bash
git init
git status
git add disk_check.sh
git commit -m "Add disk usage check script"
```

Commit işlemi başarıyla gerçekleştirildi.

## 📌 Öğrenilenler

Bu faz ile birlikte:

- Bash script oluşturmayı
- Linux komutlarını script içerisinde kullanmayı
- `df` ile disk kullanımını kontrol etmeyi
- `awk` ile belirli bir sütunu almayı
- `tr` ile `%` karakterini kaldırmayı
- `if/else` ile koşul oluşturmayı
- Bash değişkenlerini kullanmayı
- Scriptlere çalıştırma izni vermeyi
- Git ile dosya ekleme ve commit işlemlerini

öğrendim.

## ✅ Faz Sonucu

Disk kullanımını otomatik olarak kontrol eden ve kullanım `%80` seviyesini geçtiğinde uyarı veren Bash scripti başarıyla oluşturuldu, test edildi ve Git repository'sine commit edildi.
