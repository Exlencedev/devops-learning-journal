# ⏰ Cron & Otomasyon

Bu belge, `crontab` ile periyodik görev zamanlamayı, cron içinde gerçek bir
`sudo` çakışmasını debug etmeyi, ve `logrotate`'e bir bakışı kapsar.

---

## 1. Script'i Cron'a Bağlama

14. fazdaki disk kullanım scripti (`~/disk_check.sh`) temel alındı. Cron'da
göreli yol güvenilir olmadığı için önce tam yol doğrulandı:

```bash
realpath ~/disk_check.sh
chmod +x ~/disk_check.sh
```

`crontab -e` ile her gece 02:00'de çalışacak şekilde tanımlandı:

```
0 2 * * * /home/ege/disk_check.sh >> /home/ege/disk_check.log 2>&1
```

- `0 2 * * *` → her gün saat 02:00
- `>>` → stdout'u log dosyasına **ekleyerek** yaz
- `2>&1` → stderr'i de aynı dosyaya yönlendir

### 🐛 Hata & Çözüm: `vi` İçinde Satır Bölünmesi

İlk denemede `crontab -e` kaydedilirken şu hata alındı:

```
crontab: installing new crontab
"/tmp/crontab.gDq5F4":2: bad minute
Invalid crontab file, can't install.
```

Sebep: Uzun crontab satırı, terminal genişliği yüzünden `vi` içinde **görsel
olarak** sarılıyor gibi görünse de, aslında gerçek bir satır sonuyla ikiye
bölünmüştü (`vi` durum çubuğu `2L` — 2 satır — diyordu, 1 değil). Cron her
görevi tek satır olarak okuduğu için, ikinci parçayı (`/home/ege/disk_check.log
2>&1`) bağımsız, eksik bir görev satırı sandı ve dakika alanını bulamayınca
`bad minute` hatası verdi.

**Çözüm:** `vi` yerine, satır bütünlüğünü garanti eden bir shell pipe kalıbı
kullanıldı:

```bash
(crontab -l 2>/dev/null; echo "0 2 * * * /home/ege/disk_check.sh >> /home/ege/disk_check.log 2>&1") | crontab -
```

Doğrulama:

```bash
crontab -l
```

```
0 2 * * * /home/ege/disk_check.sh >> /home/ege/disk_check.log 2>&1
```

> 💡 **Ders:** Uzun crontab satırları eklerken `vi`'nin satırı görsel olarak
> sarması yanıltıcı olabilir. `(crontab -l; echo "...") | crontab -` kalıbı
> editör bağımlılığını tamamen ortadan kaldırır.

---

## 2. Gerçek Bir `sudo`-Cron Çakışması

Script'e kasıtlı olarak `sudo` gerektiren bir satır eklendi:

```bash
sudo systemctl status sshd > /dev/null
echo "sudo command finished"
```

**Elle çalıştırıldığında** (interaktif terminal, şifre sorulabiliyor):

```bash
~/disk_check.sh
```

Sorunsuz çalıştı — şifre girildi, `"sudo command finished"` yazdırıldı.

### 🐛 Hata & Çözüm: Terminal'siz Ortamda `sudo` Başarısız Oluyor

Cron'un çalışma bağlamını (hiçbir TTY olmadan tetiklenme) gerçekçi şekilde
simüle etmek için `at` kullanıldı — gece 02:00'yi beklemek yerine:

```bash
sudo -k                              # sudo'nun önbelleğini sıfırla
echo "/home/ege/disk_check.sh >> /home/ege/disk_check.log 2>&1" | at now + 1 minute
```

(`at` kurulu değildi, önce kuruldu: `sudo dnf install at -y` ve
`sudo systemctl enable --now atd`.)

1 dakika sonra log'da gerçek hata görüldü:

```
sudo: şifreyi okumak için bir terminal gereklidir; ya standart girdiden
      okumak için -S seçeneğini kullanın ya da bir askpol...
sudo: a password is required
sudo command finished
```

`sudo`, terminal olmadığı için şifre isteyemedi ve başarısız oldu — ama
script bu noktada durmadı, `echo` satırı koşulsuz olarak yine çalıştı. Bu,
kaynak repodaki "cron'un varsayılan mail-gönderme davranışı olmadan bu tür
hataların fark edilmesi zor olur" uyarısıyla birebir örtüşüyor:
`2>&1` ile log'a yönlendirme olmasaydı bu hata hiç görülmeyebilirdi.

### ✅ Çözüm: Dar Kapsamlı `sudoers` Kuralı

Geniş bir yetki vermek yerine, sadece gerçekten gerekli olan tek komut için
şifresiz izin tanımlandı:

```bash
sudo visudo
```

```
ege ALL=(ALL) NOPASSWD: /usr/bin/systemctl status sshd
```

Doğrulama:

```bash
sudo -l -U ege
```

```
User ege may run the following commands on localhost:
    (ALL) NOPASSWD: /usr/bin/systemctl status sshd
```

**Yeniden test** — aynı `at` senaryosu tekrarlandı:

```bash
echo "/home/ege/disk_check.sh >> /home/ege/disk_check.log 2>&1" | at now + 1 minute
```

Sonuç, hatasız:

```
Disk usage: 12%
Disk usage is normal.
sudo command finished
```

---

## 3. `logrotate`'e Bir Bakış

Nginx paketiyle birlikte gelen hazır config incelendi:

```bash
cat /etc/logrotate.d/nginx
```

```
/var/log/nginx/*.log {
    create 0640 nginx root
    daily
    rotate 10
    missingok
    notifempty
    compress
    delaycompress
    sharedscripts
    postrotate
        /bin/kill -USR1 `cat /run/nginx.pid 2>/dev/null` 2>/dev/null || true
    endscript
}
```

| Direktif | Anlamı |
|---|---|
| `create 0640 nginx root` | Döndürülen dosyanın yerine, bu izin/sahiplikte yeni boş dosya oluştur |
| `daily` | Her gün döndür |
| `rotate 10` | En fazla 10 eski kopyayı sakla, sonra en eskisini sil |
| `missingok` | Log dosyası yoksa hata verme |
| `notifempty` | Log dosyası boşsa döndürme |
| `compress` | Eski logları gzip ile sıkıştır |
| `delaycompress` | Bir önceki (en son döndürülen) dosyayı bir sonraki döngüye kadar sıkıştırma |
| `sharedscripts` | Birden fazla dosya döndürülse bile `postrotate`'i sadece bir kez çalıştır |
| `postrotate` (`kill -USR1`) | Nginx'e log dosyalarını yeniden açması için sinyal gönder |

### 🐛 Neden `kill -USR1` Gerekli

Nginx, log dosyasını isimle değil, açık dosya tanıtıcısıyla (file
descriptor) tutar. `logrotate` eski `access.log`'u `access.log.1`'e taşıyıp
yeni bir `access.log` oluşturduğunda, Nginx bunun farkında olmaz ve hâlâ
eski (artık `.1` olan) dosyaya yazmaya devam eder — `USR1` sinyali alana
kadar.

### Canlı Doğrulama

`/var/log/nginx/` dizini `711` izinli (`rwx--x--x`) olduğu için normal
kullanıcı `ls` ile listeleyemedi — `sudo` gerekti:

```bash
sudo logrotate -f /etc/logrotate.d/nginx
sudo ls -la /var/log/nginx/
```

```
-rw-r-----. 1 nginx root      0 access.log      ← yeni, boş
-rw-r--r--. 1 root  root    168 access.log.1    ← eski içerik, henüz sıkıştırılmamış
-rw-r-----. 1 nginx root      0 error.log       ← yeni, boş
-rw-r--r--. 1 root  root  10395 error.log.1     ← eski içerik, henüz sıkıştırılmamış
```

`create 0640 nginx root` ve `delaycompress` davranışı birebir doğrulandı.
İkinci bir `-f` denemesinde dosyalar değişmedi — sebebi
`/var/lib/logrotate/logrotate.status` durum dosyasının aynı gün içinde
tekrar döndürmeyi engellemesiydi (`-f` sadece zamanlamayı zorlar, durum
takibini değil):

```bash
sudo cat /var/lib/logrotate/logrotate.status | grep nginx
```

```
"/var/log/nginx/error.log" 2026-8-7-22:6:51
"/var/log/nginx/access.log" 2026-8-7-22:6:51
```

---

## 📊 Komut Referansı

| Komut | Amacı |
|-------|-------|
| **`crontab -e` / `-l`** | Kullanıcı crontab'ını düzenler / listeler |
| **`(crontab -l; echo "...") \| crontab -`** | Editör riski olmadan crontab'a satır ekler |
| **`sudo -k`** | sudo'nun önbelleğe alınmış doğrulamasını sıfırlar |
| **`echo "cmd" \| at now + 1 minute`** | Cron'un terminal'siz bağlamını gerçekçi test etmek için tek seferlik görev planlar |
| **`visudo`** | `/etc/sudoers`'ı güvenli şekilde düzenler |
| **`sudo -l -U <user>`** | Bir kullanıcının sudo yetkilerini listeler |
| **`logrotate -f <config>`** | Zamanlamayı yok sayıp döndürmeyi zorla tetikler |
| **`kill -USR1 <pid>`** | Nginx'e log dosyalarını yeniden açması için sinyal gönderir |

---

## 🧠 Quiz (Çoktan Seçmeli)

**S1:** Cron içinde çalışan bir script'te `sudo` komutu neden "a password is required" hatasıyla başarısız olur?

- A) Cron, sudo komutlarını güvenlik gereği tamamen engeller
- B) Cron'un çalıştırdığı görevlerin bağlı olduğu bir terminal (TTY) yoktur, bu yüzden sudo şifre isteyemez ✅
- C) crontab dosyasında sudo komutlarına izin veren bir ayar eksiktir
- D) sudo, sadece root kullanıcısının crontab'ında çalışır

**S2:** Bu sorunu çözmek için en doğru yaklaşım hangisidir?

- A) Kullanıcıyı `wheel` grubuna ekleyip tüm komutlara sınırsız NOPASSWD yetkisi vermek
- B) Script'i her zaman root olarak, `su -` ile çalıştırmak
- C) `visudo` ile sadece gerekli olan spesifik komut için dar kapsamlı bir NOPASSWD kuralı tanımlamak ✅
- D) Script içindeki `sudo` komutunu tamamen kaldırıp elle çalıştırmak

---

ℹ️ _Tüm komutlar yerel Rocky Linux VM'inde (Rocky9-Test) test edilmiştir. Bu
fazda 2 gerçek hata debug edilmiştir: `vi`'da crontab satırının ikiye
bölünmesinden kaynaklanan `bad minute` hatası, ve cron/at bağlamında
terminal'siz `sudo` başarısızlığı — ikisi de kaynak repodaki senaryolarla
örtüşen, gerçek debug süreçleriyle çözülmüştür._
