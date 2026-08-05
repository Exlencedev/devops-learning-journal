# 🔐 Kullanıcı & Yetki Yönetimi (En Düşük Yetki Prensibi)

Bu belge, kısıtlı bir kullanıcı oluşturup `visudo` ile "En Düşük Yetki Prensibi"ni (Principle of Least Privilege) uygulama sürecini kapsıyor. Amaç: bir kullanıcının sadece belirli bir servisi (nginx) yönetebilmesini sağlamak, sistemin geri kalanına dokunamamasını garanti etmek.

---

## 1. Test Servisi Kurulumu (nginx)

```bash
sudo dnf install nginx -y
sudo systemctl start nginx
sudo systemctl status nginx
```

`Active: active (running)` durumu doğrulandı, config sözdizimi kontrolü de başarılı çıktı (`nginx: configuration file /etc/nginx/nginx.conf test is successful`).

---

## 2. Kısıtlı Kullanıcı Oluşturma

```bash
sudo useradd -m devopstester
sudo passwd devopstester
```

### 🐛 Hata & Çözüm

İlk şifre denemesi reddedildi:
```
KÖTÜ PAROLA: Parola sözlük denetiminden geçmedi - bu parola çok basit/sistematik
```
Ardından iki şifre girişi birbirini tutmadı (`parolalar birbirine uymuyor`), işlem `password unchanged` ile sonuçlandı. Daha karmaşık (harf+rakam+sembol karışık) bir şifre ile, iki girişi de dikkatlice eşleştirerek tekrar denendi ve başarılı oldu.

---

## 3. Komutun Tam Yolunu Doğrulama

```bash
which systemctl
```
**Çıktı:** `/usr/bin/systemctl`

Bu tam yol, sudoers kuralında **mutlaka** kullanılmalı — göreli yol (`systemctl`) güvenlik açığına yol açabilir.

---

## 4. `visudo` ile Kısıtlı Yetki Tanımlama

```bash
sudo visudo
```

Dosyanın sonuna (vi editöründe `Shift+G` ile en alta gidilip, `o` ile yeni satır açılıp) şu kural eklendi:

```
devopstester ALL=(ALL) NOPASSWD: /usr/bin/systemctl start nginx, /usr/bin/systemctl stop nginx, /usr/bin/systemctl restart nginx, /usr/bin/systemctl status nginx
```

### 🐛 Hata & Çözüm

İlk yazımda iki yazım hatası oluştu: `/usr/bsn/systemctl` (`bin` yerine `bsn`) ve `restart ngingx` (`nginx` yerine `ngingx`). Bu hata, satırı doğrudan gözle kontrol etmek yerine şu doğrulama komutuyla tespit edildi:

```bash
sudo -l -U devopstester
```

Bu komut, sudoers dosyasını yorumlayarak kullanıcıya tam olarak hangi yetkilerin tanımlı olduğunu gösterir — en güvenilir doğrulama yöntemi. Hatalı satır `visudo` içinde imleç o satıra getirilip `dd` (satır sil) ile silindi, doğru satır yeniden yazılıp `:wq` ile kaydedildi.

**✅ visudo'nun güvenlik avantajı:** `nano`/`vim` ile dosyayı doğrudan düzenlemek yerine `visudo` kullanmanın sebebi, sözdizimi hatası varsa dosyayı **kaydetmene izin vermemesi** — böylece sudo sistemi tamamen bozulmuyor.

---

## 5. Doğrulama Testleri

### Test 1 — Kullanıcı değiştirme yöntemleri

```bash
su - devopstester        # ❌ Yetkilendirme hatası (root şifresi soruyor, bilinmiyordu)
sudo su - devopstester   # ✅ Başarılı (kendi sudo şifremle geçiş yaptım)
```

**Fark:** `su`, hedef kullanıcının (veya root'un) şifresini ister. `sudo su`, komutu çalıştıran kişinin (benim) sudo yetkisini kullanarak geçiş yapar — kendi şifremle giriş yapabildim.

### Test 2 — İzin verilen komut (şifre sormamalı)

```bash
sudo systemctl restart nginx
```
**Sonuç:** ✅ Hiçbir şifre sormadan, hatasız çalıştı — `NOPASSWD` kuralı doğru işliyor.

### Test 3 — İzin verilmeyen komut (reddedilmeli)

```bash
sudo dnf update -y
```
**Sonuç:**
```
Sorry, user devopstester is not allowed to execute '/bin/dnf update -y' as root on localhost.localdomain.
```

**🔍 Önemli gözlem:** Bu komut için sudo yine de şifre sordu, *sonra* reddetti. Sebep: `sudo`, önce **kimlik doğrulaması** yapar (şifre doğru mu?), ardından **yetki kontrolü** yapar (bu komutu çalıştırmaya izni var mı?). `NOPASSWD` sadece belirtilen komutlar için şifre adımını atlatır; listede olmayan komutlarda şifre yine sorulur, ama yetki olmadığı için sonunda reddedilir.

---

## 📊 Komut Referansı

| Komut | Amacı | Örnek |
|-------|-------|-------|
| **`useradd -m`** | Ev dizini ile yeni kullanıcı oluşturur | `sudo useradd -m devopstester` |
| **`passwd`** | Kullanıcı şifresi belirler/değiştirir | `sudo passwd devopstester` |
| **`which`** | Bir komutun tam yolunu gösterir | `which systemctl` |
| **`visudo`** | sudoers dosyasını güvenli şekilde düzenler | `sudo visudo` |
| **`sudo -l -U <kullanıcı>`** | Bir kullanıcının sudo yetkilerini listeler | `sudo -l -U devopstester` |
| **`su - <kullanıcı>`** | Hedef kullanıcının şifresiyle geçiş yapar | `su - devopstester` |
| **`sudo su - <kullanıcı>`** | Kendi sudo yetkinle geçiş yapar | `sudo su - devopstester` |

---

## 🧠 Quiz (Gerçek Sonuçlar)

| # | Soru | Cevap | Sonuç |
|---|------|-------|-------|
| 1 | `su` ile `sudo su` arasındaki fark nedir? | `su` hedef kullanıcının şifresini ister; `sudo su` kendi sudo şifreni ister | ✅ Doğru |
| 2 | `devopstester`, `dnf update` denediğinde neden yine şifre soruldu? | `NOPASSWD` kapsamı dışındaki komutlar için sudo hâlâ kimlik doğrulaması ister, sonra yetki kontrolü yapıp reddeder | ✅ Doğru |

---

ℹ️ _Tüm adımlar yerel Rocky Linux VM'inde test edilmiştir._
