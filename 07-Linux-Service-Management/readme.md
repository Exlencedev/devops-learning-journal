# 🏗️ Linux Servis & Log Yönetimi (`systemd` Mimarisi)

Bu belge, `systemd` servis yönetimini (`enable`/`start`/`reload`/`restart`) ve `journalctl` ile log analizini kapsıyor.

---

## 1. Servis Etkinleştirme (`enable`) & Başlatma (`start`)

```bash
systemctl status nginx
```
Rocky Linux'ta nginx, kurulumdan sonra varsayılan olarak **devre dışı** (`disabled`) — Ubuntu'nun aksine otomatik açılmaz.

```bash
sudo systemctl enable nginx
```

### 🐛 Hata & Çözüm

İlk denemede `nginx` yerine yanlışlıkla **`nignx`** yazıldı (harfler yer değiştirdi):
```
Failed to enable unit: Unit nignx.service does not exist
```
Dikkatli yazımla (`nginx`) tekrar denendi ve başarılı oldu.

```bash
sudo systemctl start nginx
systemctl status nginx
```

**Çıktı:**
```
Loaded: loaded (/usr/lib/systemd/system/nginx.service; enabled; preset: disabled)
Active: active (running) since Fri 2026-08-07 18:15:51 +03; 3s ago
```

### 🔍 Kavramsal Ayrım

`start` ile `enable` **birbirinden bağımsız** iki ayardır:
- **`start`** → servisi **şu an** çalıştırır (yeniden başlatmada kalıcı değil).
- **`enable`** → servisin **her sistem açılışında** otomatik başlamasını sağlar.

İkisi bir arada kullanıldığında hem "şimdi çalışıyor" hem "her zaman otomatik başlayacak" durumu elde edilir.

---

## 2. `reload` vs `restart`

```bash
sudo systemctl reload nginx
sudo systemctl restart nginx
```

Görünürde ikisi de sessizce (çıktısız) tamamlandı, ama `journalctl` ile log geçmişine bakınca fark net ortaya çıktı (bkz. Bölüm 3).

| Komut | Davranış | Kesinti |
|-------|----------|---------|
| **`reload`** | Aktif bağlantıları kesmeden sadece yapılandırmayı yeniler | Sıfır kesinti |
| **`restart`** | Servisi tamamen durdurup yeniden başlatır | Kısa kesinti |

---

## 3. `journalctl` ile Log Analizi

### Tam log geçmişi

```bash
journalctl -u nginx
```

Log çıktısında `reload` ve `restart` arasındaki fark somut olarak görüldü:
- **`reload`** → sadece `Reloading nginx.service` → `Reloaded nginx.service` (worker process'ler kesintisiz devam etti)
- **`restart`** → tam bir durdur-başlat döngüsü: `Stopping` → `nginx.service: Deactivated successfully` → `Stopped` → `Starting` → config syntax kontrolü → `Started`

### Ciddiyet seviyesine göre filtreleme (`-p`)

```bash
journalctl -u nginx -p err
```
**Çıktı:** `-- No entries --` — hiçbir hata kaydı yok, nginx'in tamamen sorunsuz çalıştığının kanıtı.

Systemd log ciddiyet seviyeleri (en kritikten en az kritiğe): `emerg` → `alert` → `crit` → `err` → `warning` → `notice` → `info` → `debug`. `-p err` denildiğinde `err` **ve daha ciddi olan** her şey (crit, alert, emerg) gösterilir.

### Zamana göre filtreleme (`--since`)

```bash
journalctl -u nginx --since "1 hour ago"
```
Aynı log geçmişi göründü çünkü tüm test işlemleri zaten son 1 saat içinde yapılmıştı.

### 🔍 Gerçek Dünya Kullanımı

İki filtreyi birleştirmek, gerçek bir troubleshooting senaryosunda hızlı sonuç sağlar:
```bash
journalctl -u nginx -p err --since "1 hour ago"
```
Bu, "son bir saatte sadece hata seviyesi ve üstünü göster" demektir — alakasız log geçmişini elemeden doğrudan sorunlu satırlara gidilir.

---

## 📊 Komut Referansı

| Komut | Amacı | Örnek | Notlar |
|-------|-------|-------|--------|
| **`systemctl start`** | Servisi şimdi başlatır | `sudo systemctl start nginx` | Kalıcı değildir |
| **`systemctl enable`** | Açılışta otomatik başlatma ayarlar | `sudo systemctl enable nginx` | `start`'tan bağımsız |
| **`systemctl stop`** | Çalışan servisi durdurur | `sudo systemctl stop nginx` | |
| **`systemctl restart`** | Durdurup yeniden başlatır | `sudo systemctl restart nginx` | Kısa kesinti |
| **`systemctl reload`** | Kesintisiz yapılandırma yeniler | `sudo systemctl reload nginx` | Tüm servisler desteklemez |
| **`journalctl -u`** | Belirli birim için log gösterir | `journalctl -u nginx` | |
| **`journalctl -p`** | Ciddiyet seviyesine göre filtreler | `journalctl -u nginx -p err` | emerg→debug sıralaması |
| **`journalctl --since`** | Zaman aralığına göre filtreler | `journalctl --since "1 hour ago"` | Göreli veya mutlak |

---

## 🧠 Quiz (Gerçek Sonuçlar)

| # | Soru | Cevap | Sonuç |
|---|------|-------|-------|
| 1 | `systemctl start` ile `systemctl enable` arasındaki fark nedir? | `start` sadece şu an çalıştırır; `enable` servisin her açılışta otomatik başlamasını sağlar — birbirinden bağımsız ayarlar | ✅ Doğru |
| 2 | Log çıktısında `reload` ile `restart` farkı nasıl gözlemlendi? | `reload` bağlantıları kesmeden yapılandırmayı yeniler (sıfır kesinti); `restart` tamamen durdurup başlatır (kısa kesinti) | ✅ Doğru |

---

ℹ️ _Tüm adımlar yerel Rocky Linux VM'inde (Rocky9-Test) test edilmiştir._
