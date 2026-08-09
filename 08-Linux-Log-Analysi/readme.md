# 📜 Linux Log Analizi & Metin İşleme (`sed`)

Bu belge, nginx access log analizini (`awk`/`grep` pipeline'ları) ve `sed` ile metin düzenlemeyi kapsıyor.

---

## 1. Test HTTP İstekleri Oluşturma

```bash
curl -I http://localhost/
curl -I http://localhost/aa
```

**Sonuç:** İlki `HTTP/1.1 200 OK`, ikincisi (var olmayan path) `HTTP/1.1 404 Not Found` döndü.

---

## 2. Log Satırı Formatını Okuma

```bash
sudo tail -n 5 /var/log/nginx/access.log
```

### 🐛 Hata & Çözüm

İlk denemede `-n 5` yerine yanlışlıkla `-n tt` yazıldı: `tail: geçersiz satır sayısı: 'tt'`. Sayısal değerle tekrar denendi.

**Log çıktısı:**
```
::1 - - [07/Aug/2026:18:21:57 +0300] "HEAD / HTTP/1.1" 200 0 "-" "curl/8.12.1" "-"
::1 - - [07/Aug/2026:18:22:00 +0300] "HEAD /aa HTTP/1.1" 404 0 "-" "curl/8.12.1" "-"
```

### Önemli Sütunlar

| Sütun | Değer | Anlamı |
|-------|-------|--------|
| `$1` | `::1` | İstemci IP adresi |
| `$7` | `/` veya `/aa` | İstenen path |
| `$9` | `200` veya `404` | HTTP durum kodu |

### 🔍 Beklenmedik Gözlem

Orijinal müfredat, Rocky Linux'ta loopback'in IPv4 (`127.0.0.1`) olacağını belirtiyordu, ama bu VM'de nginx logunda **IPv6 (`::1`)** görüldü. Bu, sistemler arası varsayılan davranışın konfigürasyona (nginx `listen` yönergesi, `/etc/hosts`) bağlı olarak garanti olmadığını gösteren gerçek bir örnek.

---

## 3. İşleme Pipeline'ları

### En Çok İstek Gönderen IP'yi Bulma

```bash
sudo awk '{print $1}' /var/log/nginx/access.log | sort | uniq -c | sort -nr
```
**Çıktı:** `2 ::1` — iki test isteğimiz de aynı IP'den geldiği için doğru sayıldı.

**Pipeline mantığı:**
- `awk '{print $1}'` — IP'yi (Sütun 1) çıkarır.
- `sort` — aynı girdileri yan yana getirir.
- `uniq -c` — her IP'nin kaç kez geçtiğini sayar.
- `sort -nr` — sayısal olarak büyükten küçüğe sıralar.

### Path Bazında 404 Hatalarını Sayma

```bash
sudo grep " 404 " /var/log/nginx/access.log | awk '{print $7}' | sort | uniq -c | sort -nr
```

### 🐛 Hata & Çözüm

İlk denemede `awk '{print $7}'` yerine yanlışlıkla `awk '{pring $7}'` yazıldı (`print` → `pring`), bu yüzden çıktı boş geldi. Düzeltilip tekrar çalıştırıldı.

**Çıktı:** `1 /aa` — gerçek dünyada bu pipeline, "hangi path'ler en çok 404 üretiyor?" sorusuna cevap vermek için kullanılır.

---

## 4. `sed` ile Metin Düzenleme

`awk` sütun seçmek için, `sed` (**s**tream **ed**itor) ise metin bulup değiştirmek/silmek için kullanılır.

### Test Dosyası Oluşturma

```bash
echo "satir1 dunya" > sed-test.txt
echo "satir2 Dunya" >> sed-test.txt
echo "satir3 test" >> sed-test.txt
echo "satir4 son" >> sed-test.txt
```

### 🐛 Hata & Çözüm

`printf` ile tek satırda `\n` kullanarak dosya oluşturma denemesi, Türkçe klavyede `\` karakterinin doğru basılamaması yüzünden başarısız oldu. Bunun yerine her satır ayrı bir `echo` komutuyla (`>` ilk satır için, `>>` sonrakiler için) eklendi — `\` karakterine hiç ihtiyaç duyulmadı.

### Büyük/Küçük Harf Duyarlılığı

```bash
sed 's/dunya/world/' sed-test.txt
```
**Çıktı:** Sadece küçük harfli `dunya` → `world` değişti, `Dunya` aynı kaldı. `sed` **varsayılan olarak büyük/küçük harfe duyarlı**.

```bash
sed 's/dunya/world/I' sed-test.txt
```
**Çıktı:** `I` flag'i ile hem `dunya` hem `Dunya` → `world` değişti — büyük/küçük harf duyarsız eşleştirme.

### Kalıcı Değişiklik (`-i`)

```bash
sed -i 's/dunya/world/I' sed-test.txt
cat sed-test.txt
```
`-i` (in-place) olmadan `sed` sadece ekrana basar, dosya değişmez. `-i` ile değişiklik **dosyaya kalıcı olarak yazılır** — `cat` ile kontrol edildiğinde değişikliğin kalıcı olduğu doğrulandı.

### Satır Silme (Numaraya Göre)

```bash
sed -i '2d' sed-test.txt
cat sed-test.txt
```
**Sonuç:** 2. satır kalıcı olarak silindi, geriye 3 satır kaldı.

---

## 📊 Komut Referansı

| Komut | Amacı | Örnek | Notlar |
|-------|-------|-------|--------|
| **`curl -I`** | Sadece HTTP başlıklarını gösterir (test isteği) | `curl -I http://localhost/` | |
| **`tail -n`** | Son N satırı bir kez gösterir | `tail -n 5 access.log` | |
| **`awk '{print $N}'`** | Belirli sütunu çıkarır | `awk '{print $1}'` | |
| **`sort \| uniq -c`** | Sıralayıp tekrar sayısını gösterir | `sort \| uniq -c` | Önce sıralanmış girdi gerekir |
| **`sed 's/x/y/'`** | İlk eşleşmeyi değiştirir (ekrana basar) | `sed 's/eski/yeni/'` | Dosyayı değiştirmez |
| **`sed 's/x/y/I'`** | Büyük/küçük harf duyarsız değiştirme | `sed 's/dunya/world/I'` | |
| **`sed -i`** | Değişikliği dosyaya kalıcı yazar | `sed -i 's/x/y/' dosya` | Olmadan sadece ekrana basılır |
| **`sed 'Nd'`** | Belirli satırı numarasına göre siler | `sed '2d' dosya` | |

---

## 🧠 Quiz (Gerçek Sonuçlar)

| # | Soru | Cevap | Sonuç |
|---|------|-------|-------|
| 1 | `sed` varsayılan olarak büyük/küçük harfe duyarlı mı? `I` flag'i ne işe yarar? | `sed` varsayılan olarak duyarlıdır; `I` flag'i eklenince duyarsız hale gelir, ikisini de eşleştirir | ✅ Doğru |
| 2 | `sed -i` ile `-i` olmadan `sed` arasındaki fark nedir? | `-i` olmadan sadece ekrana basılır, dosya değişmez; `-i` ile değişiklik dosyaya kalıcı yazılır | ✅ Doğru |

---

ℹ️ _Tüm işlemler yerel Rocky Linux VM'inde (Rocky9-Test) test edilmiştir._
