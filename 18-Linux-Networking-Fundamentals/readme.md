# Faz 18 — OSI Modeli: Katmanlar, Gerçek Senaryolar, ve Gerçek Paket Doğrulaması

## 📝 Özet

Bu fazda OSI'nin 7 katmanı, teorik tanımlarla değil, gerçek komutlar ve gerçek paket yakalamalarıyla çalışıldı. WSL2/Ubuntu 26.04 ortamında (bkz. 17. Faz notu — WSL, Windows'un arkasında NAT'lı olsa da, giden trafik için gerçek internet yolunu kullanıyor, bu yüzden bu faz için tamamen geçerli sonuçlar üretti) `tcpdump` ile gerçek bir DNS sorgusu paket seviyesinde yakalandı, `traceroute`/`ping` ile 4 farklı gerçek hedef (Cloudflare, Claude.ai, Google, Türkiye Sigorta) arasında ICMP politika farkları karşılaştırıldı, gerçek bir routing tablosu okunup yorumlandı, ve `dig` ile DNS çözümleme zinciri incelendi.

## 1. OSI Modeli — 7 Katman

| Katman | İşi | Bu Fazda Gözlemlendiği Yer |
|---|---|---|
| 7 — Application | HTTP, DNS, SSH gibi kullanıcı protokolleri | `dig`, `ssh` komutlarının kendisi |
| 6 — Presentation | Şifreleme (TLS/SSL), veri formatı | `https://` bağlantılarında TLS; düz DNS'te yok |
| 5 — Session | Bağlantı yaşam döngüsü | SSH oturum yönetimi |
| 4 — Transport | TCP/UDP, port numaraları | DNS'in UDP port 53 kullanması (paket yakalamada doğrulandı) |
| 3 — Network | IP adresleri, yönlendirme | `ip route` çıktısı, traceroute'taki değişmeyen hedef IP |
| 2 — Data Link | MAC adresleri, yerel ağ | Her router hop'unda (kavramsal olarak) yeniden yazılır |
| 1 — Physical | Ham sinyal/bit iletimi | Bu fazda doğrudan gözlemlenmedi |

## 2. Gerçek Paket Yakalama: DNS Sorgusu (`tcpdump`)

### Amaç
`dig google.com` komutunun ürettiği gerçek ağ trafiğini, Layer 3 (IP) ve Layer 4 (UDP/port 53) seviyesinde paket olarak görmek.

### Kullanılan Araçlar
```bash
sudo apt install tcpdump -y
sudo apt install dnsutils -y   # dig komutu için (bu image'da varsayılan kurulu değildi)
```

### Yakalama Komutu (Nihai, Çalışan Hali)
```bash
sudo rm -f /tmp/dns_capture.pcap
sudo timeout 5 tcpdump -i any -n port 53 -w /tmp/dns_capture.pcap &
sleep 2
dig google.com
wait
sudo tcpdump -r /tmp/dns_capture.pcap -n
```

**Komutun açıklaması:**
- `sudo timeout 5 tcpdump -i any -n port 53 -w /tmp/dns_capture.pcap &` → tcpdump'ı arka planda başlatır, en fazla 5 saniye çalışır (kendini otomatik kapatır), tüm ağ arayüzlerini (`any`) dinler, sadece port 53 (DNS) trafiğini filtreler, sonucu bir dosyaya yazar (`-w`).
- `sleep 2` → tcpdump'ın dinlemeye başlaması için 2 saniye bekler.
- `dig google.com` → gerçek DNS trafiği üretir (bu, yakalanacak paketi oluşturan komut).
- `wait` → arka plandaki tcpdump işinin (timeout ile) kendiliğinden bitmesini bekler.
- `sudo tcpdump -r /tmp/dns_capture.pcap -n` → kaydedilen paketleri okunabilir formatta ekrana basar (`-r` = read from file, `-n` = IP adreslerini isimlere çevirme, ham haliyle göster).

### Yakalanan Paketler
```
00:14:59.098415 lo In  IP 10.255.255.254.55173 > 10.255.255.254.53: 4112+ [1au] A? google.com. (51)
00:14:59.177907 lo In  IP 10.255.255.254.53 > 10.255.255.254.55173: 4112 1/0/1 A 142.250.180.14 (55)
```

**Paket analizi:**
- 1. paket: DNS **sorgusu** — kaynak port `55173` (rastgele/geçici port), hedef port `53` (DNS standart portu), tip `A?` (A kaydı sorgusu — IPv4 adresi istiyor).
- 2. paket: DNS **cevabı** — kaynak/hedef portlar ters dönmüş (cevap sorgunun tam tersi yönde), sonuç `A 142.250.180.14` (google.com'un IPv4 adresi).

### 🔍 WSL'e Özgü Bir Gözlem
Paketler `eth0` yerine `lo` (loopback) arayüzünden geçti. Sebep: WSL2, kendi dahili DNS proxy'sini (`10.255.255.254`) yerel olarak çalıştırıyor — DNS isteği, gerçek dış ağa çıkmadan önce bu yerel proxy'ye gidiyor. Gerçek bir bulut sunucusunda veya fiziksel bir Linux makinesinde bu paketler muhtemelen `eth0` üzerinden görünecekti; bu WSL2'nin ağ mimarisine özgü bir davranış olarak not edildi.

## 3. Encapsulation / Decapsulation (Kavramsal)

Veri, gönderilirken Layer 7'den Layer 1'e inerken her katman kendi header'ını ekler:

```
[Layer 2: MAC header]
  [Layer 3: IP header]
    [Layer 4: UDP header — port 53 bilgisini içerir]
      [Layer 7: DNS sorgusu verisi]
```

Yukarıdaki paket yakalamasında bu görüldü: `tcpdump` çıktısı IP adresleri (Layer 3) ve port numaralarını (Layer 4) aynı satırda gösteriyor — bunlar, DNS verisini (Layer 7) saran header'lardan geliyor.

## 4. Routing Tablosu Okuma (`ip route`)

### Komut ve Çıktı
```bash
ip route
```
```
default via 172.22.224.1 dev eth0 proto kernel
172.17.0.0/16 dev docker0 proto kernel scope link src 172.17.0.1 linkdown
172.22.224.0/20 dev eth0 proto kernel scope link src 172.22.227.220
```

### Satır Satır Yorumlama
| Satır | Anlamı |
|---|---|
| `default via 172.22.224.1 dev eth0` | Yedek kural: başka hiçbir kural eşleşmezse, paket bu gateway'e (WSL'in sanal NAT gateway'i) gönderilir. |
| `172.17.0.0/16 dev docker0 ... linkdown` | Docker'ın kendi sanal ağı. `linkdown`, o an aktif çalışan hiçbir container olmadığını gösteriyor. |
| `172.22.224.0/20 dev eth0 ... src 172.22.227.220` | Bu makinenin kendi yerel ağ aralığı — bu aralıktaki başka bir adrese, gateway'e gerek kalmadan doğrudan ulaşılabilir. |

### Docker Ağının Doğrulanması
```bash
docker ps -a
# Çıktı: "permission denied while trying to connect to the Docker API at unix:///var/run/docker.sock"
```
**Not:** `egeadmin` kullanıcısı `docker` grubunda olmadığı için (17. fazda Docker'ı `sudo` ile çalıştırmıştık, kullanıcıyı `docker` grubuna eklememiştik) bu komut izin hatası verdi. `sudo docker ps -a` ile çalıştırılabilirdi; bu fazda zaman kısıtı nedeniyle `ip route`'daki `linkdown` durumu tek başına yeterli kanıt olarak kabul edildi (aktif container olmadığını zaten gösteriyor).

### IP Forwarding Kontrolü
```bash
cat /proc/sys/net/ipv4/ip_forward
# Çıktı: 1
```
**Yorum:** `1` (açık) döndü. Normalde düz bir sunucuda `0` (kapalı) beklenir, çünkü başkasının trafiğini yönlendirmesi için sebep yoktur. Burada `1` olmasının sebebi **Docker** — Docker, container'ların dış ağa çıkabilmesi için host'un IP forwarding yapmasına ihtiyaç duyar ve kurulumu sırasında bu ayarı otomatik olarak açar. Bu, rastgele bir varsayılan değil, Docker'ın kurulu olmasının doğrudan bir sonucu.

## 5. ICMP ve Traceroute — Gerçek Sağlayıcı Karşılaştırması

### Test Yöntemi
Aynı WSL/Ubuntu ortamından, 4 farklı gerçek hedefe `traceroute` ve `ping` çalıştırılarak, ICMP politikalarındaki farklar doğrudan gözlemlendi.

### Sonuç Tablosu

| Hedef | `traceroute` Sonucu | `ping` Sonucu |
|---|---|---|
| **1.1.1.1** (Cloudflare) | ✅ 11 hop'ta tam tamamlandı, `one.one.one.one` olarak doğrulandı | (ayrıca test edilmedi — traceroute zaten ulaşılabilirliği kanıtladı) |
| **claude.ai** | ✅ 11 hop'ta tam tamamlandı, ilk 9 hop Cloudflare testiyle neredeyse birebir aynı (aynı ISP çıkışından geçtiği için beklenen) | (aynı) |
| **google.com** | ⚠️ Kaynak metinden farklı: 15 hop sınırında hedefe tam varmadı, ama 30 hop'a çıkarınca Google'ın kendi ağına (`142.251.x`, `108.170.x`, `216.239.x`) kadar ulaştı — bazı ara hop'lar `* * *` gösterse de tamamen susturulmadı | ✅ %0 paket kaybı, 4/4 paket başarılı (59-68ms) |
| **turkiyesigorta.com.tr** | ❌ 20 hop'a kadar tamamen sessiz kaldı (Türk Telekom altyapısındaki bazı ara hop'lar göründü ama hedefin kendisine hiç ulaşılamadı) | ❌ %100 paket kaybı, 4/4 paket başarısız |

### Komutlar
```bash
traceroute -m 15 1.1.1.1
traceroute -m 15 claude.ai
traceroute -m 30 google.com
ping -c 4 google.com
traceroute -m 20 turkiyesigorta.com.tr
ping -c 4 turkiyesigorta.com.tr
```

### Yorum — Kurumların ICMP'yi Farklı Ele Alması
- **Cloudflare & Claude.ai (Cloudflare altyapısı):** tamamen açık — şeffaflık, bir altyapı sağlayıcısı için değer önerisinin parçası.
- **Google:** `ping`'e tam izin veriyor, `traceroute`'u ise kısmen kısıtlıyor (bazı hop'larda `* * *`) ama tamamen engellemiyor — kaynak metinde "tamamen engelliyor" denmiş olsa da, bu testte kısmi bir kısıtlama gözlemlendi. Bu fark muhtemelen zamanla değişen bir politika ya da farklı ağ yolundan kaynaklanıyor olabilir.
- **Türkiye Sigorta:** ICMP'yi tamamen kapatmış — hem `ping` hem `traceroute` sıfır sonuç verdi. Finans/sigorta sektöründe yaygın olan deny-by-default güvenlik yaklaşımıyla tutarlı.

Bu, önceki fazlarda (SSH sertleştirme, sudoers) öğrenilen **En Düşük Yetki Prensibi**nin ağ seviyesine uygulanmış hali: sadece gerçekten gerekli olan açık bırakılıyor.

## 6. DNS Çözümleme

### Denenen: Tam Resolver Zinciri (`dig +trace`)
```bash
dig +trace google.com
```
**Sonuç:** Başarısız — `;; communications error to 10.255.255.254#53: timed out` / `no servers could be reached`.

**🐛 Sebep:** WSL2'nin dahili DNS proxy'si (`10.255.255.254`), root nameserver'lara doğrudan sorgu yapma (`+trace` modu) için tasarlanmamış — sadece normal, tek adımlı DNS sorgularını (kendi üst zincirine) yönlendiriyor. Bu, WSL2'nin ağ mimarisine özgü bir kısıtlama.

### Çalışan Alternatif: Normal `dig`
```bash
dig google.com
```
```
;; ANSWER SECTION:
google.com.    160    IN    A    192.178.24.142

;; Query time: 1103 msec
;; SERVER: 10.255.255.254#53(10.255.255.254) (UDP)
```

**Yorum:** DNS sorgusu, WSL'in yerel proxy'si üzerinden başarıyla çözümlendi. TTL değeri 160 saniye — bu kaydın 160 saniye boyunca cache'lenebileceğini gösteriyor. Query time (1103ms), WSL'in ekstra proxy katmanı yüzünden gerçek bir Linux sunucuya göre daha yavaş.

---

## 🐛 Hata & Çözüm

### Hata 1: `dig` komutu bulunamadı
**Hata mesajı:** `Command 'dig' not found, but can be installed with: sudo apt install bind9-dnsutils`
**Neden:** Bu WSL/Ubuntu image'ı gerçekten minimal — ağ araçları önceden kurulu gelmiyor.
**Çözüm:** `sudo apt install dnsutils -y`

### Hata 2: Arka plandaki `tcpdump` için "Permission denied"
**Hata mesajı:** `tcpdump: /tmp/dns_capture.pcap: Permission denied`
**Neden:** `sudo` ile başlatılan bir komut `&` ile arka plana atıldığında, `sudo` şifre istemi göstermek için interaktif bir terminale erişemiyor, bu yüzden yetkilendirme sessizce başarısız oluyor.
**Çözüm:** Arka plana almadan önce `sudo -v` ile sudo yetkisini "önceden onaylayıp" kısa süreliğine cache'e almak.

### Hata 3: Root tarafından oluşturulmuş dosyayı normal kullanıcı silemiyor
**Hata mesajı:** `rm: cannot remove '/tmp/dns_capture.pcap': Operation not permitted`
**Neden:** Bir önceki başarısız denemede dosya kısmen `root` yetkisiyle oluşturulmuş, normal kullanıcı (`egeadmin`) onu silemiyor.
**Çözüm:** `sudo rm -f ...` ile silme işlemini de yetkili şekilde yapmak.

### Hata 4: `tcpdump -i eth0` ile 0 paket yakalandı
**Neden:** DNS trafiği, WSL2'nin dahili DNS proxy'sine (`10.255.255.254`) gidiyor — bu trafik `eth0` üzerinden değil, `lo` (loopback) arayüzü üzerinden geçiyor.
**Çözüm:** `-i eth0` yerine `-i any` (tüm arayüzleri dinle) kullanmak.

### Hata 5: Yazım hatası — `car` yerine `cat`
**Hata mesajı:** `Command 'car' not found, but can be installed with: sudo apt install ucommon-utils`
**Neden:** Basit bir yazım hatası (`cat` yerine `car` yazıldı).
**Çözüm:** Doğru komutla (`cat /proc/sys/net/ipv4/ip_forward`) tekrar çalıştırıldı.

### Hata 6: `dig +trace` başarısız oldu
**Hata mesajı:** `;; communications error to 10.255.255.254#53: timed out` / `no servers could be reached`
**Neden:** WSL2'nin dahili DNS proxy'si, root nameserver'lara doğrudan `+trace` sorgusu yapılmasını desteklemiyor.
**Çözüm:** Tam kesintisiz bir çözüm bulunamadı (WSL2'nin mimari kısıtlaması); bunun yerine normal `dig google.com` ile DNS çözümlemesinin çalıştığı doğrulandı, `+trace` kısıtlaması journal'a not olarak düşüldü.

---

## 📊 Komut Referansı

| Komut | Açıklama |
|---|---|
| `sudo apt install tcpdump dnsutils traceroute -y` | Bu fazda kullanılan ağ analiz araçlarının kurulumu |
| `sudo timeout N tcpdump -i any -n port PORT -w dosya.pcap` | Belirli bir portu, N saniye boyunca, tüm arayüzlerden dinleyip dosyaya kaydeder |
| `sudo tcpdump -r dosya.pcap -n` | Kaydedilmiş bir pcap dosyasını okunabilir formatta gösterir |
| `dig <domain>` | Bir domain adını IP adresine çözümler |
| `dig +trace <domain>` | Resolver zincirinin (root → TLD → authoritative) her adımını gösterir (WSL2'de çalışmadı) |
| `traceroute -m N <hedef>` | Hedefe giden yolu, en fazla N hop ile gösterir |
| `ping -c N <hedef>` | Hedefe N adet ICMP Echo Request gönderir |
| `ip route` | Sistemin routing tablosunu gösterir |
| `cat /proc/sys/net/ipv4/ip_forward` | IP forwarding'in açık (1) mı kapalı (0) mı olduğunu gösterir |
| `docker ps -a` | Çalışan/durmuş tüm Docker container'larını listeler |

---

## 🧠 Quiz

| # | Soru | Doğru Cevap |
|---|---|---|
| 1 | `tcpdump -i eth0` ile DNS paketleri yakalanamazken `-i any` ile yakalanabildi. Bu WSL2'ye özgü davranışın sebebi nedir? | WSL2'nin dahili DNS proxy'si (`10.255.255.254`) trafiği `eth0` değil `lo` (loopback) arayüzü üzerinden yönlendiriyor |
| 2 | `cat /proc/sys/net/ipv4/ip_forward` komutunun `1` (açık) döndürmesinin, düz bir sunucu için beklenmedik ama bu ortamda mantıklı olmasının sebebi nedir? | Docker, container'ların dış ağa çıkabilmesi için host'un IP forwarding yapmasına ihtiyaç duyar ve kurulumu sırasında bu ayarı otomatik açar |

---

## 📌 Sonraki Adımlar / İncelenmemiş Konular
- `dig +trace`'in WSL2'de neden çalışmadığına dair daha derin bir araştırma (gerçek bir Linux sunucuda veya Rocky VM'de tekrar denenebilir).
- Docker container ağının (`docker0`) canlı bir container ile (`sudo docker run -it ubuntu bash` gibi) test edilmesi.
- DNSSEC imzalarının (`DS`, `RRSIG` kayıtları) daha ileri seviye incelenmesi.
