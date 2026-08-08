# ⚙️ Linux İşlem Yönetimi (Process Management)

Bu belge, gerçek zamanlı işlem izleme (`top`), sinyaller (`kill`) ile işlem sonlandırma, ve önceliklendirme (`nice`/`renice`) konularını kapsıyor. Bu fazda, CPU yükü testinin kontrolden çıkıp sistemi geçici olarak kilitlemesiyle **gerçek bir kriz anı** da yaşandı.

---

## 1. İşlem İzleme (`top`)

```bash
dd if=/dev/zero of=/dev/null &
top
```

`dd if=/dev/zero of=/dev/null` komutu, sürekli sıfır byte üretip "çöp kutusuna" (`/dev/null`) yazan, CPU'yu maksimum zorlayan zararsız bir test yükü. Sonundaki `&` işlemi arka planda başlatır.

**`top` çıktısı:**
```
PID   USER  PR  NI  %CPU  %MEM  COMMAND
5889  ege   20  0   90.8  0.1   dd
```

`dd` işlemi listenin tepesinde, tek başına **%90.8 CPU** kullanıyor — diğer tüm sistem işlemleri (systemd, kworker'lar) neredeyse boşta (%0.0).

---

## 2. 🐛 Gerçek Kriz: Kernel Soft Lockup

`top` ile işlem uzun süre izlenirken (birkaç dakika), VM tamamen **tepkisiz** hale geldi — klavye girdileri ekrana yansımadı, fare hareketi görmezden gelindi. Ekranda şu kernel uyarısı belirdi:

```
kernel: watchdog: BUG: soft lockup - CPU#3 stuck for 30s!
```

### 🔍 Sebep Analizi

`dd` işlemi, atanmış bir CPU çekirdeğini (CPU#3) 30 saniyeden uzun süre kesintisiz meşgul etti. Kernel'in "watchdog" mekanizması bunu bir donma riski olarak algılayıp uyardı — bu, VM'in sadece 4 vCPU'ya sahip olması ve `dd`'nin agresif, kesintisiz CPU talebiyle sistemin genel tepkisini bastırmasından kaynaklandı.

### Çözüm İşlemi

1. 30-60 saniye beklendi, sistem kendiliğinden toparlanmadı.
2. VM penceresinden **Power Off** (sert kapatma) ile VM kapatıldı — normal (ACPI) kapatma, sistem tepkisiz olduğu için işe yaramazdı.
3. VM yeniden başlatıldı, `uptime` ile sistemin sağlıklı açıldığı doğrulandı: `up 0 min, load average: 0.25, 0.08, 0.03`
4. `pidof dd` ile eski işlemin tamamen temizlendiği (VM yeniden başladığı için) doğrulandı.

### 🧠 Öğrenilen Ders

Gerçek bir sunucuda, CPU'yu bu şekilde aşırı zorlayan bir işlem sistemi erişilemez hale getirebilir — bu yüzden `kill`/`SIGKILL` gibi acil müdahale araçları, ve işlemleri **başlamadan önce** `nice` ile önceliklendirmek (bkz. Bölüm 4) DevOps'ta kritik önem taşıyor.

---

## 3. Sinyallerle İşlem Sonlandırma (`kill`)

Dersten sonra, testi **kontrollü** şekilde tekrarladık — bu sefer `top` ile uzun süre izlemek yerine, işlemi hızlıca başlatıp PID'sini alıp saniyeler içinde sonlandırdık:

```bash
dd if=/dev/zero of=/dev/null &
pidof dd
# çıktı: 5921 (örnek)

kill -9 5921
pidof dd
# çıktı: (boş) — işlem başarıyla sonlandırıldı
```

### `kill -15` (SIGTERM) vs `kill -9` (SIGKILL)

- **`kill -15` (SIGTERM):** Sürece "lütfen kendini düzgünce kapat" der — işlem, dosyaları kaydetme, bağlantıları kapatma gibi temizlik işlemlerini yapma fırsatı bulur.
- **`kill -9` (SIGKILL):** İşlemi anında ve zorla sonlandırır — hiçbir temizlik yapılmaz, işletim sistemi işlemi derhal siler. `dd` gibi basit test işlemlerinde risk yok, ama gerçek uygulamalarda veri kaybına yol açabilir.

---

## 4. Öncelik Yönetimi (`nice` & `renice`)

Bu bölüm tamamen güvenli test yükleriyle (`sleep`) yapıldı — CPU riski yok.

### Düşük öncelikli işlem başlatma

```bash
nice -n 19 sleep 60 &
ps -eo pid,ni,cmd | grep sleep
```
`NI` sütununda `19` (en düşük öncelik) görüldü.

### Çalışan işlemin önceliğini canlı değiştirme

```bash
pidof sleep
# çıktı: 1953

sudo renice -n 5 -p 1953
```
**Çıktı:** `1953 (process ID) old priority 19, new priority 5`

Doğrulama:
```bash
ps -eo pid,ni,cmd | grep sleep
```
`NI` sütununda artık `5` görüldü — işlem **durdurulmadan**, canlı olarak önceliği artırıldı.

### 🔍 Gerçek Dünya Kullanımı

Bu teknik, örneğin bir sunucuda arka planda çalışan bir yedekleme işini önce düşük öncelikte başlatıp (kritik işlemleri yavaşlatmasın diye), gerekirse sonradan önceliğini artırmak için kullanılır.

---

## 📊 Komut Referansı

| Komut | Amacı | Örnek |
|-------|-------|-------|
| **`top`** | Gerçek zamanlı işlem/kaynak izleme | `top` (çıkış: `q`) |
| **`pidof <isim>`** | Bir işlemin PID'sini isimle bulur | `pidof dd` |
| **`kill -15 <PID>`** | Nazikçe sonlandırma isteği (SIGTERM) | `kill -15 5889` |
| **`kill -9 <PID>`** | Zorla, anında sonlandırma (SIGKILL) | `kill -9 5889` |
| **`nice -n <değer>`** | Yeni işlemi belirli öncelikle başlatır | `nice -n 19 sleep 60` |
| **`renice -n <değer> -p <PID>`** | Çalışan işlemin önceliğini değiştirir | `sudo renice -n 5 -p 1953` |

---

## 🧠 Quiz (Gerçek Sonuçlar)

| # | Soru | Cevap | Sonuç |
|---|------|-------|-------|
| 1 | `kill -15` ile `kill -9` arasındaki fark nedir? | SIGTERM işleme nazikçe kapanma şansı verir (temizlik yapabilir); SIGKILL anında ve zorla sonlandırır, temizlik yapılmaz | ✅ Doğru |
| 2 | VM'in donmasına (soft lockup) ne sebep oldu? | Çok yüksek CPU yüklü bir işlemi uzun süre çalıştırmak, kernel'in "soft lockup" uyarısı vermesine ve sistemin geçici olarak tepkisiz kalmasına yol açtı | ✅ Doğru |

---

ℹ️ _Tüm adımlar yerel Rocky Linux VM'inde (Rocky9-Test) test edilmiştir — bu faz sırasında gerçek bir sistem donması yaşanmış ve başarıyla çözülmüştür._
