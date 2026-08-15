# 🚀 DevOps & Linux Öğrenme Yolculuğu

Bu repo, Linux ve DevOps temellerini öğrenirken tuttuğum günlük notları belgeleyen bir öğrenme günlüğüdür. [alifurkan-altuntas/devops-internship](https://github.com/alifurkan-altuntas/devops-internship) reposundaki müfredatı takip ederek, VirtualBox üzerinde kurduğum Rocky Linux VM'inde adım adım ilerliyorum.

## 📍 Şu An Neredeyim

Forward Proxy / Reverse Proxy fazında, Nginx'in reverse proxy olarak nasıl çalıştığını kavramsal olarak öğrendim ve ilk hands-on denemede gerçek bir 502 Bad Gateway hatasıyla karşılaştım. Backend (Python HTTP sunucusu) ile Nginx'i farklı VM'lerde çalıştırıp `proxy_pass`'te `localhost` kullanınca, `localhost`'un her zaman "bu makine" anlamına geldiğini ve Nginx'in kendi üzerinde olmayan bir backend'i arayamayacağını bizzat deneyimledim. Bu kavramsal temel üzerine, path bazlı yönlendirme ve gerçek sunucu kurulumu sonraki fazda (19-Nginx-Derinlestirme) tamamlanacak.

Bundan önce, Görev Zamanlama fazında `crontab`'ın 5 yıldızlı zamanlama mantığını, `crontab -e` ile periyodik görev tanımlamayı, çıktıları log dosyalarına (`>>` ve `2>&1`) yönlendirmeyi ve modern Linux sistemlerinde `systemd timers` ile zaman tabanlı servis tetiklemeyi öğrendim.

SSH Yönetimi fazında, VirtualBox'ın NAT ağ modunda host-VM arası doğrudan IP erişimi olmadığını keşfedip Port Forwarding ile çözdüm, key-based authentication kurdum, `sshd_config`'i sertleştirdim ve fail2ban ile brute-force koruması aktifleştirdim.

Sırada kapsamlı Bash Scripting ve CI/CD / Konteynerizasyon fazları var.

---

## 📁 Repo Yapısı

- [00-VM-Setup](./00-VM-Setup/): VirtualBox kurulumu ve Rocky Linux 10.2'nin ISO'dan manuel kurulumu.
- [01-Linux-Basics](./01-Linux-Basics/): Sistem kimliği komutları (`hostname`, `hostnamectl`) ve pipe tabanlı metin işleme (`grep`, `cut`, `awk`, `tr`).
- [02-Vagrant-Automation](./02-Vagrant-Automation/): Windows host'a Vagrant kurulumu, VirtualBox provider ile otomatik VM oluşturma (`vagrant up`, `vagrant ssh`).
- [03-File-System-Management](./03-File-System-Management/): `dd` ile test dosyası oluşturma, XFS speculative preallocation keşfi, ve en büyük dosyaları bulma pipeline'ı.
- [04-User-Privilege-Management](./04-User-Privilege-Management/): `visudo` ile En Düşük Yetki Prensibi — kısıtlı bir kullanıcıya sadece nginx komutlarına özel, şifresiz sudo yetkisi tanımlama.
- [05-Linux-Permissions](./05-Linux-Permissions/): `chmod` sayısal sistemi, sticky bit ile paylaşımlı dizin koruması, `chown`/`chgrp`, `umask` matematiği.
- [06-Linux-Process-Management](./06-Linux-Process-Management/): `top` ile işlem izleme, gerçek bir kernel "soft lockup" krizi ve çözümü, `kill` sinyalleri, `nice`/`renice` ile önceliklendirme.
- [07-Linux-Service-Management](./07-Linux-Service-Management/): `systemd` ile servis yönetimi (`enable`/`start`/`reload`/`restart`), `journalctl` ile log filtreleme.
- [08-Linux-Log-Analysis](./08-Linux-Log-Analysis/): nginx access log analizi (`awk`/`grep`/`sort`/`uniq` pipeline'ları), `sed` ile bul-değiştir ve satır silme.
- [09-Linux-Network-Management](./09-Linux-Network-Management/): `dig` ile DNS sorgulama, `ss` ile dinleme portları, `openssl` ile TLS sertifika doğrulama.
- [10-Linux-Storage-Management](./10-Linux-Storage-Management/): loop device ile sanal disk oluşturma, `fdisk` ile bölümleme, `mkfs.ext4`, `/etc/fstab` ile UUID tabanlı kalıcı mount.
- [11-Linux-LVM-Management](./11-Linux-LVM-Management/): PV/VG/LV katmanları, `fallocate` ile güvenli test diski, `vgextend`+`lvextend`+`resize2fs` ile kesintisiz depolama genişletme.
- [12-Linux-Task-Scheduling](./12-Linux-Task-Scheduling/): `crontab` sözdizimi, periyodik arka plan otomasyonu, standart çıktı/hata yönlendirmesi (`>>` ve `2>&1`), `systemd timer` ile modern zamanlama.
- [12-Linux-SSH-Management](./12-Linux-SSH-Management/): SSH servis yönetimi, VirtualBox NAT port forwarding, key-based authentication, `sshd_config` sertleştirme, firewalld/SELinux doğrulaması, fail2ban ile brute-force koruması.
- [13-Forward-Reverse-Proxy](./13-Forward-Reverse-Proxy/): Forward proxy ve reverse proxy kavramları, Nginx `location`/`proxy_pass` direktifleri, ilk reverse proxy denemesinde alınan 502 Bad Gateway hatası ve kök nedeni.

---

## 📅 Günlük İlerleme Kayıtları

### 🔹 Gün 1 | VirtualBox Kurulumu & Rocky Linux Kurulumu

_VirtualBox'ı ilk defa kullanıyordum. Rocky Linux ISO indirirken farkında olmadan sürüm 10'u indirdim (müfredat 9.8 kullanıyor), pratik fark olmadığı için 10 ile devam ettim._

- **Görevler & Hedefler:**
  - VirtualBox 7.2.14 kuruldu.
  - Rocky Linux 10.2 Minimal ISO indirildi ve VM'e bağlandı (4096 MB RAM, 4 vCPU, 20 GB disk).
  - Anaconda installer üzerinden kurulum tamamlandı (kurulum hedefi, root şifresi, kullanıcı oluşturma).
  - `sudo dnf update -y` ile sistem güncellendi.
- **Kilometre Taşları & Çıktılar:**
  - 🛠️ VM Kurulum Süreci: [00-VM-Setup](./00-VM-Setup/readme.md)

### 🔹 Gün 2 | Linux Temelleri & Dosya Sistemi Yönetimi

_Pipe (`|`) karakterini Türkçe klavyede yazmakta zorlandım (AltGr + < gerekiyor), ve `dd` ile oluşturduğum 2 GB'lık dosyanın `du` çıktısında 3.7 GB görünmesi beni şaşırttı — araştırınca XFS'in performans için fazladan blok ayırdığını (speculative preallocation) öğrendim._

- **Görevler & Hedefler:**
  - `hostname`, `hostnamectl`, `uname -a` ile sistem kimliği bilgisi toplandı.
  - `grep`/`cut`/`tr` pipeline'ı ile `/etc/os-release`'den dağıtım adı ayıklandı.
  - `awk` ile `df -h` çıktısından özel formatlı disk raporu üretildi.
  - `dd` ile 2 GB'lık test dosyası oluşturuldu, `stat` ile XFS blok ayırma davranışı doğrulandı.
  - `find`/`du`/`sort`/`head` pipeline'ı ile sistemdeki en büyük 10 dosya listelendi.
- **Kilometre Taşları & Çıktılar:**
  - 📜 Linux Temelleri Notları: [01-Linux-Basics](./01-Linux-Basics/readme.md)
  - 📁 Dosya Sistemi Notları: [03-File-System-Management](./03-File-System-Management/readme.md)

### 🔹 Gün 3 | Kullanıcı & Yetki Yönetimi

_`visudo` içinde vi editörüyle yazarken iki yazım hatası yaptım (`/usr/bsn/` ve `ngingx`), bunu gözle değil `sudo -l -U devopstester` doğrulama komutuyla tespit ettim. Ayrıca `su` ile `sudo su` arasındaki şifre farkını canlı olarak deneyimledim._

- **Görevler & Hedefler:**
  - nginx kuruldu ve servis durumu doğrulandı.
  - Kısıtlı bir kullanıcı (`devopstester`) oluşturuldu.
  - `visudo` ile sadece nginx komutlarına özel, şifresiz (`NOPASSWD`) sudo yetkisi tanımlandı.
  - İzinli komut (`restart nginx`) ve izinsiz komut (`dnf update`) ile En Düşük Yetki Prensibi test edildi.
- **Kilometre Taşları & Çıktılar:**
  - 🔐 Yetki Yönetimi Notları: [04-User-Privilege-Management](./04-User-Privilege-Management/readme.md)

### 🔹 Gün 4 | Vagrant Otomasyonu & Linux İzinleri

_Vagrant'ı önce yanlışlıkla VM'in içine kurmaya çalıştım, sonra bunun host makinede (Windows) olması gerektiğini fark ettim — VirtualBox'ı dışarıdan yönetmesi gerektiği için mantıklı geldi. İzinler fazında ise sticky bit'i gerçek iki kullanıcıyla (ege ve devopstester) test edip, `devopstester`'ın başkasının dosyasını gerçekten silemediğini kanıtladım._

- **Görevler & Hedefler:**
  - Windows host'a Vagrant 2.4.9 kuruldu, VirtualBox provider ile `generic/rocky9` box'ı ayağa kaldırıldı.
  - `chmod` sayısal sistemi (750) test edildi.
  - `/tmp/test` üzerinde sticky bit uygulanıp, 2 farklı kullanıcıyla gerçek silme testi yapıldı.
  - `chown`/`chgrp` ile sahiplik ve grup ayrı ayrı değiştirildi.
  - `umask` matematiği hem varsayılan (022) hem sıkılaştırılmış (077) değerle doğrulandı.
- **Kilometre Taşları & Çıktılar:**
  - 🤖 Vagrant Notları: [02-Vagrant-Automation](./02-Vagrant-Automation/readme.md)
  - 🔑 İzinler Notları: [05-Linux-Permissions](./05-Linux-Permissions/readme.md)

### 🔹 Gün 5 | İşlem Yönetimi (Kriz Anı Dahil)

_`dd if=/dev/zero of=/dev/null &` ile başlattığım CPU testini `top` ile uzun süre izlerken VM tamamen tepkisiz kaldı — ekranda `kernel: watchdog: BUG: soft lockup - CPU#3 stuck for 30s!` uyarısı belirdi. VM'i Power Off ile kurtarıp, testi bir daha bu sefer saniyeler içinde başlatıp-sonlandırarak kontrollü şekilde tekrarladım._

- **Görevler & Hedefler:**
  - `top` ile gerçek zamanlı CPU/işlem izleme yapıldı.
  - Gerçek bir kernel soft lockup krizi yaşandı ve VM güvenli şekilde kurtarıldı.
  - `pidof` ile PID alma, `kill -9` ile hızlı ve kontrollü sonlandırma yapıldı.
  - `nice -n 19` ile düşük öncelikli işlem başlatıldı, `renice` ile çalışırken önceliği canlı değiştirildi (19 → 5).
- **Kilometre Taşları & Çıktılar:**
  - ⚙️ İşlem Yönetimi Notları: [06-Linux-Process-Management](./06-Linux-Process-Management/readme.md)

### 🔹 Gün 6 | Servis & Log Yönetimi

_`systemctl enable nginx` yazarken yine bir yazım hatası yaptım (`nignx`), hata mesajının netliği sayesinde hemen fark edip düzelttim. `reload` ile `restart` arasındaki teorik farkı, `journalctl` log çıktısında (Reloading/Reloaded vs Stopping/Stopped/Starting/Started) somut olarak gördüm._

- **Görevler & Hedefler:**
  - nginx servisi `enable` ile kalıcı hale getirildi, `start` ile çalıştırıldı.
  - `reload` ve `restart` komutları test edildi.
  - `journalctl -u`, `-p err`, `--since` filtreleriyle log analizi yapıldı.
- **Kilometre Taşları & Çıktılar:**
  - 🏗️ Servis Yönetimi Notları: [07-Linux-Service-Management](./07-Linux-Service-Management/readme.md)

### 🔹 Gün 7 | Log Analizi & `sed`

_`awk '{print $7}'` yazarken `pring` yazım hatası yaptım, çıktı boş geldiği için hemen fark ettim. `printf` ile `\n` kullanarak tek satırda test dosyası oluşturmaya çalıştım ama Türkçe klavyede `\` karakteri doğru basılamadı — bunun yerine her satırı ayrı `echo` komutuyla ekleyerek çözdüm._

- **Görevler & Hedefler:**
  - `curl -I` ile test HTTP istekleri oluşturuldu (200 ve 404), nginx access log formatı okundu.
  - `awk`/`sort`/`uniq -c` ile en çok istek gönderen IP bulundu.
  - `grep`/`awk` pipeline'ıyla path bazlı 404 hataları sayıldı.
  - `sed` ile bul-değiştir, büyük/küçük harf duyarlılığı (`I` flag'i), `-i` ile kalıcı değişiklik, `sed 'Nd'` ile satır silme test edildi.
- **Kilometre Taşları & Çıktılar:**
  - 📜 Log Analizi Notları: [08-Linux-Log-Analysis](./08-Linux-Log-Analysis/readme.md)

### 🔹 Gün 8 | Ağ Yönetimi (DNS, Portlar, TLS)

_`dig` ve `openssl` komutlarının ikisi de "komut yok" hatası verdi — Rocky Linux minimal kurulumun gerçekten minimal olduğunu, ağ araçlarının ayrıca kurulması gerektiğini öğrendim (`bind-utils`, `openssl` paketleri). Ayrıca `dig google.com` ile `dig @8.8.8.8 google.com`'un aynı sonucu verdiğini görünce `/etc/resolv.conf`'a bakıp sistemin zaten varsayılan olarak 8.8.8.8 kullandığını keşfettim._

- **Görevler & Hedefler:**
  - `bind-utils` kurulup `dig` ile DNS sorgulama yapıldı, `/etc/resolv.conf` incelendi.
  - `ss -lntp` ile nginx'in 80 portunda IPv4+IPv6 dinlediği doğrulandı (master+worker PID'leri gözlemlendi).
  - `openssl` kurulup `s_client` ile TLS sertifika doğrulaması yapıldı (`Verify return code: 0 (ok)`, `TLSv1.3`).
- **Kilometre Taşları & Çıktılar:**
  - 🌐 Ağ Yönetimi Notları: [09-Linux-Network-Management](./09-Linux-Network-Management/readme.md)

### 🔹 Gün 9 | Depolama Yönetimi (En Riskli Faz)

_Bu fazda 3 gerçek hatayla karşılaştım: `losetup -fP` yerine `-fp` yazdım, mount komutunda bir race condition yaşadım (mkdir ve mount çok hızlı ardışık çalışınca), ve en önemlisi `/etc/fstab`'a satır eklerken tırnak kullanımı yüzünden satır ikiye bölündü. Önceden aldığım yedek (`fstab.backup`) sayesinde saniyeler içinde güvenli şekilde geri dönebildim — kritik sistem dosyalarında yedek almanın neden önemli olduğunu bizzat deneyimledim._

- **Görevler & Hedefler:**
  - `dd` + `losetup -fP` ile 1GB'lık loop device oluşturuldu.
  - `fdisk` ile etkileşimli bölümleme yapıldı (`n` → Enter×3 → `w`).
  - `mkfs.ext4` ile biçimlendirme yapıldı, UUID alındı.
  - `/etc/fstab`'a UUID ile kalıcı mount girişi eklendi (2 hata debug edilerek).
  - `mount -a` ile sistem yeniden başlatılmadan güvenli test yapıldı.
- **Kilometre Taşları & Çıktılar:**
  - 💾 Depolama Yönetimi Notları: [10-Linux-Storage-Management](./10-Linux-Storage-Management/readme.md)

### 🔹 Gün 10 | LVM (Mantıksal Hacim Yönetimi)

_Orijinal müfredatta `dd` ile 50GB'lık dosya oluşturulurken host diskinin dolup VM'in donduğu bir olay yaşanmıştı — bu dersi baştan uygulayıp `fallocate` ve küçük MB boyutlarıyla ilerledim. `mkfs.ext4`'te `test_Data` (büyük D) yazıp "dosya yok" hatası aldım, `vgextend`'de de `test_poo` yazım hatası yaptım — ikisini de hızlıca debug ettim. En etkileyici kısım, bir hacmi hiç umount etmeden canlı olarak 455M'dan 638M'a büyütebilmekti._

- **Görevler & Hedefler:**
  - `fallocate` ile 2 güvenli test diski oluşturuldu, `pvcreate`/`vgcreate`/`lvcreate` ile PV→VG→LV katmanları kuruldu.
  - `mkfs.ext4` ve `mount` ile hacim kullanılabilir hale getirildi.
  - `vgextend`+`lvextend`+`resize2fs` ile hacim, mount'luyken kesintisiz genişletildi.
- **Kilometre Taşları & Çıktılar:**
  - 🏗️ LVM Yönetimi Notları: [11-Linux-LVM-Management](./11-Linux-LVM-Management/readme.md)

### 🔹 Gün 11 | Görev Zamanlama & Otomasyon (Cron & Timers)

_`crontab` sözdizimindeki 5 yıldızın (`* * * * *`) mantığını öğrenip dakikalık test görevleri çalıştırdım. Cron işlerinde göreli dosya yolu (relative path) kullanımının ortam değişkenleri (`$PATH` ve `$HOME`) kısıtlı olduğu için betiklerin sessizce çökmesine yol açtığını bizzat deneyimledim — tam yol (absolute path) ve `>> output.log 2>&1` ile hem standart çıktıyı hem hataları yönlendirmenin neden kritik olduğunu kavradım._

- **Görevler & Hedefler:**
  - `crontab -e` ve `crontab -l` ile kullanıcı bazlı zamanlanmış görevler tanımlandı.
  - Zamanlama sözdizimi (dakika, saat, gün, ay, haftanın günü) ve özel aralıklar (`*/5`, `0 2 * * *`) test edildi.
  - Otomasyon çıktılarının ve hata loglarının (stdout / stderr) dosyaya yönlendirilmesi sağlandı.
  - `systemctl list-timers` ile systemd tabanlı modern zamanlayıcıların durumu ve çalışma aralıkları incelendi.
- **Kilometre Taşları & Çıktılar:**
  - ⏰ Görev Zamanlama Notları: [12-Linux-Task-Scheduling](./12-Linux-Task-Scheduling/readme.md)

### 🔹 Gün 11.5 | SSH Yönetimi (Uzaktan Erişim Güvenliği)

_VirtualBox'ın varsayılan NAT modunda VM'e host'tan doğrudan IP ile bağlanamayacağımı keşfettim — VM host'un arkasında gizli kalıyor. Port Forwarding kuralıyla (host:2222 → VM:22) çözdüm. Ayrıca Windows'un yerleşik OpenSSH istemcisinde Linux/Mac'e özgü `ssh-copy-id` komutunun olmadığını, Rocky Linux minimal kurulumda da `nano`'nun bulunmadığını öğrendim — ikisi için de alternatif (`type | ssh ... cat >>` ve `sed -i`) buldum. En kritik ders: `PasswordAuthentication no` gibi riskli bir değişiklik yapmadan önce, olası bir hatada kendimi VM'den tamamen dışarıda bırakmamak için mevcut SSH oturumunu kapatmadan değişikliği ayrı bir pencereden test etmekti._

- **Görevler & Hedefler:**
  - `sshd` servis durumu ve dinlenen portlar (`ss -tlnp`) doğrulandı.
  - VirtualBox NAT + Port Forwarding ile host:2222 → VM:22 yönlendirmesi kuruldu.
  - `ed25519` SSH key çifti üretildi, public key manuel olarak `authorized_keys`'e aktarıldı ve şifresiz giriş doğrulandı.
  - `sshd_config` `sed` ile sertleştirildi (`PermitRootLogin no`, `PasswordAuthentication no`, `MaxAuthTries 3`, `ClientAliveInterval`/`CountMax`, `X11Forwarding no`), `sshd -t` ile syntax kontrolü ve güvenli restart yapıldı.
  - `firewalld` üzerinde `ssh` servisinin zaten açık geldiği doğrulandı.
  - `SELinux` durumu (`Enforcing`/`targeted`) kontrol edildi, port değişmediği için ekstra işlem gerekmediği teyit edildi.
  - EPEL + CRB üzerinden `fail2ban` kuruldu, `jail.local` ile SSH'a özel brute-force koruması (3 deneme/10dk → 1 saat ban) aktifleştirildi.
- **Kilometre Taşları & Çıktılar:**
  - 🔒 SSH Yönetimi Notları: [12-Linux-SSH-Management](./12-Linux-SSH-Management/readme.md)

### 🔹 Gün 12 | Forward Proxy / Reverse Proxy Kavramları

_Nginx'i reverse proxy olarak ilk kez denedim ve beklenmedik bir 502 Bad Gateway hatası aldım. Sebebini araştırınca `localhost`'un proxy açısından her zaman "bu makine" anlamına geldiğini, backend farklı bir VM'de çalıştığı için Nginx'in orada bir şey bulamadığını kavradım — bu, proxy kavramını ezbere değil gerçek bir hatadan öğrenmemi sağladı._

- **Görevler & Hedefler:**
  - Forward proxy ile reverse proxy arasındaki fark (istemci/sunucu önünde durma) kavramsal olarak öğrenildi.
  - Nginx'in `location` bloklarının reverse proxy mantığıyla (path → backend eşlemesi) örtüştüğü görüldü.
  - Backend olarak `python3 -m http.server 8080` başlatıldı, Nginx'te `proxy_pass http://localhost:8080;` tanımlandı.
  - `nginx -t` ile config test edildi, `systemctl restart nginx` ile uygulandı.
  - `curl localhost` ile test edildi → **502 Bad Gateway** alındı ve kök nedeni (backend farklı VM'de, `localhost` yanlış hedefi işaret ediyor) analiz edildi.
- **Kilometre Taşları & Çıktılar:**
  - 🔀 Proxy Kavramları Notları: [13-Forward-Reverse-Proxy](./13-Forward-Reverse-Proxy/readme.md)

---

## 🛠️ Ortam

- **Hypervisor:** VirtualBox 7.2.14
- **Guest OS:** Rocky Linux 10.2 (Red Quartz)
- **VM Kaynakları:** 4096 MB RAM, 4 vCPU, 20 GB Disk
- **Klavye Düzeni:** Türkçe (TR)

---

## 📚 Kaynak

Bu öğrenme yolculuğu [alifurkan-altuntas/devops-internship](https://github.com/alifurkan-altuntas/devops-internship) reposundaki müfredat sırası takip edilerek hazırlanmıştır.
