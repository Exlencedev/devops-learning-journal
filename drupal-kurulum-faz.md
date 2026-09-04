# Faz — Drupal Kurulumu: Rocky Linux 9, nginx, PHP-FPM ve PostgreSQL

## 📝 Özet

Bu fazda, Vagrant ile ayağa kaldırılmış bir **Rocky Linux 9** sanal makinesi üzerinde **nginx + PHP-FPM** stack'i kullanılarak sıfırdan bir **Drupal** kurulumu gerçekleştirildi. Kurulum sürecinde dosya sistemi izinleri, veritabanı seçimi (PostgreSQL), kimlik doğrulama yöntemleri ve sürüm uyumsuzluğu gibi birbirini takip eden gerçek hatalarla karşılaşıldı; her biri adım adım teşhis edilip çözüldü. Süreç, Drupal web arayüzü üzerinden site kurulumunun tamamlanmasıyla sonuçlandı.

**Ortam:**
- İşletim sistemi: Rocky Linux 9 (Vagrant üzerinde)
- Web sunucu: nginx
- PHP: 8.3.33 (PHP-FPM, `apache` kullanıcısı ile çalışıyor)
- Veritabanı: PostgreSQL (ilk denemede 13.23, sonrasında 16'ya yükseltildi)
- Proje dizini (VM içinde): `/var/www/drupal/web`
- Host tarafı proje klasörü: `C:\Users\ege\drupal-vagrant`

---

## 1. Dosya Sistemi Hataları ve Çözümü

Drupal kurulum sihirbazı ilk açıldığında iki hata verdi:

1. `sites/default/files` dizini yok ve otomatik oluşturulamadı (izin sorunu).
2. `sites/default/settings.php` dosyası yok.

### Teşhis: PHP-FPM hangi kullanıcıyla çalışıyor?

```bash
ps aux | grep nginx
ps aux | grep php-fpm
```

Çıktıya göre:
- nginx worker process'leri `nginx` kullanıcısıyla,
- **PHP-FPM pool'ları `apache` kullanıcısıyla** çalışıyordu.

Bu, ilk denemede kullanılan `www-data` kullanıcısının bu sistemde **var olmadığını** gösterdi — `www-data`, Debian/Ubuntu ailesine özgüdür; Rocky Linux (RHEL tabanlı) sistemlerde karşılığı genellikle `apache` veya `nginx`'tir.

### Drupal proje dizininin bulunması

Doğru dizini bulmak için nginx config dosyaları kontrol edildi:

```bash
sudo grep -r "root" /etc/nginx/conf.d/ /etc/nginx/nginx.conf
```

Sonuç: `root /var/www/drupal/web;` → gerçek Drupal kök dizini bu şekilde tespit edildi (host'taki `C:\Users\ege\drupal-vagrant` klasörü, VM içinde farklı bir yola karşılık geliyordu).

### Çözüm

```bash
cd /var/www/drupal/web
sudo mkdir -p sites/default/files
sudo chown -R apache:apache sites/default/files
sudo chmod -R 755 sites/default/files

sudo cp sites/default/default.settings.php sites/default/settings.php
sudo chown apache:apache sites/default/settings.php
sudo chmod 666 sites/default/settings.php
```

`getenforce` ile SELinux durumu da kontrol edildi (Rocky Linux'ta varsayılan olarak aktif olabilir); gerekirse `httpd_sys_rw_content_t` context'i `semanage`/`restorecon` ile ayarlanacaktı, ancak bu ortamda dosya izinleri düzeltmesi yeterli oldu.

---

## 2. Veritabanı Kurulumu — PostgreSQL

Drupal kurulum formunda veritabanı türü olarak **PostgreSQL** seçildi. Ancak bu veritabanı ve kullanıcı önceden oluşturulmamıştı.

### Kullanıcı ve veritabanı oluşturma

```bash
sudo -i -u postgres psql
```

```sql
CREATE USER drupaluser WITH PASSWORD 'root';
CREATE DATABASE drupaldb OWNER drupaluser;
GRANT ALL PRIVILEGES ON DATABASE drupaldb TO drupaluser;
```

🐛 **Hata:** `CREATE USER` ve `CREATE DATABASE` komutları `role "drupaluser" already exists` ve `database "drupaldb" already exists` hatası verdi.

**Neden:** Kullanıcı ve veritabanı muhtemelen Vagrant provisioning script'i tarafından daha önce oluşturulmuştu, ancak şifre bilinmiyordu.

**Çözüm:** Şifre garantiye alınarak güncellendi:

```sql
ALTER USER drupaluser WITH PASSWORD 'root';
```

---

## 3. Kimlik Doğrulama Hataları

Drupal kurulum formuna `localhost:5432 / drupaldb / drupaluser / root` bilgileri girildiğinde sırasıyla iki farklı hata alındı:

### Hata: `Permission denied` (SQLSTATE[08006])

İlk bağlantı denemesinde "could not connect to server: Permission denied" hatası alındı. Bunun tipik sebebi SELinux'un PHP-FPM/httpd sürecinin ağ üzerinden veritabanına bağlanmasına izin vermemesidir:

```bash
getenforce
sudo setsebool -P httpd_can_network_connect_db 1
```

### Hata: `Ident authentication failed for user "drupaluser"`

Bağlantı ağ seviyesinde kurulmaya başladı, ancak PostgreSQL'in `pg_hba.conf` dosyasında `host` satırları `ident` doğrulama yöntemini kullanıyordu — bu yöntem şifreyi değil, işletim sistemi kullanıcı adını kontrol eder.

**Teşhis:**

```bash
sudo find / -name "pg_hba.conf" 2>/dev/null
```

**Planlanan çözüm:** `host ... ident` satırlarının `md5` (şifre tabanlı doğrulama) olarak değiştirilmesi.

> **Not:** Dosya incelendiğinde aslında `ident` değil, `scram-sha-256` kullanıldığı görüldü. `scram-sha-256`, `md5`'in daha güvenli bir şifre-tabanlı alternatifidir ve zaten kullanıcı adı/şifre ile bağlanmaya izin verir — bu yüzden dosyada herhangi bir değişiklik **yapılmasına gerek kalmadı**.

---

## 4. Sürüm Uyumsuzluğu — PostgreSQL 13 → 16

Kimlik doğrulama sorunları çözüldükten sonra Drupal şu hatayı verdi:

> The database server version 13.23 is less than the minimum required version 16.

**Neden:** Rocky Linux'un standart AppStream reposundan kurulan PostgreSQL sürümü (13.23), Drupal'ın (bu sürümünün) gerektirdiği minimum sürümün (16) altındaydı.

### Kurulu paketlerin tespiti

```bash
rpm -qa | grep postgresql
```

```
postgresql-private-libs-13.23-5.el9_8.x86_64
postgresql-13.23-5.el9_8.x86_64
postgresql-server-13.23-5.el9_8.x86_64
postgresql-contrib-13.23-5.el9_8.x86_64
```

### Çözüm: PostgreSQL 16'ya geçiş

```bash
# Eski sürümü durdur
sudo systemctl stop postgresql
sudo systemctl disable postgresql

# Eski paketleri kaldır
sudo dnf remove -y postgresql-server postgresql postgresql-contrib postgresql-private-libs

# PGDG (resmi PostgreSQL) reposunu ekle
sudo dnf install -y https://download.postgresql.org/pub/repos/yum/reporpms/EL-9-x86_64/pgdg-redhat-repo-latest.noarch.rpm

# Rocky Linux'un yerleşik postgresql modülünü devre dışı bırak
sudo dnf -qy module disable postgresql

# PostgreSQL 16'yı kur
sudo dnf install -y postgresql16-server postgresql16-contrib

# Veritabanı kümesini başlat (initialize)
sudo /usr/pgsql-16/bin/postgresql-16-setup initdb

# Servisi etkinleştir ve başlat
sudo systemctl enable --now postgresql-16
```

Yeni kurulum ayrı bir veritabanı kümesi olduğu için `drupaluser` ve `drupaldb` bu yeni PostgreSQL 16 örneğinde tekrar oluşturuldu (bkz. Bölüm 2).

---

## 5. Site Yapılandırması ve Kurulumun Tamamlanması

Veritabanı bağlantısı başarılı olduktan sonra Drupal'ın "Configure site" adımına geçildi:

- **Site information:** Site adı ve site e-posta adresi girildi.
- **Site maintenance account:** `admin` kullanıcı adı ve güçlü bir şifre ile yönetici hesabı oluşturuldu.
- **Regional settings:** Ülke ve saat dilimi seçildi.

Form gönderildikten sonra Drupal kurulumu başarıyla tamamlandı ve site ana sayfası açıldı. ✅

### Kurulum sonrası güvenlik notu

Kurulum sırasında `settings.php` dosyasına geçici olarak `666` izni verilmişti (kurulum sihirbazının veritabanı bilgilerini yazabilmesi için). Kurulum tamamlandıktan sonra bu izin güvenlik amacıyla sıkılaştırıldı:

```bash
sudo chmod 644 /var/www/drupal/web/sites/default/settings.php
```

---

## 🐛 Hata & Çözüm Özeti

| # | Hata | Neden | Çözüm |
|---|------|-------|-------|
| 1 | `sites/default/files` oluşturulamadı | PHP-FPM'in çalıştığı kullanıcının (`apache`) dizine yazma izni yoktu; `www-data` kullanıcısı bu sistemde mevcut değil | Dizin manuel oluşturulup `chown apache:apache` ve `chmod 755` uygulandı |
| 2 | `settings.php` yok | Kurulum sihirbazı dosyayı otomatik oluşturamadı | `default.settings.php` kopyalanıp geçici olarak `666` izni verildi |
| 3 | `role/database already exists` | Kullanıcı/veritabanı Vagrant provisioning ile önceden oluşturulmuş, şifresi bilinmiyordu | `ALTER USER ... WITH PASSWORD` ile şifre yeniden belirlendi |
| 4 | `SQLSTATE[08006] Permission denied` | SELinux, PHP-FPM'in veritabanına ağ üzerinden bağlanmasını engelliyordu | `setsebool -P httpd_can_network_connect_db 1` |
| 5 | `Ident authentication failed` | `pg_hba.conf` başta `ident`/kullanıcı bazlı doğrulama bekleniyordu | İncelemede aslında `scram-sha-256` (şifre tabanlı) kullanıldığı görüldü, ek değişiklik gerekmedi |
| 6 | `database server version 13.23 < 16` | Sistemde Rocky'nin varsayılan AppStream reposundan eski PostgreSQL sürümü kuruluydu | PGDG reposu eklenip PostgreSQL 16'ya geçildi, kullanıcı/veritabanı yeniden oluşturuldu |

---

## 📊 Komut Referansı

| Komut | Açıklama |
|-------|----------|
| `ps aux \| grep php-fpm` | PHP-FPM'in hangi kullanıcıyla çalıştığını gösterir |
| `sudo grep -r "root" /etc/nginx/conf.d/` | nginx'in gerçek doküman kök dizinini bulmak için |
| `sudo chown -R apache:apache <dizin>` | Dizin sahipliğini PHP-FPM kullanıcısına verir |
| `sudo -i -u postgres psql` | PostgreSQL komut satırına `postgres` kullanıcısıyla bağlanır |
| `CREATE USER ... WITH PASSWORD ...` | PostgreSQL kullanıcısı oluşturur |
| `ALTER USER ... WITH PASSWORD ...` | Var olan kullanıcının şifresini değiştirir |
| `getenforce` | SELinux'un aktif olup olmadığını gösterir |
| `setsebool -P httpd_can_network_connect_db 1` | httpd/PHP-FPM'in veritabanına ağ üzerinden bağlanmasına SELinux izni verir |
| `sudo find / -name "pg_hba.conf"` | PostgreSQL'in host tabanlı erişim kontrol dosyasını bulur |
| `sudo dnf -qy module disable postgresql` | Rocky Linux'un yerleşik PostgreSQL modülünü devre dışı bırakır (PGDG ile çakışmayı önler) |
| `sudo /usr/pgsql-16/bin/postgresql-16-setup initdb` | Yeni PostgreSQL 16 veritabanı kümesini başlatır |

---

## 🧠 Öğrenilen Dersler

1. **Kullanıcı adları dağıtıma göre değişir:** Debian/Ubuntu'da `www-data`, RHEL/Rocky'de genellikle `apache` veya `nginx` kullanılır. Web sunucusuyla ilgili izin komutları çalıştırılmadan önce `ps aux` ile gerçek kullanıcı doğrulanmalı.
2. **SELinux, "connection refused" değil "permission denied" şeklinde kendini gösterebilir:** Ağ bağlantısı seviyesinde bir engel varsa ve servis çalışıyor görünüyorsa, SELinux booleans (`httpd_can_network_connect_db` gibi) ilk şüpheli olmalı.
3. **PostgreSQL kimlik doğrulama yöntemleri (`peer`, `ident`, `md5`, `scram-sha-256`) birbirinden farklıdır:** Hata mesajındaki yöntem adını görmeden dosyayı değiştirmeye kalkmak yerine önce gerçek içerik kontrol edilmeli — bu fazda `scram-sha-256` zaten doğru çalışıyordu ve gereksiz bir değişiklikten kaçınıldı.
4. **Dağıtımın varsayılan repo'sundaki paket sürümü, uygulamanın minimum gereksinimini karşılamayabilir:** Bu durumda uygulamanın resmi paket deposunu (PostgreSQL için PGDG) eklemek, dağıtımın kendi sürümüyle uğraşmaktan daha güvenilir bir çözümdür.
