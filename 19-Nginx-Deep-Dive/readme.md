# Faz 19 — Nginx Derinleşme: Reverse Proxy, Path Yönetimi, ve Forward Proxy

## 📝 Özet

Bu fazda Nginx'in reverse proxy yeteneklerine derinlemesine girildi: temel reverse proxy kurulumu, path bazlı yönlendirme (birden fazla backend), path rewrite davranışı, path engelleme (`deny`/`allow`), ve son olarak Nginx'ten tamamen farklı bir araç olan Squid ile forward proxy kurulumu. Tüm testler WSL2/Ubuntu 26.04 sunucusunda (17-18. fazlarda kurulan aynı ortam), backend servisler olarak Python'un yerleşik HTTP sunucusu kullanılarak yapıldı.

## 1. Reverse Proxy Nedir ve Neden Kullanılır

Nginx, dışarıdan gelen istekleri alıp arka plandaki servislere iletir. Kullanıcı, arka planda kaç servis olduğunu veya hangi portlarda çalıştığını bilmez.

**Neden kullanılır:**
- Backend servislerin port/IP bilgisi dışarıya çıkmaz (gizlilik)
- Tek bir giriş noktası (port 80/443) üzerinden birden fazla servis yönetilir
- Load balancing, SSL sonlandırma, rate limiting Nginx'te yapılır, backend'e yük binmez

## 2. Temel Reverse Proxy Kurulumu

### Backend Servisi
```bash
mkdir -p /tmp/backend
echo "<h1>Backend Servisi Çalışıyor - Port 8080</h1>" > /tmp/backend/index.html
cd /tmp/backend && python3 -m http.server 8080 &
```

### Nginx Config (`/etc/nginx/sites-available/default`)
```nginx
server {
        listen 80 default_server;
        listen [::]:80 default_server;

        root /var/www/html;
        index index.html index.htm index.nginx-debian.html;
        server_name _;

        location / {
                proxy_pass http://localhost:8080;
                proxy_set_header Host $host;
                proxy_set_header X-Real-IP $remote_addr;
        }
}
```

**Direktiflerin anlamı:**
- `proxy_pass http://localhost:8080` — gelen isteği 8080 portuna ilet.
- `proxy_set_header Host $host` — isteğin hangi domain'e geldiğini backend'e de söyle.
- `proxy_set_header X-Real-IP $remote_addr` — kullanıcının gerçek IP'sini backend'e de söyle (bu olmasa backend her isteğin 127.0.0.1'den — yani Nginx'ten — geldiğini sanır).

### Doğrulama
```bash
curl localhost
# → <h1>Backend Servisi Çalışıyor - Port 8080</h1>
```
Backend log'unda isteğin `127.0.0.1`'den (Nginx'ten) geldiği görüldü — kullanıcının gerçek IP'si değil, çünkü test yerelden yapıldı.

## 3. Path Bazlı Yönlendirme

Aynı Nginx üzerinden farklı path'lere gelen istekler farklı backend'lere yönlendirildi.

### Ek Backend'ler
```bash
mkdir -p /tmp/users && echo "<h1>Users Servisi</h1>" > /tmp/users/index.html
cd /tmp/users && python3 -m http.server 3000 &

mkdir -p /tmp/computers && echo "<h1>Computers Servisi</h1>" > /tmp/computers/index.html
cd /tmp/computers && python3 -m http.server 4000 &
```

### Config Eklentisi
```nginx
location /users/ {
        proxy_pass http://localhost:3000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
}

location /computers/ {
        proxy_pass http://localhost:4000/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
}
```

### Test Sonuçları
```bash
curl http://localhost/            → <h1>Backend Servisi Çalışıyor - Port 8080</h1>
curl http://localhost/users/      → <h1>Users Servisi</h1>
curl http://localhost/computers/  → <h1>Computers Servisi</h1>
```
Üçü de doğru backend'e yönlendi, doğrulandı.

## 4. Path Rewrite

`proxy_pass` direktifindeki sondaki `/` karakteri, path rewrite'ı (prefix soyma) belirler.

| `proxy_pass` | İstek | Backend'e Giden | Sonuç |
|---|---|---|---|
| `http://localhost:3000` (sonda `/` yok) | `/users/liste` | `/users/liste` (olduğu gibi) | 404 (backend bunu tanımıyor) |
| `http://localhost:3000/` (sonda `/` var) | `/users/liste` | `/liste` (prefix soyulmuş) | 200 |

### Doğrulama (Gerçek Test)
```bash
echo "<h1>Users Listesi</h1>" > /tmp/users/liste.html
curl http://localhost/users/liste.html
```
**Sonuç:** `<h1>Users Listesi</h1>` — log satırı `"GET /liste.html HTTP/1.0" 200` gösterdi, yani `/users/` prefix'i gerçekten soyulup backend'e sadece `/liste.html` gönderilmiş.

### 301 Redirect Davranışı
```bash
curl -v http://localhost/users 2>&1 | grep -E "HTTP|Location"
# < HTTP/1.1 301 Moved Permanently
# < Location: http://localhost/users/

curl -L http://localhost/users
# → <h1>Users Servisi</h1>
```
Sondaki `/` olmadan (`/users`) yazılınca, Nginx otomatik olarak `/users/`'a 301 ile yönlendirdi. `-L` ile bu redirect takip edildi ve doğru sayfaya ulaşıldı.

## 5. Path Engelleme

### Herkesi Engelleme
```nginx
location /admin {
        deny all;
}
```

**🐛 Karşılaşılan sorun:** Config eklenip `sudo systemctl reload nginx` çalıştırıldığında, `curl /admin` beklenen 403 yerine backend'den gelen bir 404 döndürdü — yani istek hâlâ `location /`'a gidip proxy'leniyordu, `deny all`'a hiç uğramıyordu. `sudo tail /var/log/nginx/error.log` de tamamen boştu (deny reddi loglanmamıştı bile).

**Kök neden ve çözüm:** `reload`, çalışan worker process'lere yeni config'i "yumuşak" şekilde iletir, ama bu ortamda worker'lar hemen güncellenmemiş görünüyordu. `sudo systemctl restart nginx` (tam yeniden başlatma) ile sorun kesin olarak çözüldü — bundan sonra `curl /admin` doğru şekilde `403 Forbidden` döndürdü.

### Sadece Localhost'a İzin Verme (Gerçek Dünya Kullanımı)
```nginx
location /admin {
        allow 127.0.0.1;
        allow ::1;
        deny all;
}
```
Nginx kuralları yukarıdan aşağıya okur, ilk eşleşmede durur: `127.0.0.1`'den mi geliyor → geç; `::1`'den mi geliyor → geç; başka biri → 403.

**🐛 Karşılaşılan sorun:** Sadece `allow 127.0.0.1;` (IPv6 satırı olmadan) ile test edildiğinde, `curl http://localhost/admin` **403 Forbidden** verdi — ama `curl http://127.0.0.1/admin` (aynı makine, farklı yazım) **404** verdi (yani izin verildi, backend'e ulaştı, orada `/admin` dosyası olmadığı için 404).

**Kök neden:** `curl -v http://localhost/admin` çıktısında şu satır bulundu:
```
* Established connection to localhost (::1 port 80) from ::1 port 39000
```
`localhost` yazınca istek bu sistemde **IPv6 (`::1`)** üzerinden gidiyor. Config'te sadece `allow 127.0.0.1` (IPv4) olduğu için bu istek izin listesinde bulunamadı, `deny all`'a düştü. `allow ::1` satırı eklenince, `localhost` isteği de doğru şekilde izin listesine girdi ve backend'e ulaşıp 404 döndürdü (doğru davranış — dosya gerçekten yok, ama artık *erişim* engellenmiyor).

**Sonuç Tablosu:**
| İstek | `allow 127.0.0.1` (tek) | `allow 127.0.0.1` + `allow ::1` |
|---|---|---|
| `curl http://localhost/admin` (→ `::1`) | 403 Forbidden | 404 (izinli, dosya yok) |
| `curl http://127.0.0.1/admin` | 404 (izinli, dosya yok) | 404 (izinli, dosya yok) |

## 6. Forward Proxy (Squid)

Nginx reverse proxy için tasarlanmıştır; forward proxy için ayrı bir araç, **Squid**, kullanıldı.

### Fark Tablosu
| | Reverse Proxy (Nginx) | Forward Proxy (Squid) |
|---|---|---|
| Kim gizleniyor | Backend sunucu | İstemci (kullanıcı) |
| Nerede duruyor | Sunucu tarafında | İstemci tarafında |
| Kullanım | Web siteleri, API'ler | Kurumsal internet kontrolü |

### Kurulum
```bash
sudo apt install squid -y
```
Squid, kurulumdan sonra otomatik olarak 3128 portunda dinlemeye başladı (`sudo ss -tlnp | grep 3128` ile doğrulandı).

### 🐛 Karşılaşılan Sorun: `http_access allow all` Etkisiz Kaldı
İlk denemede `echo "http_access allow all" | sudo tee -a /etc/squid/squid.conf` ile kural dosyanın **sonuna** eklendi. Windows'ta sistem proxy ayarı (`172.22.227.220:3128`) yapılıp `ifconfig.me`'ye gidildiğinde **"Access Denied" (403)** hatası alındı.

**Kök neden:** Squid'in varsayılan config'inde, bizim eklediğimiz satırdan **önce** gelen ve genel bir `http_access deny all` (dosyanın sonunda, tüm spesifik kurallardan sonra) içeren bir kural zinciri var. Squid ilk eşleşen kuralda durduğu için, dosyanın en sonuna eklenen bizim kuralımız hiçbir zaman değerlendirilmedi (zaten önceki kurallardan biri her isteği karşılıyordu).

**Çözüm adımları:**
1. `Safe_ports`/`SSL_ports` güvenlik kısıtlamalarını kaldırdık (sadece bu test/öğrenme ortamı için kabul edilebilir — production'da **önerilmez**).
2. Kuralı dosyanın en başına değil, mevcut `http_access allow localhost` satırından hemen sonrasına ekledik.
3. `sudo systemctl restart squid` ile tam yeniden başlattık.

Squid access log'unda (`/var/log/squid/access.log`) önce tüm isteklerin `TCP_DENIED/403` ile reddedildiği, düzeltmeden sonra ise `TCP_TUNNEL/200` ile başarıyla tünellendiği (örn. `CONNECT www.google.com:443`, `CONNECT claude.ai:443`) doğrudan gözlemlendi.

### 🔍 WSL2'ye Özgü Bir Gözlem: IP Gizleme Beklendiği Gibi Çalışmadı
Kaynak metinde, forward proxy testinde `ifconfig.me`'ye gidildiğinde **proxy sunucusunun IP'sinin** görünmesi bekleniyor (kullanıcının gerçek IP'si değil). Bu testte ise:
- Squid access log'u, Windows'un trafiğinin gerçekten Squid üzerinden geçtiğini kanıtladı (`TCP_TUNNEL/200 CONNECT claude.ai:443` gibi kayıtlar).
- Ama `ifconfig.me` sayfasında görünen IP, WSL sunucusunun IP'si değil, **kullanıcının gerçek Windows/ISP IP'si** (`85.107.65.67`) oldu.
- `X-Forwarded-For` header'ında ise Squid'in kendi IP'si (`172.22.224.1`) görülebiliyordu — yani Squid IP'yi "unutmuyor", sadece dışarıdan bakan servis onu göremiyor.

**Olası açıklama:** HTTPS trafiği (`CONNECT` metodu) ile Squid, veriyi şifresini çözmeden bir tünel gibi ilettiği için, TLS bağlantısının orijinal kaynağı görünür kalabiliyor. Ayrıca WSL2'nin kendisi zaten Windows'un arkasında NAT'lı olduğu için ("çift NAT" — Windows → ISP), Squid'in eklediği ek katman, dışarıdan bakan bir servis için görünmez kalmış olabilir. Bu, kaynak metindeki (gerçek, tek-katmanlı bir kiralık sunucu) senaryosundan **WSL2'nin çok katmanlı ağ mimarisi yüzünden farklılaşan** bir sonuç olarak not edildi — Squid'in çalıştığı (log'larla) kanıtlanmış olsa da, IP gizleme davranışı bu ortamda birebir gözlemlenemedi.

### Test Sonrası Temizlik
Windows'taki sistem proxy ayarı test bitince kapatıldı (`Ayarlar > Ağ ve İnternet > Proxy` → "Proxy sunucusu kullan" kapatıldı) — açık bırakılması, WSL kapandığında tüm internet bağlantısını kesebilirdi.

---

## 🐛 Hata & Çözüm (Özet)

### Hata 1: `location / { }` bloğunu değiştirirken satırlar yorum (`#`) olarak kaldı
**Neden:** `nano` ile düzenleme sırasında `Ctrl+O` ile kaydetme adımı gerçekleşmemiş, ekran görüntüsü yanıltıcıydı.
**Çözüm:** `cat` ile dosyanın gerçek halini kontrol edip, düzenlemenin aslında hiç uygulanmadığını fark ettik.

### Hata 2: `sed` ile `location / { }` değiştirilirken, dosyada iki kez geçen aynı desen (biri aktif blokta, biri tamamen yorumlu örnek blokta) ikisini birden etkiledi
**Neden:** `sed`'in aralık eşleşmesi (`/pattern1/,/pattern2/`) ilk eşleşmeden ikinci eşleşmeye kadar her şeyi kapsıyor, dosyada desenin birden fazla geçtiği hesaba katılmamıştı.
**Çözüm:** Dosyayı `sed` ile kısmi değiştirmek yerine, `tee` ile tamamen yeniden yazmaya geçildi — bu, kalan fazlarda da tercih edilen yöntem oldu.

### Hata 3: `deny all` eklendikten sonra `reload` ile değişiklik uygulanmadı
**Detay:** Yukarıda (Bölüm 5) anlatıldı. Çözüm: `reload` yerine `restart`.

### Hata 4: `allow 127.0.0.1` yeterli değildi, `localhost` isteği hâlâ engellendi
**Detay:** Yukarıda (Bölüm 5) anlatıldı. Çözüm: `allow ::1` (IPv6) de eklendi.

### Hata 5: Squid `http_access allow all` kuralı dosyanın sonuna eklenince etkisiz kaldı
**Detay:** Yukarıda (Bölüm 6) anlatıldı. Çözüm: Kuralı doğru sıraya (mevcut `allow localhost` kuralından hemen sonra) taşımak, ayrıca port güvenlik kısıtlamalarını (yalnızca bu test ortamı için) kaldırmak.

### Hata 6: `ipconfig.me` yazıldı, doğrusu `ifconfig.me`
**Neden:** Basit bir yazım hatası — Windows'un kendi `ipconfig` komutuyla karıştırılmış olabilir.
**Çözüm:** Squid access log'unda `CONNECT ipconfig.me:443` isteğinin `503` (sunucu yanıt vermedi) döndüğü görülüp doğru adresle tekrar denendi.

---

## 📊 Komut Referansı

| Komut | Açıklama |
|---|---|
| `proxy_pass http://host:port;` | İsteği belirtilen backend'e ilet (prefix soyulmaz) |
| `proxy_pass http://host:port/;` | İsteği ilet, path prefix'ini soy (rewrite) |
| `proxy_set_header Host $host;` | Orijinal domain bilgisini backend'e ilet |
| `proxy_set_header X-Real-IP $remote_addr;` | Kullanıcının gerçek IP'sini backend'e ilet |
| `deny all;` | Tüm istekleri engelle (403) |
| `allow 127.0.0.1; allow ::1;` | Localhost'a izin ver (IPv4 ve IPv6 ikisi de) |
| `sudo nginx -t` | Config sözdizimini test eder (uygulamadan) |
| `sudo systemctl reload nginx` | Config'i "yumuşak" şekilde uygular (bazen yetersiz kalabilir) |
| `sudo systemctl restart nginx` | Nginx'i tam olarak yeniden başlatır (kesin çözüm) |
| `curl -v <url> 2>&1 \| grep "Connected"` | curl'ün gerçekte hangi IP'ye bağlandığını gösterir |
| `sudo apt install squid -y` | Forward proxy için Squid kurulumu |
| `sudo tail -f /var/log/squid/access.log` | Squid'in gerçek zamanlı erişim log'unu izler |

---

## 🧠 Quiz

| # | Soru | Doğru Cevap |
|---|---|---|
| 1 | `location /admin { deny all; }` eklendikten sonra `reload` ile değişiklik uygulanmadı, `restart` ile çözüldü. Neden? | `reload`, worker process'lere yeni config'i her zaman hemen yansıtmayabiliyor; `restart` ile kesin çözüldü |
| 2 | `allow 127.0.0.1` varken `curl http://localhost/admin` 403 verirken `curl http://127.0.0.1/admin` çalıştı. Neden? | `localhost` bu sistemde IPv6 (`::1`) üzerinden çözümleniyordu, config'te sadece IPv4 izni vardı — `allow ::1` eklenince düzeldi |
