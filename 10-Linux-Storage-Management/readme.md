# 💾 Linux Depolama & Dosya Sistemi Yönetimi

Bu belge, loop device ile sanal disk oluşturma, bölümleme (`fdisk`), biçimlendirme (`mkfs.ext4`), ve `/etc/fstab` aracılığıyla kalıcı mount yapılandırmasını kapsar. Bu fazda **iki gerçek hata** debug edildi ve bir "yedek al, hatadan geri dön" senaryosu yaşandı — orijinal repo'nun da vurguladığı, `fstab` hatalarının sistemi **emergency mode**'a düşürebilme riski nedeniyle özellikle dikkatli ilerlendi.

---

## 1. Sanal Disk Oluşturma (Loop Device)

```bash
sudo dd if=/dev/zero of=/tmp/test_disk.img bs=1M count=1024
```
**Çıktı:** `1073741824 bytes (1,1 GB, 1,0 GiB) copied`

```bash
sudo losetup -fP /tmp/test_disk.img
```

### 🐛 Hata & Çözüm

İlk denemede `-fP` yerine yanlışlıkla `-fp` (küçük p) yazıldı: `losetup: geçersiz seçenek -- 'p'`. Büyük **P** ile (partition scanning etkinleştirmek için gerekli) tekrar denendi ve başarılı oldu.

```bash
lsblk
```
**Sonuç:** `loop0  7:0  0  1G  0  loop` — 1G'lık, henüz bölümlenmemiş yeni cihaz doğrulandı.

---

## 2. Bölümleme (`fdisk`)

```bash
sudo fdisk /dev/loop0
```

`fdisk`, **etkileşimli bir menü** açar — `w` (write) tuşuna basılana kadar hiçbir şey diske yazılmaz, `q` ile her an güvenle iptal edilebilir.

### Kullanılan Adımlar

| Tuş | Anlamı |
|-----|--------|
| `n` | new — yeni bölüm oluştur |
| _(Enter)_ | Partition type: varsayılan `p` (primary) kabul edildi |
| _(Enter, Enter, Enter)_ | Partition number, first sector, last sector — varsayılanlar kabul edildi (diskin tamamı kullanıldı) |
| `w` | write — değişiklikleri diske yaz ve çık |

**Sonuç:** `Created a new partition 1 of type 'Linux' and of size 1023 MiB.` → `The partition table has been altered.`

`lsblk` ile doğrulandığında `loop0` altında `loop0p1` alt bölümü görüldü.

---

## 3. Dosya Sistemi Oluşturma

```bash
sudo mkfs.ext4 /dev/loop0p1
```

**Çıktı:** `Filesystem UUID: d0294aa3-7748-4525-b3d9-319a617b0f74` — bu UUID, ilerleyen `fstab` adımında kullanıldı.

---

## 4. Mount Etme

```bash
sudo mkdir -p /mnt/test_storage
sudo mount /dev/loop0p1 /mnt/test_storage
```

### 🐛 Hata & Çözüm: Race Condition

İlk denemede kernel logunda ilginç bir sıralama görüldü: `mounted filesystem...` hemen ardından `unmounting filesystem...`, sonra `mount point does not exist` hatası. Muhtemelen `mkdir` ve `mount` komutları çok hızlı ardışık çalıştırılınca bir senkronizasyon sorunu (race condition) oluştu.

**Çözüm:** Her komut **ayrı ayrı, birer birer** (aralarında doğrulama yaparak) çalıştırıldı:
```bash
ls -ld /mnt/test_storage    # mount noktasının var olduğu doğrulandı
sudo mount /dev/loop0p1 /mnt/test_storage
df -h /mnt/test_storage
```
**Sonuç:** `/dev/loop0p1  989M  24K  922M  1%  /mnt/test_storage` — başarıyla mount edildi.

---

## 5. UUID ile Kalıcı Mount Yapılandırması (`/etc/fstab`)

`/dev/loop0p1` gibi cihaz yolları yeniden başlatmalar arasında **değişebilir**, bu yüzden `/etc/fstab`'da her zaman **UUID** kullanılır — UUID, bölümün değişmeyen, benzersiz kimliğidir.

```bash
sudo blkid /dev/loop0p1
```
**Çıktı:** `UUID="d0294aa3-7748-4525-b3d9-319a617b0f74"` — `mkfs.ext4` çıktısıyla eşleştiği doğrulandı.

### Güvenlik Önlemi: Yedek Alma

```bash
sudo cp /etc/fstab /etc/fstab.backup
```

### 🐛 Hata & Çözüm: Editör Eksikliği + Satır Bölünmesi

Rocky Linux minimal kurulumda **hem `vim` hem `nano` kurulu değildi** (`komut bulunamadı`). Editöre gerek kalmadan, `echo | sudo tee -a` ile doğrudan dosyanın sonuna satır eklenmeye çalışıldı:

```bash
echo "UUID=d0294aa3-... /mnt/test_storage ext4 defaults 0 2" | sudo tee -a /etc/fstab
```

Ancak tırnak kullanımı yüzünden terminal `>` işaretiyle devam etti ve satır **ikiye bölündü**:
```
UUID=d0294aa3-7748-4525-b3d9-319a617b0f74 /mnt/test_storage ext4
defaults 0 2
```

**Çözüm süreci:**
1. `sudo cp /etc/fstab.backup /etc/fstab` ile dosya temiz haline geri döndürüldü.
2. Satır bu sefer **tırnak kullanmadan, tek satırda** eklendi (tırnak olmayınca satır sonu belirsizliği ortadan kalkar):
```bash
echo UUID=d0294aa3-7748-4525-b3d9-319a617b0f74 /mnt/test_storage ext4 defaults 0 2 | sudo tee -a /etc/fstab
```
3. `cat /etc/fstab` ile satırın tek satırda, doğru formatta eklendiği doğrulandı.

### 🧠 Öğrenilen Ders

Kritik bir sistem dosyasını değiştirmeden önce **yedek almak**, hata olduğunda saniyeler içinde güvenli şekilde geri dönmeyi sağladı — bu, gerçek bir prodüksiyon ortamında hayat kurtaran bir alışkanlık.

---

## 6. `fstab` Girişini Güvenli Şekilde Test Etme

```bash
sudo umount /mnt/test_storage
sudo mount -a
```

**Sonuç:** Hatasız tamamlandı — `EXT4-fs (loop0p1): mounted filesystem ... r/w with ordered data mode.`

```bash
sudo systemctl daemon-reload
```
(Uyarı mesajı: "your fstab has been modified, but systemd still uses the old version" — bu komutla systemd'nin fstab önbelleği güncellendi.)

### 🔍 Bu Adım Neden Kritik

`/etc/fstab`'daki bir yazım hatası veya yanlış UUID **o an değil, bir sonraki yeniden başlatmada** sorun çıkarır. Sistem boot sırasında fstab'daki her şeyi otomatik mount etmeye çalışır — hatalı bir giriş sistemi **emergency mode**'a düşürebilir, düzeltmek için recovery console üzerinden root şifresiyle manuel müdahale gerekir.

`mount -a`, fstab'ı **sistem çalışırken, hemen şimdi** yeniden okur. Hata varsa anında gösterir, yerinde düzeltilebilir. Temiz çalışırsa, "sunucuyu yeniden başlattım ve artık açılmıyor" senaryosundan tamamen kaçınılmış olur.

---

## 🔬 `/etc/fstab` Alan Yapısı

| Alan | Örnek Değer | Anlamı |
|------|-------------|--------|
| 1 — Cihaz Tanımlayıcı | `UUID=d0294aa3-...` | Bölümü benzersiz kimliğiyle tanımlar |
| 2 — Mount Noktası | `/mnt/test_storage` | Bölümün nereye bağlanacağı |
| 3 — Dosya Sistemi Türü | `ext4` | |
| 4 — Mount Parametreleri | `defaults` | Standart seçenekler (okuma/yazma, boot'ta otomatik mount) |
| 5 — Dump Yedekleme | `0` | Eski `dump` aracını devre dışı bırakır |
| 6 — FSCK Sırası | `2` | Root (`/`) önce (`1`), bu bölüm sonra (`2`) kontrol edilir |

---

## 📊 Komut Referansı

| Araç | Katman | Örnek | Amacı |
|------|--------|-------|-------|
| **`losetup -fP`** | Blok | `losetup -fP dosya.img` | Bir dosyayı loop device olarak bağlar, partition taramasını etkinleştirir |
| **`lsblk`** | Blok | `lsblk` | Block device'ları ağaç görünümünde gösterir |
| **`fdisk`** | Bölümleme | `sudo fdisk /dev/loop0` | Etkileşimli bölüm editörü |
| **`mkfs.ext4`** | Dosya Sistemi | `mkfs.ext4 /dev/loop0p1` | Bölümü ext4 ile biçimlendirir |
| **`blkid`** | Metadata | `blkid /dev/loop0p1` | UUID ve dosya sistemi türünü gösterir |
| **`mount -a`** | Çalışma Zamanı | `sudo mount -a` | fstab'daki her şeyi mount eder — yeniden başlatmadan önce güvenli test yöntemi |
| **`systemctl daemon-reload`** | Sistem | `sudo systemctl daemon-reload` | systemd'nin fstab değişikliğini tanımasını sağlar |

---

## 🧠 Quiz (Gerçek Sonuçlar)

| # | Soru | Cevap | Sonuç |
|---|------|-------|-------|
| 1 | `mount -a` çalıştırmanın güvenlik açısından önemi neydi? | fstab hatası yeniden başlatana kadar fark edilmeyebilir ve emergency mode'a düşürebilir; `mount -a` bunu şimdi test etmeyi sağlar | ✅ Doğru |
| 2 | fstab'da cihaz yolu yerine neden UUID kullanılır? | Cihaz yolları (`/dev/loop0p1`) yeniden başlatmalar arasında değişebilir; UUID bölümün değişmeyen, benzersiz kimliğidir | ✅ Doğru |

---

ℹ️ _Tüm komutlar yerel Rocky Linux VM'inde (Rocky9-Test) test edilmiştir. Bu fazda 3 gerçek hata (losetup flag, mount race condition, fstab satır bölünmesi) debug edilmiş ve bir yedekten geri dönüş senaryosu yaşanmıştır._
