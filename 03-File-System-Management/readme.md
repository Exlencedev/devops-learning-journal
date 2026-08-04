# 📁 Dosya Sistemi Yönetimi & Depolama Tanılaması

Bu belge, `dd` ile test dosyası oluşturma, disk kullanımı analizi ve en büyük dosyaları bulma pipeline'ını kapsıyor.

---

## 1. Test Dosyası Oluşturma (`dd`)

```bash
mkdir -p /tmp/disk-test && cd /tmp/disk-test
dd if=/dev/zero of=test_file.img bs=1G count=2
```

**Çıktı:**
```
2+0 records in
2+0 records out
2147483648 bytes (2,1 GB, 2,0 GiB) copied, 6,30421 s, 341 MB/s
```

### 🔍 Neden `dd`?

`dd`, `/dev/zero`'dan veri akışı yazarak dosyayı **gerçekten** diske yazar — bu yüzden `du` ile gerçek boyutu doğru görünür. Alternatif olan `fallocate` ise fiziksel I/O yapmadan blokları anında rezerve eder ("mış gibi yapar"), bazı durumlarda `du` gerçek blok boyutunu sıfır gösterebilir.

**Not:** Orijinal müfredatta 10 GB'lık dosya oluşturuluyor. VM diskimin 20 GB olması sebebiyle riski azaltmak için 2 GB (`count=2`) ile devam edildi.

### 🐛 Hata & Çözüm

İlk denemede `bs=1G` yerine yanlışlıkla `bs=16` yazıldı, bu da sadece 160 byte'lık bir dosya oluşturdu (10 GB yerine). Komut dikkatlice tekrar yazılarak (`bs=1G count=2`) düzeltildi.

---

## 2. Beklenmedik Bulgu: XFS Speculative Preallocation

`dd` ile tam 2 GB yazıldığı bilinmesine rağmen:

```bash
du -sh test_file.img
```
**Çıktı:** `3,7G test_file.img`

Bu tutarsızlığı araştırmak için:

```bash
stat test_file.img
```

**Çıktı (özet):**
```
Boyut: 2147483648   (gerçek veri: tam 2 GB)
Bloklar: 7650944    (7650944 × 512 byte ≈ 3.65 GB)
```

### 🔍 Açıklama

Rocky Linux'un varsayılan dosya sistemi **XFS**, büyüyen dosyalar için performans amacıyla gerçekte kullanılandan **fazladan blok ayırır** ("speculative preallocation") — dosya ileride büyürse hazır bulunsun diye. Bu yüzden `du` ve `ls -lh`'nin "toplam blok" hesabı, dosyanın gerçek içeriğinden (2 GB) daha büyük görünür (3.7 GB). Bu bir disk sızıntısı değil, XFS'in normal bir davranışı.

**Pratik önemi:** Gerçek bir sunucuda disk alanı azaldığında `du` çıktısının bazen gerçek veriden fazla görünmesi, XFS kullanan sistemlerde bu yüzden olabilir — panik yapmadan önce bu davranış akılda tutulmalı.

---

## 3. En Büyük 10 Dosyayı Bulma

```bash
sudo find / -type f -exec du -ah {} + 2>/dev/null | sort -rh | head -n 10
```

**Örnek Çıktı:**
```
2,0G   /tmp/disk-test/test_file.img
220M   /usr/lib/locale/locale-archive
79M    /boot/initramfs-...rescue-...img
55M    /usr/lib/sysimage/rpm/rpmdb.sqlite
41M    /boot/initramfs-...kdump.img
...
```

### Pipeline Adım Adım

1. **`find / -type f`** — kök dizinden başlayarak tüm normal dosyaları tarar.
2. **`-exec du -ah {} +`** — bulunan her dosyayı `du`'ya gönderir, insan tarafından okunabilir boyut gösterir.
3. **`2>/dev/null`** — izin hatalarını (permission denied) gizler, çıktıyı temiz tutar.
4. **`sort -rh`** — büyükten küçüğe sıralar (`-h`: "2G" gibi insan-okunabilir boyutları doğru anlar).
5. **`head -n 10`** — sadece ilk 10 sonucu gösterir.

### 🐛 Küçük Not

Komuttaki `{}` karakterlerini Türkçe Q klavyede yazmak için **AltGr + 7** (`{`) ve **AltGr + 0** (`}`) kombinasyonu kullanıldı.

---

## 4. Temizlik

```bash
rm test_file.img
df -h /
```

Test dosyası silinerek disk eski durumuna döndürüldü.

---

## 📊 Komut Referansı

| Komut | Amacı | Örnek | Önemli Flag'ler |
|-------|-------|-------|------------------|
| **`dd`** | Ham veri akışı ile dosya/disk yazar | `dd if=/dev/zero of=file bs=1G count=2` | `bs`: blok boyutu, `count`: blok sayısı |
| **`stat`** | Dosyanın detaylı meta verisini gösterir | `stat file.img` | Boyut vs. Blok farkını gösterir |
| **`du`** | Dosya/dizin disk kullanımını gösterir | `du -sh /var` | `-sh`: özet, insan okunabilir |
| **`df`** | Disk bölümü kullanımını gösterir | `df -h /` | `-h`: insan okunabilir |
| **`find`** | Dizin ağacında arama yapar | `find / -type f` | `-type f`: sadece dosyalar |
| **`sort -rh`** | İnsan-okunabilir boyutlara göre ters sıralar | `sort -rh` | `-r`: ters, `-h`: human-readable |

---

ℹ️ _Tüm adımlar yerel Rocky Linux VM'inde test edilmiştir._
