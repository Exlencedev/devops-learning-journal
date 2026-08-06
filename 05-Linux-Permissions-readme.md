# 🔑 Linux İzinleri & Güvenlik Sıkılaştırma

Bu belge, dosya izinlerini, `chmod`, sticky bit, sahiplik (`chown`/`chgrp`) ve `umask`'ı kapsıyor.

---

## 1. `ls -l` Çıktısını Okuma

```bash
touch test-script.sh
mkdir test-folder
ls -l
```

**Çıktı:**
```
drwxr-xr-x. 2 ege ege    6 Ağu  3 17:13 test-folder
-rw-r--r--. 1 ege ege    0 Ağu  3 17:13 test-script.sh
```

İlk karakter **tür** göstergesidir (`-` normal dosya, `d` dizin). Geri kalan 9 karakter, 3'erli üç gruba ayrılır (`rwx`) — sahip, grup, diğerleri.

### 🔍 Gözlem

`test-script.sh` adında ".sh" olmasına rağmen çalıştırma izni (`x`) yok — `touch` ile oluşturulan dosyalar varsayılan olarak `666`'dan başlar, `umask` bunu kısar (bkz. Bölüm 5).

---

## 2. `chmod` Sayısal Sistemi

| Sayı | İzin | 
|------|------|
| 4 | okuma (r) |
| 2 | yazma (w) |
| 1 | çalıştırma (x) |

```bash
chmod 750 test-script.sh
ls -l test-script.sh
```

**Çıktı:**
```
-rwxr-x---. 1 ege ege 0 Ağu  3 17:13 test-script.sh
```

`750` = sahip: `7` (rwx), grup: `5` (r-x), diğerleri: `0` (---). Sonuç birebir doğrulandı.

---

## 3. Sticky Bit ile Paylaşımlı Dizin Koruması

Tam yazma erişimine (`777`) sahip paylaşımlı bir klasör güvenlik açığı oluşturur: herhangi bir kullanıcı başkasının dosyasını silebilir. **Sticky bit** bunu çözer.

### Adımlar

```bash
sudo mkdir /tmp/test
sudo chmod 777 /tmp/test
ls -ld /tmp/test
# Çıktı: drwxrwxrwx. 2 root root 6 Ağu  3 17:15 /tmp/test

sudo chmod +t /tmp/test
ls -ld /tmp/test
# Çıktı: drwxrwxrwt. 2 root root 6 Ağu  3 17:15 /tmp/test
```

Sondaki **`t`** aktif sticky bit'i gösteriyor.

### 🔐 Gerçek Test — İki Kullanıcıyla

```bash
# ege kullanıcısı olarak bir dosya oluşturuldu
touch /tmp/test/dosyam.txt

# devopstester kullanıcısına geçildi
sudo su - devopstester
rm /tmp/test/dosyam.txt
```

**Sonuç:** `rm`, silme onayı istedi (`y` yazıldı), ama işlem yine de gerçekleşmedi — `ls -l /tmp/test` ile kontrol edildiğinde dosyanın **hâlâ orada** olduğu doğrulandı.

**Açıklama:** Sticky bit, dizinde yazma izni olan herkesin dosya oluşturabilmesine izin verir, ama **sadece dosya sahibi (veya root)** o dosyayı silebilir/yeniden adlandırabilir — yazma izni tek başına silme yetkisi vermez. `/tmp` dizini gerçek sistemlerde tam olarak bu şekilde korunur.

---

## 4. Sahiplik & Grup Değiştirme (`chown` & `chgrp`)

```bash
sudo chown -R ege /tmp/test
ls -ld /tmp/test
# Çıktı: drwxrwxrwt. 2 ege root 24 Ağu  3 17:17 /tmp/test
```

🔍 **Gözlem:** `chown -R ege` sadece **sahibi** değiştirdi, grup hâlâ `root` olarak kaldı.

```bash
sudo chgrp -R ege /tmp/test
ls -ld /tmp/test
# Çıktı: drwxrwxrwt. 2 ege ege 24 Ağu  3 17:17 /tmp/test
```

`chgrp` ile sadece grup değiştirildi — artık hem sahip hem grup `ege`. (`chown ege:ege` ile ikisi tek komutta da yapılabilirdi.)

---

## 5. Varsayılan İzin Maskeleme (`umask`)

`umask`, yeni dosya/dizinlerin varsayılan iznini, maksimum değerlerden (dosya: `666`, dizin: `777`) çıkararak belirler.

### Test 1 — Varsayılan umask (0022)

```bash
touch umask-test.txt
ls -l umask-test.txt
```
**Çıktı:** `-rw-r--r--` → `666 - 022 = 644` ✅ Doğrulandı.

### Test 2 — Sıkılaştırılmış umask (0077)

```bash
umask 0077
touch hardened.conf
ls -l hardened.conf
```
**Çıktı:** `-rw-------` → `666 - 077 = 600` ✅ Doğrulandı — artık sadece sahip erişebiliyor.

---

## 📊 Komut Referansı

| Komut | Amacı | Örnek | Notlar |
|-------|-------|-------|--------|
| **`chmod`** | Dosya izin bayraklarını ayarlar | `chmod 750 script.sh` | `+t` sticky bit ekler, `-R` özyinelemeli |
| **`chown`** | Sahibi (ve isteğe bağlı grubu) değiştirir | `chown -R ege:ege /tmp/test` | |
| **`chgrp`** | Sadece grubu değiştirir | `chgrp ege app.log` | |
| **`umask`** | Yeni dosya/dizin varsayılan iznini ayarlar | `umask 0077` | Dosyada `666`'dan, dizinde `777`'den çıkarır |

---

## 🧠 Quiz (Gerçek Sonuçlar)

| # | Soru | Cevap | Sonuç |
|---|------|-------|-------|
| 1 | Dizinlerde `x` biti neden önemli? | İçine girme/geçme (cd) iznini kontrol eder — `r` olsa bile `x` olmadan içine girilemez | ✅ Doğru |
| 2 | Sticky bit tam olarak neyi engeller? | Dizindeki dosyaları sadece sahibinin (veya root'un) silebilmesini sağlar, yazma izni olan herkesin değil | ✅ Doğru |

---

ℹ️ _Tüm adımlar yerel Rocky Linux VM'inde (Rocky9-Test) test edilmiştir._
