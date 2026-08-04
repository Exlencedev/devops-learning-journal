# 🐧 Linux Temelleri: Sistem Kimliği & Metin İşleme

Bu belge, temel sistem kimliği komutlarını ve pipe (`|`) kullanarak metin işleme (grep, cut, awk) pratiğini kapsıyor.

---

## 1. Sistem Kimliği Komutları

```bash
hostname
hostnamectl
hostname -I
uname -a
```

### 🔍 Gözlemler

- `hostname` sadece düz makine adını (`localhost`) gösterirken, `hostnamectl` çok daha fazla detay verir: OS, kernel, mimari, sanallaştırma tipi, donanım bilgisi.
- `hostnamectl` çıktısında `Static hostname: (unset)` görüldü — sisteme kalıcı bir isim atanmamış, sadece geçici (`Transient hostname: localhost`) bir isim var.
- `hostname -I` iki IP adresi döndürdü: `10.0.2.15` (IPv4) ve `fd17:625c:f037:2:a00:27ff:feb6:f012` (IPv6).
- `uname -a` çıktısında `localhost.localdomain` FQDN'i görüldü — Rocky Linux'ta varsayılan davranış.

---

## 2. Metin İşleme Pipeline'ı: Dağıtım Adını Çıkarma

Amaç: `/etc/os-release` dosyasından, sadece dağıtım adını temiz bir şekilde çıkarmak.

```bash
cat /etc/os-release | grep "PRETTY_NAME" | cut -d '=' -f 2 | tr -d '"'
```

**Çıktı:**
```
Rocky Linux 10.2 (Red Quartz)
```

### Pipeline Adım Adım

1. **`cat /etc/os-release`** — dosyanın tüm ham içeriğini okur.
2. **`grep "PRETTY_NAME"`** — sadece bu satırı filtreler: `PRETTY_NAME="Rocky Linux 10.2 (Red Quartz)"`
3. **`cut -d '=' -f 2`** — `=` işaretine göre böler, ikinci parçayı alır: `"Rocky Linux 10.2 (Red Quartz)"`
4. **`tr -d '"'`** — tırnak işaretlerini siler, nihai temiz sonucu verir.

### 🐛 Hata & Çözüm

İlk denemede `cat` yerine yanlışlıkla `car` yazıldı:
```
bash: car: komut yok
```
Sebep basit bir yazım hatası (harf kayması). Linux komutları harf hassasiyetine çok duyarlı — tek harflik hata bile "command not found" hatasına yol açar. Doğru yazımla (`cat`) tekrar denendi ve sorun çözüldü.

---

## 3. Metin İşleme Pipeline'ı: Disk Kullanımını Formatlama

```bash
df -h / | awk 'NR==2 {print "Total: " $2 " | Used: " $3 " | Free: " $4}'
```

**Çıktı:**
```
Total: 16G | Used: 1,5G | Free 15G
```

### Açıklama

`awk 'NR==2 {...}'` ifadesi, `df -h /` çıktısının **2. satırını** (başlık satırı değil, gerçek veri satırı) hedef alır ve `$2`, `$3`, `$4` sütunlarını (Boyut, Dolu, Boş) özel bir formatta yeniden yazdırır.

### 🐛 Hata & Çözüm

İlk denemede pipe (`|`) karakteri Türkçe klavye düzeninde doğru basılamadı, komut hatalı çalıştı (`awk: Böyle bir dosya ya da dizin yok`). Türkçe Q klavyede `|` karakteri **AltGr + <** ile yazılır. Doğru tuş kombinasyonuyla tekrar denendiğinde komut başarıyla çalıştı.

---

## 📊 Komut Referansı

| Komut | Amacı | Örnek |
|-------|-------|-------|
| **`hostname`** | Makine adını gösterir | `hostname` |
| **`hostnamectl`** | Detaylı sistem/OS/kernel bilgisi gösterir | `hostnamectl` |
| **`uname -a`** | Kernel ve mimari bilgisi gösterir | `uname -a` |
| **`grep`** | Metinde satır bazlı arama/filtreleme yapar | `grep "PRETTY_NAME" file` |
| **`cut -d -f`** | Belirteç bazlı sütun ayıklama yapar | `cut -d '=' -f 2` |
| **`awk`** | Satır/sütun bazlı gelişmiş metin işleme yapar | `awk '{print $1}'` |
| **`tr -d`** | Belirtilen karakterleri siler | `tr -d '"'` |
| **`\|`** | Bir komutun çıktısını diğerine "besler" (pipe) | `cmd1 \| cmd2` |

---

ℹ️ _Tüm komutlar yerel Rocky Linux VM'inde test edilmiştir._
