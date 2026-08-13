# 🏗️ Linux Mantıksal Hacim Yönetimi (LVM)

Bu belge, LVM temellerini kapsar: fiziksel hacim (PV), hacim grubu (VG), mantıksal hacim (LV), ve **kesintisiz** (canlı) depolama genişletme.

---

## 1. Neden Düz Bölümler Yerine LVM

`fdisk` ile oluşturulan normal bir bölümün boyutu, oluşturma anında sabittir. Alan dolunca büyütmek zor, riskli ve genelde kesinti gerektirir. LVM ise havuzlama mantığıyla çalışır:

```
Fiziksel Disk(ler) → Hacim Grubu (havuz) → Mantıksal Hacim(ler)
```

Havuzda boş alan olduğu sürece, mevcut bir mantıksal hacim **canlı olarak, kesintisiz** büyütülebilir.

### ⚠️ Ders: `dd` Yerine `fallocate`

Orijinal müfredatta, `dd` ile 50GB'lık bir test dosyası oluşturulmaya çalışılırken host diskinin tamamen dolup VM'in donduğu gerçek bir olay yaşanmış (06. fazdaki soft lockup'a benzer, ama sebep bu sefer disk doluluğu). Bu riskten kaçınmak için baştan **`fallocate`** (anında, veri yazmadan alan rezerve eder) ve **küçük MB boyutları** kullanıldı.

---

## 2. Loop Device'lar Oluşturma

```bash
sudo fallocate -l 500M /tmp/lvm_disk1.img
sudo fallocate -l 200M /tmp/lvm_disk2.img
sudo losetup -fP /tmp/lvm_disk1.img
sudo losetup -fP /tmp/lvm_disk2.img
```

`lsblk` ile doğrulandığında `loop1` (200M) ve `loop2` (500M) yeni cihazlar olarak göründü (`loop0`, 10. fazdan kalma, halen `/mnt/test_storage`'a bağlı).

---

## 3. PV → VG → LV Katmanlarını Kurma

```bash
sudo pvcreate /dev/loop2
sudo vgcreate test_pool /dev/loop2
sudo lvcreate -l 100%FREE -n test_data test_pool
```

Üç komut da hatasız tamamlandı: `Physical volume "/dev/loop2" successfully created.` → `Volume group "test_pool" successfully created` → `Logical volume "test_data" created.`

---

## 4. Biçimlendirme ve Mount

```bash
sudo mkfs.ext4 /dev/test_pool/test_data
sudo mkdir -p /mnt/lvm_test
sudo mount /dev/test_pool/test_data /mnt/lvm_test
```

### 🐛 Hata & Çözüm: Büyük/Küçük Harf Duyarlılığı

İlk denemede `mkfs.ext4 /dev/test_pool/test_Data` yazıldı (büyük **D**), ama `lvcreate` ile oluşturulan hacmin gerçek adı `test_data` (küçük d) idi. Linux, cihaz adlarında büyük/küçük harfe duyarlı olduğu için `mkfs.ext4` "dosya yok" hatası verdi, ardından `mount` da olmayan bir dosya sistemini bağlamaya çalışınca FAT/ISOFS/XFS gibi çeşitli format denemeleri başarısız oldu.

**Çözüm:** `sudo lvs` ile hacmin gerçek adı doğrulandı, doğru küçük harfle (`test_data`) tekrar denendi ve başarılı oldu.

**Sonuç:**
```
/dev/mapper/test_pool-test_data   455M   14K   426M   1%   /mnt/lvm_test
```

---

## 5. Kesintisiz Genişletme (LVM'in Asıl Gücü)

```bash
sudo pvcreate /dev/loop1
sudo vgextend test_pool /dev/loop1
sudo lvextend -l +100%FREE /dev/test_pool/test_data
sudo resize2fs /dev/test_pool/test_data
```

### 🐛 Hata & Çözüm

`vgextend` ilk denemesinde `test_pool` yerine yanlışlıkla **`test_poo`** yazıldı (l harfi eksik): `Volume group "test_poo" not found`. Doğru yazımla anında tekrar denendi.

### Sonuç

```
Size of logical volume test_pool/test_data changed from 496,00 MiB (124 extents) to 692,00 MiB (173 extents).
Filesystem at /dev/test_pool/test_data is mounted on /mnt/lvm_test; on-line resizing required
The filesystem on /dev/test_pool/test_data is now 708608 (1k) blocks long.

/dev/mapper/test_pool-test_data   638M   14K   602M   1%   /mnt/lvm_test
```

### 🔍 Kesintisiz Olduğunun Kanıtı

- Hacim **hiçbir noktada** `umount` edilmedi.
- Kernel logunda görülen `"Filesystem ... is mounted ...; on-line resizing required"` ifadesi, resize işleminin **mount'luyken, canlı** yapıldığını doğruluyor.
- Boyut **455M → 638M** olarak büyüdü, sıfır kesinti ile.

Normal bir `fdisk` bölümünde bu imkansızdı — disk alanı bitince bölümü silip yeniden oluşturmak gerekirdi.

---

## 📊 Komut Referansı

| Komut | Katman | Örnek | Amacı |
|-------|--------|-------|-------|
| **`pvcreate`** | Fiziksel | `sudo pvcreate /dev/loop0` | Bir block device'ı LVM kullanımı için başlatır |
| **`vgcreate`** | Havuzlama | `sudo vgcreate pool_name /dev/loop0` | PV'leri bir hacim grubunda (havuz) birleştirir |
| **`lvcreate`** | Mantıksal | `sudo lvcreate -n data_vol -L 10G pool_name` | Havuzdan kullanılabilir bir hacim oluşturur |
| **`vgextend`** | Havuzlama | `sudo vgextend pool_name /dev/loop1` | Mevcut havuza daha fazla fiziksel alan ekler |
| **`lvextend`** | Mantıksal | `sudo lvextend -l +100%FREE /dev/pool/vol` | Havuzdaki boş alanı kullanarak mantıksal hacmi büyütür |
| **`resize2fs`** | Dosya Sistemi | `sudo resize2fs /dev/pool/vol` | Dosya sistemini yeni hacim boyutuyla eşleştirir |
| **`lvs`** | Sorgu | `sudo lvs` | Mevcut mantıksal hacimleri, tam adlarıyla listeler |

---

## 🧠 Quiz (Gerçek Sonuçlar)

| # | Soru | Cevap | Sonuç |
|---|------|-------|-------|
| 1 | LVM ile normal (fdisk) bölümleme arasındaki temel fark nedir? | Normal bölüm sabit boyutludur, büyütmek zor/riskli ve kesinti gerektirir; LVM havuza yeni disk ekleyip mantıksal hacmi canlı olarak, kesintisiz büyütebilir | ✅ Doğru |
| 2 | Hacmi genişletirken kesintisiz olduğunu nasıl doğru bildin? | Umount etmeden, hacim kullanımdayken `vgextend`+`lvextend`+`resize2fs` ile genişletme yapıldı — sistem loglarında "on-line resizing required" görülerek doğrulandı | ✅ Doğru |

---

ℹ️ _Tüm komutlar yerel Rocky Linux VM'inde (Rocky9-Test) test edilmiştir. Bu fazda 2 gerçek hata (büyük/küçük harf duyarlılığı, `test_poo` yazım hatası) debug edilmiştir. Ders çıkarılan bir host disk doluluğu olayı (orijinal müfredattan) nedeniyle `dd` yerine baştan `fallocate` kullanılmıştır._
