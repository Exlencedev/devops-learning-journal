# Faz 25 — Docker Güvenliği: Non-Root Container, .dockerignore, Image Scanning

## 📝 Özet

Docker güvenliği üç katmanda ele alındı: container içindeki kullanıcı yetkisi (non-root), image'a giren dosyalar (`.dockerignore`) ve image içindeki bilinen güvenlik açıkları (Trivy ile tarama). Üç konunun testi de WSL2/Ubuntu ortamında gerçek komutlarla tekrarlandı ve aşağıdaki çıktılar bu ortamdan alındı.

## 1. Non-Root Container

### Sorun
Varsayılan olarak container'lar `root` olarak çalışır. Uygulamaya sızan biri container içinde her şeye erişebilir.

### Çözüm — Kendi Kullanıcımızı Oluşturmak

`Dockerfile.root` (karşılaştırma için, kullanıcı tanımlamadan):
```dockerfile
FROM python:3.11-slim
WORKDIR /app
COPY app.py .
CMD ["python", "app.py"]
```

`Dockerfile.nonroot`:
```dockerfile
FROM python:3.11-slim
RUN useradd -m -u 1000 appuser
USER appuser
WORKDIR /home/appuser
COPY --chown=appuser:appuser app.py .
CMD ["python", "app.py"]
```

- `useradd -m -u 1000 appuser` → `appuser` adında kullanıcı oluşturur (`-m` home dizini, `-u 1000` kullanıcı ID'si).
- `USER appuser` → bundan sonraki komutlar ve container bu kullanıcıyla çalışır. **Bu satır olmadan kullanıcı oluşturulur ama container hâlâ root olarak çalışır.**
- `COPY --chown=appuser:appuser` → kopyalanan dosyaların sahibi `appuser` olur.

### Test
```bash
sudo docker build -f Dockerfile.root -t python-root . -q
sudo docker build -f Dockerfile.nonroot -t python-nonroot . -q
sudo docker run --rm python-root whoami
sudo docker run --rm python-nonroot whoami
```
**Sonuç:**
```
root
appuser
```

Yetki farkı, container içinde `/etc/passwd`'ı silmeyi deneyerek gösterildi (komut yalnızca geçici container içinde çalışır, WSL sistemini etkilemez):
```bash
sudo docker run --rm python-root rm /etc/passwd; echo "root çıkış kodu: $?"
sudo docker run --rm python-nonroot rm /etc/passwd; echo "non-root çıkış kodu: $?"
```
**Sonuç:**
```
root çıkış kodu: 0
rm: cannot remove '/etc/passwd': Permission denied
non-root çıkış kodu: 1
```
Root container sistem dosyasını silebildi; non-root container `Permission denied` aldı. Birisi container'a sızsa bile non-root kullanıcının yetkisiyle sınırlı kalır.

## 2. .dockerignore

### Sorun
`COPY . .` dendiğinde proje klasöründeki her şey image'a girer: `.env` (şifreler), `.git` (tüm geçmiş), Dockerfile'lar vb. Zarf benzetmesi: zarfa mektupla birlikte banka kartı şifresini de koyup postaya vermek gibi.

### Çözüm
```
# .dockerignore
.env
.git
*.md
__pycache__
*.pyc
Dockerfile*
tests/
.dockerignore
```
(`.dockerignore`'un kendisi de listeye eklendi — hangi dosyaların hariç tutulduğu bilgisinin image'a girmesi gereksiz.)

### Test
Test için `COPY . .` kullanan bir Dockerfile ve içinde sahte bir şifre olan `.env` dosyası kullanıldı:
```bash
echo "SECRET_KEY=cokgizlisifre123" > .env

cat > Dockerfile.copyall << 'EOF'
FROM python:3.11-slim
WORKDIR /app
COPY . .
CMD ["python", "app.py"]
EOF
```

`.dockerignore` OLMADAN (dosya `.dockerignore.bak` olarak kenara alınarak):
```bash
mv .dockerignore .dockerignore.bak
sudo docker build -f Dockerfile.copyall -t python-noignore . -q
sudo docker run --rm python-noignore ls -la /app
```
```
-rw-r--r-- 1 root root   66 ... .dockerignore.bak
-rw-r--r-- 1 root root   28 ... .env                 ← image'a girdi!
-rw-r--r-- 1 root root   69 ... Dockerfile.copyall
-rw-r--r-- 1 root root  151 ... Dockerfile.nonroot
-rw-r--r-- 1 root root   74 ... Dockerfile.root
-rw-r--r-- 1 root root   17 ... app.py
```

`.dockerignore` İLE:
```bash
mv .dockerignore.bak .dockerignore
sudo docker build -f Dockerfile.copyall -t python-withignore . -q
sudo docker run --rm python-withignore ls -la /app
```
Çıktıda `total 12` göründü ve `.env` ile Dockerfile'lar listede yer almadı; image'da yalnızca `app.py` kaldı. (Terminal çıktısı yapıştırılırken son satır yarım kalmıştı, ama `total 12` ve `.env`'in yokluğu sonucu net gösteriyor.)

## 3. Image Scanning (Trivy)

### Sorun
İndirilen image'ların içinde kütüphaneler var ve bunların bir kısmında bilinen güvenlik açıkları (CVE) olabilir. Farkında olmadan açık içeren bir image kullanılıyor olabilir.

### Kurulum
Kaynaktaki sürüm yerine GitHub API'den en güncel sürüm otomatik çekildi:
```bash
VER=$(curl -s https://api.github.com/repos/aquasecurity/trivy/releases/latest | grep tag_name | cut -d '"' -f4 | sed 's/^v//')
wget -q https://github.com/aquasecurity/trivy/releases/download/v${VER}/trivy_${VER}_Linux-64bit.deb
sudo dpkg -i trivy_${VER}_Linux-64bit.deb
trivy --version
```
**Sonuç:** `Version: 0.75.0`

### Tarama
```bash
trivy image python:3.11 --severity HIGH,CRITICAL 2>/dev/null | grep -B2 "^Total:"
trivy image python:3.11-slim --severity HIGH,CRITICAL 2>/dev/null | grep -B2 "^Total:"
```

### Sonuçlar
Her iki image de Debian 13.7 tabanlı. Trivy her hedef için (işletim sistemi paketleri, Python paketleri) ayrı `Total:` satırı basar:

| İmaj | OS paketleri | Python paketleri | Toplam |
|---|---|---|---|
| `python:3.11` | 330 (HIGH 314, CRITICAL 16) | 2 (HIGH 2) | **332** |
| `python:3.11-slim` | 44 (HIGH 44, CRITICAL 0) | 2 (HIGH 2) | **46** |

Sadece `slim` image kullanarak açık sayısı yaklaşık **7 kat** azaldı (332 → 46) ve CRITICAL açıkların hepsi (16) ortadan kalktı. Python paketleri kısmı iki imajda aynı; fark işletim sistemi paketlerinden geliyor — tam imajda onlarca ekstra kütüphane ve araç var, slim'de yok.

> **Kaynak metinle karşılaştırma:** Kaynakta oranlar 412 → 20 (yaklaşık 20 kat) çıkmıştı. Burada fark daha küçük (7 kat). Trivy'nin güvenlik açığı veritabanı sürekli güncellendiği için sayılar taramanın yapıldığı güne göre değişir; yön ve sonuç (slim çok daha az açık içeriyor) aynı.

Slim image'da bulunan Python paketi açıklarından örnekler (gerçek çıktıdan):
```
wheel (METADATA)          CVE-2026-24049  HIGH  0.45.1 → 0.46.2  (Privilege Escalation / Arbitrary Code Execution)
jaraco.context (METADATA) CVE-2026-23949  HIGH  5.3.0  → 6.1.0   (Path traversal via malicious tar archives)
```
Her açık için Trivy şunu gösterir: CVE kimlik numarası, tehlike seviyesi (HIGH/CRITICAL), etkilenen kütüphane ve sürümü, düzeldiği sürüm (`Fixed Version`).

## 4 Katmanlı Güvenlik Özeti

| Yöntem | Ne Sağlıyor |
|---|---|
| Slim/alpine image | Az paket → az güvenlik açığı (bu testte 332 → 46) |
| Non-root user | Sızılsa bile sınırlı yetki (`rm /etc/passwd` → Permission denied) |
| `.dockerignore` | Şifreler ve gereksiz dosyalar image'a girmiyor |
| Trivy | Mevcut açıkları tespit et, `Fixed Version`'a göre düzelt |

---

## 🐛 Hata & Çözüm

### Hata 1: `.dockerignore` testinin ilk denemesi geçersiz çıktı
**Belirti:** "`.dockerignore` OLMADAN" diye çalıştırılan build'in `ls` çıktısında da `.env` görünmedi — yani iki test arasında fark gösterilemedi.
**Neden:** Test bloğu `.dockerignore` dosyasını önce kenara almadan "olmadan" build'ini çalıştırdı; dosya bir önceki denemeden zaten klasörde duruyordu, dolayısıyla "olmadan" testi aslında "ile" testiydi.
**Çözüm:** `mv .dockerignore .dockerignore.bak` ile dosya gerçekten kaldırıldıktan sonra build tekrarlandı; bu sefer `.env` image'a girdi (28 byte). **Ders:** karşılaştırma testlerinde başlangıç durumunu (`ls -la` ile) doğrulamak gerekir; aksi halde iki test aynı koşulu ölçebilir ve fark görünmez.

### Hata 2: İlk Trivy taramasında çıktı gelmedi
**Belirti:** `python:3.11` taramasında `=== python:3.11 ===` başlığından sonra uzun süre hiçbir şey görünmedi.
**Neden:** İlk çalıştırmada Trivy, güvenlik açığı veritabanını indirip büyük `python:3.11` image'ını tarıyordu; ayrıca `2>/dev/null` hata ve ilerleme mesajlarını gizlediği için komut donmuş gibi görünüyordu.
**Çözüm:** Komutun bitmesi beklendi, `2>&1 | tail` ile hata akışı da görünür yapılarak slim image taraması doğrulandı; veritabanı indikten sonra her iki image de sorunsuz tarandı.

---

## 📊 Komut Referansı

| Komut | Açıklama |
|---|---|
| `useradd -m -u 1000 appuser` | Home dizini olan, ID'si 1000 olan bir kullanıcı oluşturur |
| `USER <kullanıcı>` (Dockerfile) | Sonraki komutları ve container'ı bu kullanıcıyla çalıştırır |
| `COPY --chown=<k>:<g> <kaynak> <hedef>` | Kopyalanan dosyanın sahibini ayarlar |
| `docker run --rm <imaj> whoami` | Container içinde çalışan kullanıcıyı gösterir (`--rm`: bitince container'ı sil) |
| `.dockerignore` | `COPY . .` sırasında image'a girmemesi gereken dosyaların listesi |
| `trivy image <imaj> --severity HIGH,CRITICAL` | Image'ı bilinen güvenlik açıklarına karşı tarar |
| `trivy --version` | Trivy sürümünü gösterir |

---

## 🧠 Quiz

| # | Soru | Doğru Cevap |
|---|---|---|
| 1 | Dockerfile'da `RUN useradd -m -u 1000 appuser` var ama `USER appuser` yok. Container `whoami` ile sorgulanırsa ne döner? | `root` — `useradd` sadece kullanıcıyı oluşturur, çalışma kullanıcısını `USER` satırı değiştirir |
| 2 | Trivy'de `python:3.11` (332) ile `python:3.11-slim` (46) arasındaki açık sayısı farkının asıl sebebi nedir? | Tam imajda çok sayıda ekstra sistem paketi var; fark işletim sistemi paketlerinden geliyor (Python paketleri iki imajda aynı) |

---

ℹ️ Tüm testler WSL2/Ubuntu 26.04 ortamında Docker ve Trivy 0.75.0 ile yapılmıştır. Trivy sayıları taramanın yapıldığı günkü güvenlik açığı veritabanına göredir.
