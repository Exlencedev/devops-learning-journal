# Faz 24 — Docker: Image, Container, Dockerfile ve Image Optimizasyonu

## 📝 Özet

Bu faz, Docker'ın temel kavramlarını (image vs container, Dockerfile vs docker-compose.yml) derinleştirdi ve image optimizasyon tekniklerini (multi-stage build, layer caching, RUN birleştirme) kavramsal olarak işledi. Volume kalıcılığı ve network isim çözümlemesi WSL2/Ubuntu ortamında gerçek testlerle doğrulandı.

## 1. Sanal Makine vs Container

| | Sanal Makine | Container |
|---|---|---|
| Kernel | Kendi kernel'i var | Host'un kernel'ini paylaşır |
| Ağırlık | Ağır, yavaş başlar | Hafif, hızlı başlar |
| İçerik | Tam işletim sistemi | Sadece servis için gerekenler |

## 2. Image vs Container

- **Image** = şablon/kalıp — çalışmıyor, sadece bekliyor. Docker Hub'dan çekilebilir ya da `Dockerfile` ile oluşturulabilir.
- **Container** = image'dan çalıştırılan kopya. Aynı image'dan birden fazla bağımsız container açılabilir.

## 3. Dockerfile vs docker-compose.yml

- **Dockerfile** → tek bir image nasıl oluşturulur, onu tarif eder.
- **docker-compose.yml** → birden fazla container'ın birlikte nasıl çalıştırılacağını tarif eder.

```yaml
openresty:
  build: .              # Dockerfile kullan, image oluştur
postgres:
  image: postgres:15    # hazır image kullan, Dockerfile yok
```

21. fazda (OpenResty) sadece OpenResty için Dockerfile yazılmıştı çünkü `pgmoon` eklenmesi gerekiyordu; PostgreSQL, MySQL, Redis için hazır image'lar yeterliydi.

## 4. Docker Compose — Volume ve Network Testi (Gerçek Test)

### Volume Kalıcılığı Testi
```bash
mkdir -p ~/compose-practice && cd ~/compose-practice
cat > docker-compose.yml << 'EOF'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: testpass
    volumes:
      - ./pgdata:/var/lib/postgresql/data
EOF
sudo docker compose up -d
```

Bir tabloya test verisi eklendi:
```bash
sudo docker exec compose-practice-postgres-1 psql -U postgres -c \
  "CREATE TABLE test_kalicilik (id serial, mesaj text); INSERT INTO test_kalicilik (mesaj) VALUES ('bu veri kalici mi');"
```

Container tamamen silinip yeniden oluşturuldu:
```bash
sudo docker compose down
sudo docker compose up -d
sudo docker exec compose-practice-postgres-1 psql -U postgres -c "SELECT * FROM test_kalicilik;"
```

**Sonuç:** Veri hiç kaybolmadı — `docker compose down`, varsayılan olarak volume'ları silmiyor (silmek için ayrıca `-v` bayrağı gerekir). Bilgisayar (container) atılıp yenisi alınabilir, ama harddisk (volume) çıkarılıp yeni bilgisayara takılınca veriler hâlâ duruyor.

### Network İsim Çözümlemesi Testi
İkinci bir servis (`pgadmin_test`) aynı Compose dosyasına eklendi:
```bash
cat > docker-compose.yml << 'EOF'
services:
  postgres:
    image: postgres:15
    environment:
      POSTGRES_PASSWORD: testpass
    volumes:
      - ./pgdata:/var/lib/postgresql/data

  pgadmin_test:
    image: alpine
    command: sh -c "apk add --no-cache postgresql-client > /dev/null 2>&1 && sleep 3600"
EOF
sudo docker compose up -d
sudo docker exec compose-practice-pgadmin_test-1 pg_isready -h postgres -p 5432
```

**Sonuç:** `postgres:5432 - accepting connections` — `pgadmin_test` container'ı, `postgres` servisine hiçbir IP adresi bilmeden, sadece Compose'daki servis ismiyle ulaştı. Docker'ın dahili DNS'i bu çözümlemeyi otomatik yapıyor (navigasyon uygulamasına evini "ev" diye kaydetmek gibi — sonra sadece "ev" yazınca oraya gidiliyor).

## 5. Windows Containers (Kavramsal)

Windows Server Core, container'lar için Microsoft'un hazırladığı GUI'siz minimal Windows image'ı — kavramsal olarak Alpine'ın Windows tarafındaki karşılığı, ama farklı bir kernel ailesine dayanıyor.

```dockerfile
FROM mcr.microsoft.com/windows/servercore:ltsc2022
COPY app/ C:\app\
WORKDIR C:\app
RUN powershell -Command "Install-WindowsFeature -Name Web-Server"
EXPOSE 80
CMD ["powershell"]
```

**Neden Linux VPS'de çalışmaz:** Container kendi kernel'ini taşımaz, host'un kernel'ini kullanır. Windows container, Windows kernel'i gerektirir — Linux kernel'i çalıştıran bir sunucuda hiçbir şekilde çalışmaz (Nintendo kartuşunun sadece Nintendo konsoluna takılması gibi).

**"Docker her yerde çalışır" ifadesinin doğru anlamı:** "herhangi bir işletim sisteminde çalışır" değil, "aynı kernel ailesi içinde tutarlı çalışır" demek. Mac/Windows'ta Docker Desktop'ın Linux container'ları çalıştırabilmesinin sebebi de bu — arka planda gizli bir Linux sanal makinesi kuruluyor, container'lar aslında o VM'in kernel'inde çalışıyor.

## 6. Dockerfile Optimizasyonu (Kavramsal)

### a) Doğru Base Image Seçmek
```dockerfile
FROM ubuntu   # 70MB+ — gereksiz araçlar
FROM alpine   # 5MB   — sadece minimal Linux
```
Daha az araç = daha küçük image + daha küçük saldırı yüzeyi (birisi container'a sızarsa kullanabileceği araç sayısı azalır).

### b) Multi-Stage Build
```dockerfile
# 1. aşama — geçici çalışma alanı
FROM openjdk:17 AS builder
COPY . .
RUN mvn package

# 2. aşama — final image
FROM openjdk:17-jre-slim
COPY --from=builder app.jar .
CMD ["java", "-jar", "app.jar"]
```
`AS builder` ile aşamaya isim verilir, `COPY --from=builder` ile sadece istenen dosya (derlenmiş `app.jar`) final image'a taşınır — dev kit (JDK, kaynak kod, geçici dosyalar) final image'a hiç girmez. İnşaattaki iskele gibi: bina tamamlanınca iskele sökülür, bina kalır.

### c) Layer Caching
Her `RUN`/`COPY`/`ADD` satırı ayrı bir katman oluşturur. Docker her build'de "bu katman değişti mi?" diye bakar — değişmemişse cache'den alır.

**Kural: en az değişen üste, en çok değişen alta.**
```dockerfile
# Doğru sıra
FROM python:3.11-slim
COPY requirements.txt .              # nadiren değişir
RUN pip install -r requirements.txt  # cache'den gelir
COPY . .                             # sık değişir, en sona
```
Sadece kod değiştiğinde, `requirements.txt` ve `pip install` katmanları cache'den gelir — dakikalar kazandırır.

### d) RUN Satırlarını Birleştirme
```dockerfile
# Yanlış — 3 katman, curl hâlâ image'da saklı kalır
RUN apk add curl
RUN curl ... -o app
RUN apk del curl

# Doğru — 1 katman, net sonuç: curl image'a hiç girmez
RUN apk add curl && \
    curl ... -o app && \
    apk del curl
```
Ayrı satırlarda ekleme/silme yapıldığında, önceki katman (curl eklenmiş hali) hâlâ image içinde saklı kalır — tek `RUN` ile birleştirmek gerçek anlamda temiz bir sonuç verir.

---

## 📊 Komut Referansı

| Komut | Açıklama |
|---|---|
| `docker compose down` | Container'ları ve network'ü kaldırır, volume'lara dokunmaz |
| `docker compose down -v` | Volume'ları da silerek kaldırır |
| `docker exec <container> <komut>` | Çalışan bir container içinde komut çalıştırır |
| `FROM <image> AS <isim>` | Multi-stage build'de bir aşamaya isim verir |
| `COPY --from=<isim> <kaynak> <hedef>` | Belirli bir aşamadan dosya kopyalar |

---

## 🧠 Quiz

| # | Soru | Doğru Cevap |
|---|---|---|
| 1 | Windows Server Core tabanlı bir Dockerfile, Linux kernel'i çalıştıran bir sunucuda neden çalıştırılamaz? | Container'lar kendi kernel'ini taşımaz, host'un kernel'ini kullanır — Windows container, Windows kernel'i olmayan bir Linux sunucusunda çalışamaz |
| 2 | `COPY requirements.txt .` satırının `COPY . .`'den önce gelmesi neden tercih edilir? | Docker her katmanı önbelleğe alır; nadiren değişen dosyalar önce, sık değişen kod sonra konularak gereksiz yeniden kurulumlar önlenir |
