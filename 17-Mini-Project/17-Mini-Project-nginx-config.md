# Nginx Config — Faz 17 (Mini Proje) Sunucusu

## 📝 Özet

17. fazda (Mini Proje: Nginx, Docker, Git & SSH) kurulan sunucudaki Nginx'in gerçek, çalışan konfigürasyon dosyası. `apt install nginx` ile kurulum sırasında gelen varsayılan (`default`) config kullanıldı — bu fazda config dosyasının kendisinde özel bir değişiklik yapılmadı, sadece `root` dizininde (`/var/www/html`) servis edilen `index.html` dosyası, kendi journal reposundaki sayfa ile değiştirildi.

## 📂 Dosya Konumu

```
/etc/nginx/sites-available/default
```

Bu dosya, `/etc/nginx/sites-enabled/default` üzerinden bir symlink ile etkinleştirilmiş durumda (Debian/Ubuntu'nun standart Nginx site yönetim yapısı).

## 📄 Config İçeriği (Yorum Satırları Çıkarılmış, Aktif Kısım)

```nginx
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        root /var/www/html;

        # Add index.php to the list if you are using PHP
        index index.html index.htm index.nginx-debian.html;

        server_name _;

        location / {
                # First attempt to serve request as file, then
                # as directory, then fall back to displaying a 404.
                try_files $uri $uri/ =404;
        }
}
```

## 🔍 Satır Satır Açıklama

| Direktif | Açıklama |
|---|---|
| `listen 80 default_server;` | Sunucu, IPv4 üzerinden 80 numaralı portu (standart HTTP portu) dinliyor. `default_server`, bu bloğun, eşleşen başka bir `server_name` bulunamadığında kullanılacak varsayılan blok olduğunu belirtiyor. |
| `listen [::]:80 default_server;` | Aynı şeyin IPv6 karşılığı. |
| `root /var/www/html;` | Bu sunucudan servis edilecek dosyaların kök dizini. `17-Mini-Project/index.html`, `cp` komutuyla tam olarak buraya (`/var/www/html/index.html` olarak) kopyalandı. |
| `index index.html index.htm index.nginx-debian.html;` | Bir dizine istek geldiğinde sırasıyla denenecek varsayılan dosya isimleri — ilk bulunan servis edilir. |
| `server_name _;` | Alt karakteri (`_`), "hangi domain adıyla gelirse gelsin bu bloğu kullan" anlamına gelir — bu sunucuda henüz bir domain adı tanımlanmadığı için (sadece IP üzerinden erişildi) bu haliyle bırakıldı. |
| `location / { try_files $uri $uri/ =404; }` | Gelen her istek için: önce isteği birebir bir dosya olarak dene, olmazsa bir dizin olarak dene, o da olmazsa 404 (Not Found) döndür. |

## 🧪 Doğrulama Komutları

Config'in geçerliliği ve çalışırlığı şu komutlarla kontrol edilebilir:

```bash
sudo nginx -t                    # config sözdizimi hatası var mı kontrol eder
sudo systemctl status nginx      # servis çalışıyor mu kontrol eder
curl localhost                   # yerelden HTTP isteği ile test eder
```

## 🐛 Bu Fazda Config İle İlgili Karşılaşılan Bir Durum

Config dosyasının kendisinde bir hata yaşanmadı, ama bir kavramsal tuzak fark edildi (kaynak müfredatta da vurgulanan bir nokta): `root /var/www/html;` direktifi, klonlanan Git reposundaki dosyayı **otomatik olarak** işaret etmiyor. `git clone` ile indirilen `~/devops-learning-journal/17-Mini-Project/index.html` ve Nginx'in gerçekte servis ettiği `/var/www/html/index.html`, iki ayrı kopya. Repo güncellenip `git pull` yapılsa bile, Nginx'in servis ettiği dosya elle tekrar `cp` ile kopyalanmadan güncellenmez. Bu fazda bu adım manuel yapıldı; gerçek bir production ortamında bu genelde bir symlink (`root`'u doğrudan repo klasörüne işaret ettirmek) veya bir CI/CD deploy adımıyla otomatize edilir.

## 📊 İlgili Komutlar (Bu Config'in Kurulma Süreci)

| Komut | Açıklama |
|---|---|
| `sudo apt install nginx -y` | Nginx'i kurar, bu varsayılan config dosyasını da beraberinde getirir |
| `sudo systemctl enable --now nginx` | Nginx'i hem hemen başlatır hem açılışta otomatik etkinleştirir |
| `sudo cp ~/devops-learning-journal/17-Mini-Project/index.html /var/www/html/index.html` | `root` dizinindeki dosyayı, kendi repodaki sayfa ile değiştirir (config'in kendisini değiştirmez, sadece servis edilen içeriği değiştirir) |
