# Faz 20 — Nginx: Rate Limiting ve Load Balancing

## 📝 Özet

19. fazdaki (Nginx Derinleşme) sunucu üzerine iki yeni yetenek eklendi: bir IP'nin belirli sürede kaç istek atabileceğini sınırlayan **rate limiting**, ve trafiği birden fazla backend arasında dağıtan **load balancing** (round-robin + failover). Testler aynı WSL2/Ubuntu sunucusunda yapıldı.

## 1. Rate Limiting

Bir IP'nin belirli bir sürede kaç istek atabileceğini sınırlar — brute force koruması, DDoS hafifletme ve API koruması için kullanılır.

### Zone Tanımı (`/etc/nginx/nginx.conf`, `http` bloğunun içine)
```nginx
limit_req_zone $binary_remote_addr zone=genel:10m rate=5r/s;
```
- `$binary_remote_addr` — kimin istek attığını IP bazında takip et
- `zone=genel:10m` — "genel" adında bir hafıza bölgesi, 10MB
- `rate=5r/s` — her IP için saniyede maksimum 5 istek

### Location'a Uygulama
```nginx
location / {
        limit_req zone=genel burst=10 nodelay;
        proxy_pass http://localhost:8080;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
}
```
- `burst=10` — ani yoğunluğa tolerans: aniden 10 istek atılırsa hepsi kabul edilir
- `nodelay` — burst kapsamındaki istekler sıraya alınmadan hemen işlenir

`/`, `/users/`, `/computers/` bloklarının hepsine aynı `limit_req` satırı eklendi. `/admin` path'ine eklenmedi — zaten `allow`/`deny` ile kısıtlı.

### Test
```bash
for i in {1..20}; do curl -s -o /dev/null -w "%{http_code}\n" http://localhost/; done
```
**Sonuç (backend düzeltildikten sonra):**
```
200 200 200 200 200 200 200 200 200 200 200
503 503 503 503 503 503 503 503 503
```
İlk 11 istek geçti, sonra `503` (rate limit aşıldı) gelmeye başladı — kaynak metinde 12 idi, çok yakın bir sayı, timing'e bağlı küçük farklılık normal.

## 2. Load Balancing

Aynı işi yapan birden fazla backend arasında trafiği dağıtır — bir instance çökerse diğeri devam eder.

### Upstream Bloğu (`sites-available/default`, `server` bloğundan önce)
```nginx
upstream users_backend {
        server localhost:3000;
        server localhost:3001;
}
```

### İkinci Instance
```bash
mkdir -p /tmp/users2
echo "<h1>Users Servisi — Instance 2</h1>" > /tmp/users2/index.html
cd /tmp/users2 && python3 -m http.server 3001 &
```

### `/users/` Bloğunun Güncellenmesi
```nginx
location /users/ {
        limit_req zone=genel burst=10 nodelay;
        proxy_pass http://users_backend/;
        proxy_set_header Host $host;
        proxy_set_header X-Real-IP $remote_addr;
}
```

### Round-Robin Testi
```bash
for i in {1..8}; do curl -s http://localhost/users/; echo; sleep 0.3; done
```
Her iki instance'a da (Instance 1: `<h1>Users Servisi</h1>`, Instance 2: `<h1>Users Servisi — Instance 2</h1>`) trafik gittiği doğrulandı. Dağılım kaynak metindeki gibi tam düzenli (1-2-1-2) değildi, ama bu beklenen bir durum — Nginx'in round-robin algoritması önceki isteklerin durumuna göre değişebilir.

### Failover Testi
```bash
kill $(lsof -t -i:3000)
for i in {1..4}; do curl -s http://localhost/users/; echo; sleep 0.3; done
```
**Sonuç:** Instance 1 kapatıldıktan hemen sonra, tüm istekler hiçbir hata vermeden Instance 2'ye yönlendi — kullanıcı hiçbir kesinti yaşamadı.

## 3. Diğer Load Balancing Yöntemleri (Uygulanmadı, Kavramsal)

Bu fazda round-robin (varsayılan) kullanıldı, ama iki alternatif daha önemli:

- **`least_conn`** — en az aktif bağlantısı olan backend'e gönderir. İstekler eşit sürmüyorsa (örn. dosya yükleme) round-robin adaletsiz olabilir.
- **`ip_hash`** — aynı IP her zaman aynı backend'e gider. Backend'de session bilgisi tutuluyorsa gereklidir (round-robin ile session kaybolabilir).

Bu fazda backend'ler stateless (Python'un basit HTTP sunucusu) olduğu için ikisine de ihtiyaç duyulmadı, ama production'da hangisinin ne zaman gerektiğini bilmek önemli.

---

## 🐛 Hata & Çözüm

### Hata 1: Round-robin testi ilk denemede sadece Instance 2'ye gitti
**Neden:** Instance 1 (port 3000) 19. fazdan kalan bir nedenle (muhtemelen önceki bir `restart` sırasında arka plan süreci sonlanmış) hiç çalışmıyordu — sadece Instance 2 (port 3001) dinliyordu. `upstream` bloğu, ulaşılamayan backend'i otomatik devre dışı bırakıp tüm trafiği sağlıklı olana yönlendirdi (bu aslında failover'ın kendisi, ama beklenmedik bir zamanda tetiklendi).
**Çözüm:** `sudo ss -tlnp | grep -E "3000|3001"` ile hangi portların gerçekten dinlediği kontrol edildi, Instance 1 tekrar başlatıldı.

### Hata 2: Instance 1 yeniden başlatıldıktan hemen sonra tüm trafik SADECE Instance 1'e gitti
**Neden:** İlk 6 istek çok hızlı art arda (bekleme olmadan) gönderilmişti — bu, Nginx'in round-robin sayacının henüz iki backend arasında düzgün dönmemiş olabileceğini düşündürdü.
**Çözüm:** İstekler arasına `sleep 0.3` eklenip tekrar denendiğinde her iki backend'e de trafik gittiği doğrulandı.

### Hata 3: Rate limiting testinde ilk 11 istek 200 yerine 502 döndü
**Neden:** Backend servisi (`localhost:8080`, `/` path'inin arkasındaki) önceki fazlardan/restart'lardan kalma bir nedenle hiç çalışmıyordu — `sudo ss -tlnp | grep 8080` boş sonuç verdi. 502 (Bad Gateway), rate limiting'in kendisiyle (503) hiç alakalı değildi, backend'e ulaşılamamasının sonucuydu.
**Çözüm:** Backend (`python3 -m http.server 8080`) yeniden başlatıldı, test tekrarlandığında beklenen sonuç (11× 200, sonra 9× 503) alındı.

**Genel ders:** Uzun süren, çok fazlı bir çalışmada arka planda `&` ile başlatılan geçici süreçler (backend simülasyonları), `systemctl restart nginx` gibi işlemlerden veya WSL'in kendi ara sıra oturum kesintilerinden etkilenip sessizce ölebiliyor. Her yeni faza başlarken `ss -tlnp` ile hangi portların gerçekten aktif olduğunu kontrol etmek, zaman kaybını önlüyor.

---

## 📊 Komut Referansı

| Komut/Direktif | Açıklama |
|---|---|
| `limit_req_zone $binary_remote_addr zone=X:10m rate=Nr/s;` | Rate limiting zone'u tanımlar (nginx.conf'un `http` bloğunda) |
| `limit_req zone=X burst=N nodelay;` | Zone'u bir `location`'a uygular |
| `upstream isim { server host:port; ... }` | Backend havuzu tanımlar |
| `proxy_pass http://upstream_adi/;` | Trafiği upstream havuzuna yönlendirir |
| `sudo ss -tlnp \| grep PORT` | Belirli bir portun gerçekten dinlenip dinlenmediğini kontrol eder |
| `kill $(lsof -t -i:PORT)` | Belirli bir portu kullanan süreci sonlandırır (failover testi için) |
| `for i in {1..N}; do curl -s -o /dev/null -w "%{http_code}\n" URL; done` | Art arda N istek gönderip sadece HTTP durum kodlarını gösterir |

---

## 🧠 Quiz

| # | Soru | Doğru Cevap |
|---|---|---|
| 1 | 20 istekli rate limiting testinde ilk 11 istek 502 (200 yerine) döndürdü. Sebep neydi? | Backend servisi (port 8080) önceki bir fazdan kalma nedenle hiç çalışmıyordu — 502, rate limiting'den (503) tamamen bağımsız bir sorundu |
| 2 | Instance 1 `kill` ile kapatıldıktan hemen sonra tüm istekler kesintisiz Instance 2'ye gitti. Bunu sağlayan neydi? | `upstream` bloğu, çökmüş bir backend'i otomatik olarak devre dışı bırakıp kalan sağlıklı backend'e yönlendirir |
