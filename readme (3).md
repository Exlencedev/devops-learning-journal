# 🔀 Forward Proxy vs Reverse Proxy

Bu belge, forward proxy ve reverse proxy arasındaki farkı, Nginx'in reverse
proxy olarak ne yaptığını ve ilk kurulum denemesinde karşılaşılan 502
hatasını kapsar.

> 💡 Bu fazda kavramsal temel oluşturuldu. Nginx'in derinlemesine
> uygulaması (path bazlı yönlendirme, path rewrite, path engelleme, forward
> proxy kurulumu) sonraki fazda tamamlandı: bkz.
> [19-Nginx-Derinlestirme](../19-Nginx-Derinlestirme/).

---

## 1. Forward Proxy

İstemcinin önünde durur. Sizin adınıza internete istek gönderir — hedef
sunucu sizi değil, proxy'yi görür.

```
İstemci → Forward Proxy → Hedef Sunucu (örn. google.com)
```

**Kullanım alanları:**
- Kurumsal ağlarda internet erişimini kontrol etme/filtreleme
- İstemcinin gerçek IP'sini gizleme
- İçerik filtreleme

**Benzetme:** Turnike gibi — çalışanlar dışarıya çıkarken turnike'den
geçer. Dışarıdaki siteler çalışanı değil, şirketi görür. Şirket ise kimin
nereye gittiğini kontrol edip loglayabilir.

---

## 2. Reverse Proxy

Sunucu(lar)ın önünde durur. İstemciler proxy'ye bağlanır, proxy isteği
asıl backend sunucusuna iletir. İstemci arkada ne çalıştığını bilmez.

```
İstemci → Reverse Proxy (Nginx) → Backend Sunucu
```

**Kullanım alanları:**
- **Load balancing** — trafiği birden fazla backend sunucusuna dağıtma
- **SSL sonlandırma** — HTTPS'i proxy'de halletme, backend'e düz HTTP gönderme
- **Tek giriş noktası** — bir domain altında birden fazla servisi (API, frontend, admin paneli) path'e göre yönlendirme
- **Güvenlik** — backend sunucuların gerçek IP/yapısını dışarıdan gizleme

**Benzetme:** Büyük bir şirketteki danışma görevlisi gibi — "randevunuz
var mıydı, kime gidiyordunuz?" diye sorarak sizi doğru yere yönlendirir.
Binanın iç yapısını bilmiyorsunuz, sadece danışmayla konuşuyorsunuz.

Bu, Nginx'in `location` bloklarıyla doğrudan örtüşür:

```nginx
location /api/ {
    proxy_pass http://backend1:8080;
}
location /admin/ {
    proxy_pass http://backend2:9090;
}
```

Danışmanın "muhasebe mi, İK mı?" diye sorması gibi — farklı path'ler
farklı backend'lere yönlendirilir.

---

## 3. İlk Reverse Proxy Denemesi ve 502 Hatası

### Ne Yapıldı

Backend olarak port 8080'de basit bir Python HTTP sunucusu başlatıldı:

```bash
python3 -m http.server 8080
```

Nginx config'inde `location /` bloğu değiştirildi:

```nginx
location / {
    proxy_pass http://localhost:8080;
}
```

Config test edildi ve Nginx yeniden başlatıldı:

```bash
sudo nginx -t
sudo systemctl restart nginx
```

`curl localhost` denendi → **502 Bad Gateway** alındı.

### 🐛 Neden Başarısız Oldu

Python backend ve Nginx farklı VM'lerde çalışıyordu. `localhost` her
zaman "bu makine" anlamına gelir — Nginx (Ubuntu'da) `proxy_pass
http://localhost:8080` denediğinde, kendi üzerinde bir backend arıyordu,
backend'in gerçekten çalıştığı diğer VM'de değil. Ubuntu'nun kendi 8080
portunda hiçbir şey dinlemiyordu, bu yüzden 502 geldi.

### 502 Bad Gateway Ne Anlama Gelir

**502 Bad Gateway** = "Ben (proxy) isteğinizi iletmeye çalıştım ama
hedefe ulaşamadım." Yaygın sebepler:

- Backend servisi çalışmıyor
- `proxy_pass`'te yanlış IP/port
- Backend farklı bir makinede ve `localhost` oraya işaret etmiyor
- Proxy ile backend arasında bağlantıyı engelleyen bir firewall

### ✅ Düzeltme

Ya Nginx ve backend aynı makinede çalıştırılır (böylece `localhost`
geçerli olur), ya da `proxy_pass`'te `localhost` yerine backend VM'in
gerçek IP adresi kullanılır.

Bu hata, sonraki fazda gerçek bir sunucuda, backend ve Nginx aynı
makinede çalıştırılarak düzeltildi.

---

## 📊 Hızlı Referans

| Terim | Ne Yapar | Gerçek Hayat Benzetmesi |
|-------|----------|--------------------------|
| **Forward Proxy** | İstemcinin önünde durur, onun adına istek gönderir | Çalışanların geçtiği turnike |
| **Reverse Proxy** | Sunucunun önünde durur, gelen istekleri backend'e yönlendirir | Ziyaretçileri doğru ofise yönlendiren danışma görevlisi |
| **proxy_pass** | İstekleri başka bir adrese/porta ileten Nginx direktifi | Danışmanın sizi nereye göndereceğini söyleyen talimat |
| **502 Bad Gateway** | Proxy, yönlendirildiği backend'e ulaşamadı | Gönderildiğiniz ofis cevap vermiyor |

---

## 🧠 Quiz (Çoktan Seçmeli)

**S1:** Forward proxy ile reverse proxy arasındaki temel fark nedir?

- A) Forward proxy istemcinin önünde durur ve onun adına istek gönderir; reverse proxy sunucunun önünde durur ve gelen istekleri backend'e yönlendirir ✅
- B) İkisi de aynı işlevi görür, sadece isim farklıdır
- C) Forward proxy sadece HTTPS trafiğini yönetir
- D) Reverse proxy sadece load balancing yapar, forward proxy yapamaz

**S2:** İlk reverse proxy denemesinde neden 502 Bad Gateway hatası alındı?

- A) Nginx config dosyasında syntax hatası vardı
- B) Backend ve Nginx farklı VM'lerde çalışıyordu ve `proxy_pass`'te `localhost` kullanılmıştı ✅
- C) Firewall SSH portunu engelliyordu
- D) SELinux, Nginx'in proxy yapmasını engelliyordu

---

ℹ️ _Bu faz kavramsal temel olarak işlendi. Hands-on kurulum ilk denemede
502 hatasıyla yarım kaldı — hatanın neden oluştuğu anlaşıldı ve
19-Nginx-Derinlestirme fazında gerçek bir sunucuda tamamlandı._
