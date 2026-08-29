# Faz 21 — OpenResty: Token Authentication, PostgreSQL, MySQL, Redis

## 📝 Özet

Bu fazda OpenResty ile token korumalı bir API kuruldu. PostgreSQL, MySQL ve Redis dahil 4 servis Docker Compose ile birlikte ayağa kaldırıldı. Tüm testler WSL2/Ubuntu sunucusunda (17-20. fazlarda kullanılan aynı ortam) yapıldı ve hiçbir gerçek hatayla karşılaşılmadan (basit bir `sudo`/docker-grubu izin meselesi dışında) ilk denemede başarıyla tamamlandı.

## 1. Neden OpenResty

Nginx sadece yönlendirme yapabilir — "bu path'e gel, şu backend'e git". Kendi başına iş mantığı çalıştıramaz.

OpenResty, Nginx'in üzerine kurulu ama içine bir **Lua interpreter** gömülü bir dağıtım. Yani hem yönlendiriyor hem de içinde kod çalıştırabiliyor — token kontrol edebiliyor, veritabanına bağlanabiliyor, cevabı kendisi oluşturabiliyor.

## 2. Mimari

```
İstek geldi
  → Token doğru mu? (auth.lua)
    → Hayır → 401 Unauthorized
    → Evet → Hangi path?
              /users    → PostgreSQL'den kullanıcılar
              /products → MySQL'den ürünler
              /cache    → Redis'ten cache
```

## 3. Klasör Yapısı

```
openresty-demo/
├── docker-compose.yml   → 4 servisi tanımlar
├── Dockerfile           → OpenResty'ye pgmoon ekler
├── nginx.conf           → OpenResty'ye nasıl davranacağını söyler
├── lua/
│   ├── auth.lua         → token kontrol
│   ├── users.lua        → PostgreSQL'den veri çek
│   ├── products.lua     → MySQL'den veri çek
│   └── cache.lua        → Redis'ten veri çek
└── init/
    ├── postgres/init.sql → users tablosunu oluştur
    └── mysql/init.sql    → products tablosunu oluştur
```

## 4. Yapılandırma Dosyaları

### `docker-compose.yml`
4 servisi (openresty, postgres, mysql, redis) tek dosyada tanımlar. `docker compose up -d` ile hepsi birden ayağa kalkar.

```yaml
services:
  openresty:
    build: .
    ports:
      - "8080:80"
    volumes:
      - ./nginx.conf:/usr/local/openresty/nginx/conf/nginx.conf
      - ./lua:/usr/local/openresty/nginx/lua
    depends_on:
      - postgres
      - mysql
      - redis

  postgres:
    image: postgres:15
    environment:
      POSTGRES_DB: demo
      POSTGRES_USER: admin
      POSTGRES_PASSWORD: secret
    volumes:
      - ./init/postgres:/docker-entrypoint-initdb.d

  mysql:
    image: mysql:8
    environment:
      MYSQL_DATABASE: demo
      MYSQL_USER: admin
      MYSQL_PASSWORD: secret
      MYSQL_ROOT_PASSWORD: rootsecret
    volumes:
      - ./init/mysql:/docker-entrypoint-initdb.d

  redis:
    image: redis:7
```

### `Dockerfile`
OpenResty'nin hazır Alpine image'ında `pgmoon` (Lua'nın PostgreSQL ile konuşmasını sağlayan kütüphane) yoktu. Paket yöneticileriyle kurulamadığı için doğrudan GitHub'dan klonlandı:

```dockerfile
FROM openresty/openresty:alpine

RUN apk add --no-cache git && \
    git clone https://github.com/leafo/pgmoon.git /usr/local/openresty/lualib/pgmoon_repo && \
    cp -r /usr/local/openresty/lualib/pgmoon_repo/pgmoon /usr/local/openresty/lualib/pgmoon
```

### `nginx.conf`
```nginx
events {}

http {
    server {
        listen 80;
        resolver 127.0.0.11 valid=30s;

        access_by_lua_file /usr/local/openresty/nginx/lua/auth.lua;

        location /users {
            content_by_lua_file /usr/local/openresty/nginx/lua/users.lua;
        }

        location /products {
            content_by_lua_file /usr/local/openresty/nginx/lua/products.lua;
        }

        location /cache {
            content_by_lua_file /usr/local/openresty/nginx/lua/cache.lua;
        }
    }
}
```

**Önemli direktifler:**
- `access_by_lua_file` — her isteğe önce bu dosyayı çalıştır (token kontrol)
- `content_by_lua_file` — token geçerliyse cevabı bu dosya oluştursun
- `resolver 127.0.0.11 valid=30s;` — **Docker'ın iç DNS sunucusu**. Nginx, `postgres` gibi container isimlerini kendiliğinden IP'ye çeviremez; bu satır olmadan `no resolver defined to resolve "postgres"` hatası alınır.

## 5. Lua Dosyaları

### `auth.lua` — Token Kontrol
`Authorization` header'ından token'ı alır, `secret-token-123` ile eşleşmiyorsa 401 döndürüp isteği durdurur.

### `users.lua` — PostgreSQL
`pgmoon` kütüphanesiyle PostgreSQL'e bağlanır, `SELECT * FROM users` çalıştırır, sonucu JSON olarak döndürür. Her adımda (bağlantı, sorgu) hata kontrolü var.

### `products.lua` — MySQL
Aynı mantık, MySQL için. İki fark: kütüphane `resty.mysql` (OpenResty'de yerleşik geliyor, `pgmoon` gibi ayrıca kurulmasına gerek yok), ve bağlantı şekli `mysql:new()` sonra `db:connect({...})`.

### `cache.lua` — Redis
Önce Redis'te `demo_key` var mı bakar. Yoksa (`ngx.null`) oluşturur, `red:expire()` ile 30 saniyelik TTL koyar. Varsa doğrudan cache'ten döndürür, veritabanına gitmez.

## 6. Init SQL Dosyaları

Container ilk başladığında `docker-entrypoint-initdb.d` klasöründeki SQL dosyalarını otomatik çalıştırır — bu olmasa tablolar oluşmaz, sorgular hata verirdi.

- `init/postgres/init.sql` — `users` tablosu, 3 örnek kayıt (Harun, Ali, Ayşe)
- `init/mysql/init.sql` — `products` tablosu, 3 örnek kayıt (Laptop, Mouse, Klavye)

## 7. Kurulum ve Doğrulama

### Servisleri Başlatma
```bash
cd ~/openresty-demo
sudo docker compose up -d
```
(`sudo` gerekti çünkü `ege` kullanıcısı `docker` grubunda değildi — 17. fazdaki bilinçli tercihle tutarlı.)

**Sonuç:** 3 image (mysql, redis, postgres) çekildi, OpenResty image'ı Dockerfile'dan build edildi (`pgmoon` clone dahil), 4 container da başarıyla başlatıldı.

```bash
sudo docker compose ps
```
Tüm servisler `Up` durumda, OpenResty `0.0.0.0:8080->80/tcp` ile host'a doğru bağlanmış.

### Test 1: Token Olmadan (401 Beklenen)
```bash
curl http://localhost:8080/users
```
**Sonuç:** `Unauthorized: Token geçersiz veya eksik` — token kontrolü doğrulandı.

### Test 2: PostgreSQL (Token ile)
```bash
curl -H "Authorization: secret-token-123" http://localhost:8080/users
```
**Sonuç:**
```json
[{"name":"Harun","email":"harun@mail.com","id":1},{"name":"Ali","email":"ali@mail.com","id":2},{"name":"Ayşe","email":"ayse@mail.com","id":3}]
```
Türkçe karakter (`Ayşe`) dahil sorunsuz geldi.

### Test 3: MySQL (Token ile)
```bash
curl -H "Authorization: secret-token-123" http://localhost:8080/products
```
**Sonuç:**
```json
[{"name":"Laptop","id":1,"price":15000},{"name":"Mouse","id":2,"price":250},{"name":"Klavye","id":3,"price":500}]
```

### Test 4: Redis Cache (İki Ardışık İstek)
```bash
curl -H "Authorization: secret-token-123" http://localhost:8080/cache   # 1. istek
curl -H "Authorization: secret-token-123" http://localhost:8080/cache   # 2. istek
```
**Sonuç:**
- 1. istek: `{"value":"Bu veri Redis cache'den geldi! (yeni oluşturuldu)","source":"redis"}`
- 2. istek: `{"value":"Bu veri Redis cache'den geldi!","source":"redis"}`

Cache mantığı doğrulandı: ilk istekte değer oluşturulup Redis'e yazıldı, ikinci istekte (TTL süresi dolmadan) doğrudan cache'ten okundu — veritabanına tekrar gidilmedi.

---

## 🐛 Hata & Çözüm

### Hata 1: `sudo docker compose up -d` öncesi "permission denied"
**Neden:** `ege` kullanıcısı `docker` grubunda değildi (17. fazda Docker komutları bilinçli olarak `sudo` ile çalıştırılmıştı, root farkındalığı için).
**Çözüm:** Komutu `sudo` ile çalıştırmaya devam edildi (kalıcı çözüm olarak `sudo usermod -aG docker ege` de bir seçenekti, ama bu fazda tercih edilmedi).

### Küçük Bir Terminal Artefaktı (Gerçek Hata Değil)
`cache.lua` dosyasının `cat` çıktısında dosyanın başında fazladan bir `,` karakteri görünüyordu. `xxd` ile dosyanın ham baytları kontrol edilip bunun sadece bir terminal/ekran görüntüsü render sorunu olduğu, gerçek dosya içeriğinin temiz (`local redis = ...` ile başladığı) doğrulandı — bu, "her zaman şüpheli görüneni doğrudan doğrulama araçlarıyla (xxd, hexdump) kontrol et" prensibinin küçük ama gerçek bir uygulamasıydı.

**Not:** Bu fazda beklenenin aksine (kaynak metinde `resty.mysql`'in bazı Alpine varyantlarında ayrıca kurulması gerekebileceği ima ediliyordu), hiçbir ek kütüphane sorunu yaşanmadı — `openresty/openresty:alpine` image'ı `resty.mysql` ve `resty.redis`'i zaten hazır içeriyordu, sadece `pgmoon` manuel eklenmesi gerekti.

---

## 📊 Komut Referansı

| Komut | Açıklama |
|---|---|
| `docker compose version` | Docker Compose'un kurulu olup olmadığını ve sürümünü gösterir |
| `sudo docker compose up -d` | `docker-compose.yml`'deki tüm servisleri arka planda başlatır (gerekirse build eder) |
| `sudo docker compose ps` | Servislerin çalışma durumunu listeler |
| `sudo docker compose down` | Tüm servisleri durdurur ve kaldırır |
| `access_by_lua_file <dosya>` | Bir isteğe, işlenmeden önce çalıştırılacak Lua dosyasını belirtir (auth için ideal) |
| `content_by_lua_file <dosya>` | Bir location'ın cevabını oluşturacak Lua dosyasını belirtir |
| `resolver <ip> valid=<süre>;` | Nginx'in DNS çözümlemesi için kullanacağı sunucuyu belirtir (Docker'da `127.0.0.11`) |

---

## 🧠 Quiz

| # | Soru | Doğru Cevap |
|---|---|---|
| 1 | Düz Nginx ile OpenResty arasındaki temel fark nedir? | OpenResty, Nginx'in üzerine gömülmüş bir Lua interpreter'ı sayesinde içinde kod çalıştırabilir (token kontrol, veritabanı sorgusu vb.) |
| 2 | `nginx.conf` içindeki `resolver 127.0.0.11 valid=30s;` satırı neden gereklidir? | 127.0.0.11, Docker'ın iç DNS sunucusudur — Nginx, `postgres`/`mysql`/`redis` gibi container isimlerini bu olmadan IP'ye çeviremez |
