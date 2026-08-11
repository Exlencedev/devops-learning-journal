# 🧠 Quiz — Gerçek Sonuçlar

Bu belge, fazlar ilerledikçe gerçekten sorulup cevaplanan quiz sorularını içeriyor. Her soru, o fazı tamamladıktan hemen sonra sorulmuş ve gerçek cevaplarla değerlendirilmiştir — repo tamamlanana kadar her yeni fazda bu dosyaya 5'li setler halinde ekleme yapılacaktır.

---

## 📊 Genel Performans Özeti

| Set | Kapsadığı Fazlar | Soru Sayısı | Doğru | Başarı Oranı |
|-----|-------------------|:-----------:|:-----:|:-------------:|
| **Set 1** | 04-06 | 5 | 5 | %100 |
| **Set 2** | 06-08 | 5 | 5 | %100 |
| **Set 3** | 09-10 | 4 | 4 | %100 |
| **TOPLAM** | 04-10 | **14** | **14** | **%100** |

---

## Set 1 (Faz 04-06)

#### S1: `su - devopstester` ile `sudo su - devopstester` arasındaki fark neydi?

- **Cevap:** `su`, hedef kullanıcının şifresini ister; `sudo su` ise kendi sudo şifreni kullanarak geçiş yapmanı sağlar.
- **Sonuç:** ✅ Doğru
- **Faz:** [04-User-Privilege-Management](./04-User-Privilege-Management/readme.md)

#### S2: `devopstester`, `dnf update` komutunu denediğinde neden yine de şifre soruldu (sonra reddedildi)?

- **Cevap:** `NOPASSWD` kuralına dahil olmayan komutlar için sudo hâlâ kimlik doğrulaması ister, sonra yetki kontrolü yapar ve reddeder.
- **Sonuç:** ✅ Doğru
- **Faz:** [04-User-Privilege-Management](./04-User-Privilege-Management/readme.md)

#### S3: Dizinler için `x` (execute) biti neden önemli, dosyalardan farklı olarak ne işe yarar?

- **Cevap:** Dizinde execute (x) biti, içine girme/geçme (cd) iznini kontrol eder — `r` olmadan bile içini listeleyemezsin, `x` olmadan içine giremezsin.
- **Sonuç:** ✅ Doğru
- **Faz:** [05-Linux-Permissions](./05-Linux-Permissions/readme.md)

#### S4: Sticky bit tam olarak neyi engeller?

- **Cevap:** Sticky bit, dizindeki dosyaları sadece sahibinin (veya root'un) silebilmesini sağlar, yazma izni olan herkesin değil.
- **Sonuç:** ✅ Doğru
- **Faz:** [05-Linux-Permissions](./05-Linux-Permissions/readme.md)

#### S5: `kill -15` (SIGTERM) ile `kill -9` (SIGKILL) arasındaki fark nedir?

- **Cevap:** SIGTERM (15), sürece "lütfen kendini düzgünce kapat" der, süreç temizlik yapabilir; SIGKILL (9) süreci anında ve zorla sonlandırır, temizlik yapamaz.
- **Sonuç:** ✅ Doğru
- **Faz:** [06-Linux-Process-Management](./06-Linux-Process-Management/readme.md)

---

## Set 2 (Faz 06-08)

#### S6: VM'in donmasına (soft lockup) neyin sebep olduğunu düşünüyorsun?

- **Cevap:** Çok yüksek CPU yüklü bir süreci uzun süre çalıştırmak, kernel'in "soft lockup" uyarısı vermesine ve sistemin geçici olarak tepkisiz kalmasına yol açabilir.
- **Sonuç:** ✅ Doğru
- **Faz:** [06-Linux-Process-Management](./06-Linux-Process-Management/readme.md)

#### S7: `systemctl start` ile `systemctl enable` arasındaki fark nedir?

- **Cevap:** `start` sadece şu an çalıştırır; `enable` ise servisin her sistem açılışında otomatik başlamasını sağlar — ikisi birbirinden bağımsız ayarlar.
- **Sonuç:** ✅ Doğru
- **Faz:** [07-Linux-Service-Management](./07-Linux-Service-Management/readme.md)

#### S8: Log çıktısında `reload` ile `restart` arasındaki farkı nasıl gözlemledin?

- **Cevap:** `reload`, aktif bağlantıları kesmeden sadece yapılandırmayı yeniler (sıfır kesinti); `restart` servisi tamamen durdurup başlatır (kısa kesinti olur).
- **Sonuç:** ✅ Doğru
- **Faz:** [07-Linux-Service-Management](./07-Linux-Service-Management/readme.md)

#### S9: `sed` varsayılan olarak büyük/küçük harfe duyarlı mı? `I` flag'i ne işe yaradı?

- **Cevap:** `sed` varsayılan olarak büyük/küçük harf duyarlıdır; `I` flag'i eklenirse duyarsız hale gelir, ikisini de eşleştirir.
- **Sonuç:** ✅ Doğru
- **Faz:** [08-Linux-Log-Analysis](./08-Linux-Log-Analysis/readme.md)

#### S10: `sed -i` ile `-i` olmadan `sed` arasındaki fark nedir?

- **Cevap:** `-i` olmadan `sed` sadece ekrana basar, dosya değişmeden kalır; `-i` ile değişiklik dosyaya kalıcı olarak yazılır.
- **Sonuç:** ✅ Doğru
- **Faz:** [08-Linux-Log-Analysis](./08-Linux-Log-Analysis/readme.md)

---

## Set 3 (Faz 09-10)

#### S11: DNS sorun giderirken `/etc/resolv.conf`'a bakmanın önemi neydi, senin durumunda ne öğrendin?

- **Cevap:** `/etc/resolv.conf`, sistemin gerçekten kullandığı nameserver'ları gösterir — varsayımda bulunmak yerine gerçek yapılandırmayı kontrol etmek gerekir.
- **Sonuç:** ✅ Doğru
- **Faz:** [09-Linux-Network-Management](./09-Linux-Network-Management/readme.md)

#### S12: `ss -lntp` çıktısında 80 portunda neden birden fazla nginx PID'si göründü?

- **Cevap:** `ss -lntp`'nin çıktısında aynı porta bağlı birden fazla nginx PID'si göründü — nginx'in master process + worker process'ler mimarisi yüzünden.
- **Sonuç:** ✅ Doğru
- **Faz:** [09-Linux-Network-Management](./09-Linux-Network-Management/readme.md)

#### S13: `mount -a` komutunu çalıştırmanın güvenlik açısından önemi neydi?

- **Cevap:** fstab'daki bir hata, sistemi yeniden başlatana kadar fark edilmeyebilir ve emergency mode'a düşürebilir; `mount -a` bunu yeniden başlatmadan, şimdi test etmeyi sağlar.
- **Sonuç:** ✅ Doğru
- **Faz:** [10-Linux-Storage-Management](./10-Linux-Storage-Management/readme.md)

#### S14: fstab'da cihaz yolu (`/dev/loop0p1`) yerine neden UUID kullanılır?

- **Cevap:** `/dev/loop0p1` gibi cihaz yolları yeniden başlatmalar arasında değişebilir; UUID ise bölümün değişmeyen, benzersiz kimliğidir.
- **Sonuç:** ✅ Doğru
- **Faz:** [10-Linux-Storage-Management](./10-Linux-Storage-Management/readme.md)

---

ℹ️ _Bu dosya, repo tamamlandıkça 5'li setler halinde güncellenmeye devam edecektir._
