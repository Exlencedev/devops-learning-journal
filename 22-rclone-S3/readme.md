# Faz 22 — rclone & Amazon S3: Bulut Depolama ve Güvenli Erişim

## 📝 Özet

Bu fazda gerçek bir AWS hesabı açılıp bir S3 bucket'ı ve IAM kullanıcısı oluşturuldu, ardından `rclone` ile bu bucket'a bağlanıp performans testleri, `rclone serve http` (private S3'ü güvenli şekilde dışarıya açma), ve `rclone mount` (S3'ü yerel disk gibi kullanma) test edildi. WSL2/Ubuntu ortamının ağ katmanı, kaynak metindeki gerçek bir VDS'ye kıyasla performans testlerinde önemli farklar yarattı — bu fark, journal'ın en değerli bulgularından biri oldu.

## 1. AWS Kurulumu

Gerçek bir AWS hesabı açıldı ve şu kaynaklar oluşturuldu:
- **S3 Bucket:** `ege-devops-journal-1`, `eu-central-1` (Frankfurt) bölgesinde
- **IAM Kullanıcısı:** `rclone-user`, `AmazonS3FullAccess` politikasıyla (öğrenme ortamı için basit tutuldu; production'da daha dar kapsamlı bir politika tercih edilir)
- **Access Key / Secret Key:** CLI kullanımı için oluşturuldu

## 2. rclone Kurulumu ve S3 Yapılandırması

### Kurulum
```bash
sudo apt install unzip -y   # rclone kurulum scripti zip açmak için gerekli
curl https://rclone.org/install.sh -o install-rclone.sh
sudo bash install-rclone.sh
```

### S3 Remote Yapılandırması (`rclone config`)
Sihirbaz üzerinden şu seçimler yapıldı:
- Storage: `4` (Amazon S3 Compliant Storage Providers)
- Provider: AWS
- env_auth: `1` (kimlik bilgilerini elle gir)
- access_key_id / secret_access_key: AWS Console'dan alınan değerler
- region: `11` (eu-central-1 — Frankfurt)
- endpoint: boş (AWS için gerekmiyor)
- **location_constraint: `eu-central-1` (elle yazıldı — listede doğrudan seçenek olarak yoktu, `region` ile birebir eşleşmesi gerektiği için manuel girildi)**
- acl: `1` (private)
- server_side_encryption, sse_kms_key_id, storage_class, bucket_object_lock_enabled: hepsi boş bırakıldı (varsayılan)

### Doğrulama
```bash
rclone ls s3:ege-devops-journal-1
```
Boş çıktı (bucket boştu) — hata almadan çalıştığı için bağlantı başarılı.

## 3. Performans Testleri

### Test Ortamı
10 tane 5MB'lık rastgele dosya (toplam 50MB):
```bash
mkdir -p ~/test-files
for i in $(seq 1 10); do
  dd if=/dev/urandom of=~/test-files/file$i.bin bs=1M count=5 2>/dev/null
done
```

### Sonuç Tablosu

| Test | Süre | Hız |
|---|---|---|
| Varsayılan ayarlar | 41.4s | ~1.1 MiB/s |
| `--transfers 16 --checkers 16 --buffer-size 16M --fast-list` | 37.8s | ~1.28 MiB/s |
| `--transfers 4` (fast-list izole testi, temel) | 37.3s | ~1.26 MiB/s |
| `--transfers 4 --fast-list` | 38.1s | ~1.20 MiB/s |

### 🔍 Kaynak Metinle Karşılaştırma — Önemli Bir Fark
Kaynak metinde (gerçek bir Ubuntu VDS üzerinde) aynı testler **1-1.5 saniye** sürmüş ve `--fast-list` gibi parametreler **%25'e varan** fark yaratmıştı. Bu WSL2/Ubuntu ortamında ise:
- Tüm testler **37-42 saniye** aralığında kaldı (VDS'ye göre ~30 kat daha yavaş).
- Performans parametrelerinin (paralel transfer sayısı, `--fast-list`) **pratik olarak hiçbir fark yaratmadığı** gözlemlendi.

**Yorum:** Bu, WSL2'nin ağ katmanının (Windows host üzerinden NAT'lanmış çıkış yolu) S3'e giden bağlantıda ciddi bir gecikme/bant genişliği sınırlaması getirdiğini gösteriyor. Darboğaz zaten ağ katmanında olduğu için, `--fast-list` gibi listeleme overhead'ini azaltan parametrelerin faydası görünür olmadı — bu parametreler ancak darboğaz kendi hitap ettikleri noktadaysa (çok sayıda küçük dosyada listeleme maliyeti gibi) fark yaratır.

## 4. `rclone serve http` — Private S3'ü Güvenli Şekilde Dışarıya Açmak

```bash
rclone serve http s3:ege-devops-journal-1 --addr :8090 &
```

Windows tarayıcısından `http://<WSL-IP>:8090` adresine gidildiğinde, S3'teki `test1/` ve `test2/` klasörleri ve içindeki tüm dosyalar (doğru boyutlarıyla, 5.00 MiB) görüntülendi — **hiçbir AWS kimlik bilgisi kullanılmadan**. Bucket hâlâ private, ama rclone bir güvenli ara katman görevi görüyor (Nginx'teki reverse proxy mantığıyla aynı).

## 5. `rclone mount` — S3'ü Yerel Disk Gibi Kullanmak

### Cache Olmadan
```bash
rclone mount s3:ege-devops-journal-1 ~/s3mount --daemon
time cat ~/s3mount/test1/file1.bin > /dev/null
# real 0m2.413s — her okuma S3'e gidiyor
```

### Cache ile
```bash
fusermount3 -u ~/s3mount   # önce kapat
rclone mount s3:ege-devops-journal-1 ~/s3mount \
  --vfs-cache-mode full \
  --vfs-cache-max-size 2G \
  --vfs-cache-max-age 24h \
  --daemon
```

**Test Sonuçları:**
| Okuma | Süre | Ne oldu |
|---|---|---|
| 1. (ilk) | 2.566s | S3'ten indirdi, yerel diske (cache'e) yazdı |
| 2. (ikinci) | 0.009s | Doğrudan cache'ten okundu — **~285 kat daha hızlı** |

Bu, kaynak metindeki 42 kat hızlanma bulgusunun bu ortamda çok daha güçlü bir versiyonu — WSL2'nin ağ gecikmesi yüksek olduğu için, cache'in getirdiği fayda (ağa hiç gitmeme) mutlak olarak daha da büyük çıktı.

## 6. `rclone serve http` — Cache, Auth ve Remote Control Birlikte

### Tam Komut
```bash
export RCLONE_USER=admin
export RCLONE_PASS=gizlisifre123

rclone serve http s3:ege-devops-journal-1 --addr :8090 \
  --vfs-cache-mode full \
  --vfs-cache-max-size 10G \
  --vfs-cache-max-age 24h \
  --dir-cache-time 1h \
  --buffer-size 32M \
  --rc --rc-addr :5572 --rc-no-auth \
  --log-file ~/rclone-http.log \
  --log-level INFO &
```

### Auth Testi
```bash
curl -s -o /dev/null -w "%{http_code}\n" http://localhost:8090/
# → 401 (auth yok)

curl -s -o /dev/null -w "%{http_code}\n" -u admin:gizlisifre123 http://localhost:8090/
# → 200 (auth doğru)
```

### Şifre Maskeleme Doğrulaması
`~/rclone-http.log` incelendiğinde:
```
INFO  : Using --user admin --pass XXXX as authenticated user
```
Şifre, `RCLONE_PASS` environment variable ile verildiği için log'da gerçek değeriyle değil, `XXXX` olarak maskelenmiş görünüyor — komut satırı flag'i (`--pass`) kullanılsaydı log'da açık görünecekti.

### Dir Cache ve `vfs/forget`
Yeni bir dosya (`yenidosya.txt`) S3'e yüklenip HTTP arayüzünde hemen kontrol edildi.

**🔍 Kaynak Metinden Farklı Bir Sonuç:** Kaynak metinde `--dir-cache-time 1h` yüzünden yeni dosyanın 1 saat boyunca görünmemesi bekleniyordu (`rclone rc vfs/forget` ile manuel temizlemek gerekiyordu). Bu ortamda ise dosya **hemen** göründü — muhtemelen aynı rclone sürecinin (`rclone copy` ile `rclone serve http`'in aynı config/remote'u paylaşması, farklı bir kullanıcıdan/session'dan gelmemesi) dizin cache'ini otomatik olarak güncel tutmuş olabileceği düşünüldü. `rclone rc vfs/forget` yine de çalıştırılıp `--rc` arayüzünün fonksiyonel olduğu doğrulandı (boş bir `forgotten: []` listesi döndü, çünkü zaten cache güncel durumdaydı).

---

## 🐛 Hata & Çözüm

### Hata 1: rclone kurulum scripti "None of the supported tools for extracting zip archives" hatası verdi
**Neden:** `unzip`, `7z`, `busybox` — hiçbiri sistemde kurulu değildi.
**Çözüm:** `sudo apt install unzip -y` ile eksik araç kuruldu, kurulum tekrar çalıştırıldı.

### Hata 2: Access Key/Secret Key sayfası kapatıldıktan sonra bir daha görüntülenemedi
**Neden:** AWS, Secret Access Key'i güvenlik gereği sadece oluşturma anında bir kez gösterir.
**Çözüm:** Bu fazda CSV dosyası olarak önceden indirilmiş olduğu için kayıp yaşanmadı — CSV'den değerler tekrar okunabildi. (Not: CSV indirilmemiş olsaydı, eski key devre dışı bırakılıp yeni bir key oluşturulması gerekecekti.)

### Hata 3 (Kaynak Metinde de Var, Burada Önceden Önlendi): `location_constraint` için `EU` yerine `eu-central-1` yazılması gerekiyordu
Kaynak metinde bu hata gerçekten yapılıp `IllegalLocationConstraintException` alınmıştı. Bu fazda, kaynak metinden ders çıkarılarak, `location_constraint` sorulduğunda listede doğrudan seçenek olmasa da `eu-central-1` elle yazıldı ve hatasız geçildi.

### Beklenmedik Fark: Performans parametreleri hiçbir gözlemlenebilir fark yaratmadı
Kaynak metinde `--fast-list` gibi parametreler belirgin fark yaratmıştı; bu ortamda WSL2'nin ağ darboğazı her testte baskın olduğu için hiçbir performans parametresi ölçülebilir bir iyileşme sağlamadı. Bu bir "hata" değil, ama önemli bir gözlemdi — journal'a ayrıntılı not düşüldü (bkz. Bölüm 3).

### Beklenmedik Fark: Yeni dosya dizin cache'ine rağmen hemen göründü
Kaynak metinde 1 saatlik `--dir-cache-time` yüzünden yeni dosyanın gecikmeli görünmesi bekleniyordu; bu ortamda hemen göründü (bkz. Bölüm 6). Kesin kök neden belirlenemedi, ama muhtemelen aynı rclone config/process'in kendi cache'ini tutarlı tutması olası bir açıklama.

---

## 📊 Komut Referansı

| Komut | Açıklama |
|---|---|
| `rclone config` | Yeni bir cloud storage remote'u yapılandırır (etkileşimli sihirbaz) |
| `rclone ls <remote>:<bucket>` | Bucket içeriğini listeler |
| `rclone copy <kaynak> <remote>:<bucket>/<yol> -P` | Dosyaları yükler, ilerlemeyi gösterir |
| `rclone delete <remote>:<bucket>/<yol> --rmdirs` | Bir klasörü (içeriğiyle) siler |
| `rclone serve http <remote>:<bucket> --addr :PORT` | Private storage'ı HTTP üzerinden güvenli şekilde sunar |
| `rclone mount <remote>:<bucket> <yerel-dizin> --daemon` | Bulut depolamayı yerel bir disk gibi bağlar |
| `fusermount3 -u <yerel-dizin>` | Bir mount'u kapatır |
| `--vfs-cache-mode full` | Dosyaları yerel diske cache'ler (S3'e sürekli gitmeyi önler) |
| `--transfers N` / `--checkers N` | Paralel transfer/kontrol sayısı |
| `--fast-list` | Dizin listelemeyi tek API isteğiyle yapar |
| `rclone rc vfs/forget` | Çalışan bir rclone process'inin VFS cache'ini temizler |

---

## 🧠 Quiz

| # | Soru | Doğru Cevap |
|---|---|---|
| 1 | Kaynak metinde `--fast-list` %25 hız farkı yaratmıştı, bu ortamda pratik olarak hiçbir fark çıkmadı. Sebep neydi? | Bu ortamda darboğaz zaten ağ bant genişliğiydi; performans parametreleri başka tür darboğazları hedeflediği için burada fark yaratamadı |
| 2 | Cache'li mount ile ilk okuma 2.566s, ikincisi 0.009s sürdü. Bu farka sebep olan neydi? | `--vfs-cache-mode full` sayesinde ilk okuma dosyayı yerel diske cache'ledi; ikinci okuma S3'e hiç gitmeden doğrudan cache'ten okundu |
