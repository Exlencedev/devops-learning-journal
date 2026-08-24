# 🚀 DevOps & Linux Öğrenme Yolculuğu

Bu repo, Linux ve DevOps temellerini öğrenirken tuttuğum günlük notları belgeleyen bir öğrenme günlüğüdür. [alifurkan-altuntas/devops-internship](https://github.com/alifurkan-altuntas/devops-internship) reposundaki müfredatı takip ederek, VirtualBox üzerinde kurduğum Rocky Linux VM'inde adım adım ilerliyorum.

## 📍 Şu An Neredeyim

Mini Proje fazını (Faz 17) tamamladım: gerçek kiralık bir sunucu yerine WSL2 üzerinde Ubuntu 26.04 kurup, ayrı bir sudo kullanıcısı (`egeadmin`), SSH anahtar tabanlı erişim, Nginx ve Docker'ı sıfırdan tek bir ortamda bir araya getirdim. Kendi bu reponuzdaki bir `index.html` sayfasını `git clone` ile sunucuya çekip Nginx üzerinden yayınladım. Süreçte üç gerçek hatayla karşılaştım: yanlış yazılmış bir gizli klasör yolu (`~/ssh.` yerine `~/.ssh`), `su -` komutunda unutulan bir boşluk (`-egeadmin` bir seçenek sanıldı), ve en öğreticisi — `git clone`'u Windows dosya sisteminde (`/mnt/c/WINDOWS/system32`) çalıştırınca alınan `chmod ... Operation not permitted` hatası, çözümü Linux native ev dizinine geçmekti.

Bundan önce, Git — Branch, Merge ve Push Çakışması fazını tamamladım. `git checkout -b` ile branch oluşturup izolasyonu kanıtladım, sonra `git merge` ile fast-forward birleştirme yaptım. Ardından gerçek bir push çakışması senaryosu kurdum: aynı README.md'yi hem VM'de hem GitHub web arayüzünden bağımsız değiştirip `git push`'un reddedilmesini, `git pull`'un modern Git'te stratejinin açıkça belirtilmesini istemesini, ve gerçek bir merge çakışmasını elle çözdüm — sonunda GitHub'ın artık düz şifre değil, Personal Access Token istediğini de deneyimledim. Bunun ortasında hiç beklenmedik bir kriz yaşadım: `git clone` denerken VM'in interneti tamamen kesildi, teşhis sonunda 10. fazdan kalma bir loop device mount girdisinin `/etc/fstab`'da unutulduğunu ve VM'i her yeniden başlatmada tüm sistemi "emergency mode"a düşürdüğünü keşfettim — `fstab`'dan temizleyip sistemi kurtardım.

Ondan önce, Cron & Otomasyon fazını tamamladım. 14. fazdaki disk kullanım scriptini `crontab` ile her gece 02:00'de çalışacak şekilde zamanladım ve iki gerçek hatayla karşılaştım: `vi`'de uzun bir crontab satırının görsel olarak sarılıp aslında ikiye bölünmesi yüzünden `bad minute` hatası aldım, ve script içine eklediğim bir `sudo` komutunun, cron'un terminal'siz çalışma bağlamında "a password is required" hatasıyla sessizce başarısız olduğunu `at` ile simüle ederek bizzat gördüm. İkinci sorunu, geniş yetki vermek yerine `visudo` ile sadece o tek komuta özel dar kapsamlı bir `NOPASSWD` kuralıyla çözdüm.

Sırada CI/CD ve Konteynerizasyon fazları var.

---

## 📁 Repo Yapısı

- [00-VM-Setup](./00-VM-Setup/): VirtualBox kurulumu ve Rocky Linux 10.2'nin ISO'dan manuel kurulumu.
- [01-Linux-Basics](./01-Linux-Basics/): Sistem kimliği komutları (`hostname`, `hostnamectl`) ve pipe tabanlı metin işleme (`grep`, `cut`, `awk`, `tr`).
- [02-Vagrant-Automation](./02-Vagrant-Automation/): Windows host'a Vagrant kurulumu, VirtualBox provider ile otomatik VM oluşturma (`vagrant up`, `vagrant ssh`).
- [03-File-System-Management](./03-File-System-Management/): `dd` ile test dosyası oluşturma, XFS speculative preallocation keşfi ve en büyük dosyaları bulma pipeline'ı.
- [04-User-Privilege-Management](./04-User-Privilege-Management/): `visudo` ile En Düşük Yetki Prensibi — kısıtlı bir kullanıcıya sadece nginx komutlarına özel, şifresiz sudo yetkisi tanımlama.
- [05-Linux-Permissions](./05-Linux-Permissions/): `chmod` sayısal sistemi, sticky bit ile paylaşımlı dizin koruması, `chown`/`chgrp`, `umask` matematiği.
- [06-Linux-Process-Management](./06-Linux-Process-Management/): `top` ile işlem izleme, gerçek bir kernel "soft lockup" krizi ve çözümü, `kill` sinyalleri, `nice`/`renice` ile önceliklendirme.
- [07-Linux-Service-Management](./07-Linux-Service-Management/): `systemd` ile servis yönetimi (`enable`/`start`/`reload`/`restart`), `journalctl` ile log filtreleme.
- [08-Linux-Log-Analysis](./08-Linux-Log-Analysis/): nginx access log analizi (`awk`/`grep`/`sort`/`uniq` pipeline'ları), `sed` ile bul-değiştir ve satır silme.
- [09-Linux-Network-Management](./09-Linux-Network-Management/): `dig` ile DNS sorgulama, `ss` ile dinleme portları, `openssl` ile TLS sertifika doğrulama.
- [10-Linux-Storage-Management](./10-Linux-Storage-Management/): loop device ile sanal disk oluşturma, `fdisk` ile bölümleme, `mkfs.ext4`, `/etc/fstab` ile UUID tabanlı kalıcı mount.
- [11-Linux-LVM-Management](./11-Linux-LVM-Management/): PV/VG/LV katmanları, `fallocate` ile güvenli test diski, `vgextend` + `lvextend` + `resize2fs` ile kesintisiz depolama genişletme.
- [12-Linux-Task-Scheduling](./12-Linux-Task-Scheduling/): `crontab` sözdizimi, periyodik arka plan otomasyonu, standart çıktı/hata yönlendirmesi (`>>` ve `2>&1`), `systemd timer` ile modern zamanlama.
- [12-Linux-SSH-Management](./12-Linux-SSH-Management/): SSH servis yönetimi, VirtualBox NAT port forwarding, key-based authentication, `sshd_config` sertleştirme, firewalld/SELinux doğrulaması, fail2ban ile brute-force koruması.
- [13-Forward-Reverse-Proxy](./13-Forward-Reverse-Proxy/): Forward proxy ve reverse proxy kavramları, Nginx `location`/`proxy_pass` direktifleri, ilk reverse proxy denemesinde alınan 502 Bad Gateway hatası ve kök nedeni.
- [14-Linux-Bash-Scripting](./14-Linux-Bash-Scripting/): Bash ile disk kullanımını kontrol eden `disk_check.sh` scripti, `df`/`awk`/`tr` pipeline'ı, `%80` eşik kontrolü ve Git ile versiyonlama.
- [15-Linux-Cron-Automation](./15-Linux-Cron-Automation/): `disk_check.sh`'ı `crontab` ile zamanlama, `vi`'de bölünen satırdan kaynaklı `bad minute` hatası, cron/at bağlamında terminal'siz `sudo` başarısızlığı ve dar kapsamlı `sudoers` çözümü, Nginx `logrotate` config'ine bakış.
- [16-Linux-Git-Basics](./16-Linux-Git-Basics/): `git branch`/`git merge` ile fast-forward birleştirme, gerçek bir push çakışması ve merge çakışması çözümü, GitHub Personal Access Token ile kimlik doğrulama, ve bir `/etc/fstab` girdisinin sebep olduğu emergency mode krizinin çözümü.
- [17-Mini-Project](./17-Mini-Project/): WSL2/Ubuntu üzerinde sıfırdan sunucu kurulumu — ayrı sudo kullanıcısı, SSH anahtar tabanlı erişim, Nginx, Docker ve Git'in bir arada kullanımı; kendi repodaki bir sayfanın `git clone` + `cp` ile canlıya alınması.

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

_Bu fazda 3 gerçek hatayla karşılaştım: `losetup -fP` yerine `-fp` yazdım, mount komutunda bir race condition yaşadım (mkdir ve mount çok hızlı ardışık çalıştırınca), ve en önemlisi `/etc/fstab`'a satır eklerken tırnak kullanımı yüzünden satır ikiye bölündü. Önceden aldığım yedek (`fstab.backup`) sayesinde saniyeler içinde güvenli şekilde geri dönebildim — kritik sistem dosyalarında yedek almanın neden önemli olduğunu bizzat deneyimledim._

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
  - `vgextend` + `lvextend` + `resize2fs` ile hacim, mount'luyken kesintisiz genişletildi.
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

### 🔹 Gün 13 | Bash Scripting & Disk Kullanım Kontrolü

_Bash scripting fazında ilk kez gerçek bir sistem bilgisini otomatik olarak kontrol eden küçük bir script yazdım. `df` çıktısından disk kullanım yüzdesini `awk` ve `tr` ile ayıklarken birkaç yazım hatası yaptım (`Df`, `PR0NT`, `if` koşulundaki boşluk eksikliği) ve hata mesajlarını okuyarak debug ettim. Ayrıca scriptin `%80` üzerindeki davranışını gerçek diski doldurmadan `usage=90` ile simüle ederek test ettim._

- **Görevler & Hedefler:**
  - `df -h /` ile root (`/`) disk bölümünün kullanım durumu kontrol edildi.
  - `disk_check.sh` adlı Bash scripti oluşturuldu.
  - `df`, `awk` ve `tr` kullanılarak disk kullanım yüzdesi otomatik olarak alındı.
  - `if/else` kullanılarak `%80` disk kullanım eşiği oluşturuldu.
  - Disk kullanımı normal olduğunda `Disk usage is normal.` mesajı gösterildi.
  - `%80` üzerindeki durum `usage=90` ile simüle edilerek `WARNING` mesajı test edildi.
  - `chmod +x` ile script çalıştırılabilir hale getirildi.
  - `./disk_check.sh` ile script gerçek sistem üzerinde çalıştırıldı.
  - `git init`, `git add` ve `git commit` kullanılarak script Git repository'sine kaydedildi.
  - `git commit -m "Add disk usage check script"` ile commit oluşturuldu.
- **Kilometre Taşları & Çıktılar:**
  - 📊 Bash Scripting Notları: [14-Linux-Bash-Scripting](./14-Linux-Bash-Scripting/readme.md)

### 🔹 Gün 14 | Cron & Otomasyon (Gerçek Bir `sudo`-Cron Çakışması)

_13. fazdaki `disk_check.sh`'ı `crontab` ile her gece 02:00'e bağlarken, `vi`'de uzun satırın terminal genişliği yüzünden görsel olarak sarılıp aslında ikiye bölündüğünü fark etmeden kaydettim — `bad minute` hatası aldım, çözümü editörsüz bir shell pipe kalıbı (`(crontab -l; echo "...") | crontab -`) oldu. Asıl öğretici kısım script'e kasıtlı olarak eklediğim bir `sudo` komutuyla geldi: elle çalıştırınca sorunsuzdu, ama cron'un gerçek çalışma bağlamını `at now + 1 minute` ile simüle edince `sudo`'nun hiçbir terminal bulamayıp "a password is required" diyerek sessizce başarısız olduğunu bizzat gördüm — `2>&1` ile log'a yönlendirmemiş olsam bu hata muhtemelen hiç fark edilmeyecekti. Geniş yetki vermek yerine `visudo` ile sadece o tek komuta özel dar kapsamlı bir `NOPASSWD` kuralı tanımlayarak çözdüm. Sonda Nginx'in `logrotate` config'ini `-f` ile zorla tetikleyip `delaycompress`'in ve durum takip dosyasının nasıl çalıştığını canlı doğruladım._

- **Görevler & Hedefler:**
  - `disk_check.sh`, `crontab -e` ile her gece 02:00'de çalışacak şekilde zamanlandı.
  - `vi`'de satırın ikiye bölünmesinden kaynaklanan `bad minute` hatası debug edildi, editörsüz bir kalıpla çözüldü.
  - Script'e kasıtlı olarak `sudo systemctl status sshd` satırı eklendi.
  - `sudo -k` ve `at now + 1 minute` ile cron'un terminal'siz bağlamı gerçekçi şekilde simüle edildi, gerçek bir `sudo: a password is required` hatası yakalandı.
  - `visudo` ile sadece o komuta özel dar kapsamlı bir `NOPASSWD` kuralı tanımlandı ve doğrulandı (`sudo -l -U ege`).
  - Aynı senaryo tekrar test edildi, hatasız tamamlandığı doğrulandı.
  - Nginx'in `/etc/logrotate.d/nginx` config'i incelendi, `logrotate -f` ile canlı tetiklendi, `create`/`delaycompress`/durum dosyası davranışı gözlemlendi.
- **Kilometre Taşları & Çıktılar:**
  - ⏰ Cron & Otomasyon Notları: [15-Linux-Cron-Automation](./15-Linux-Cron-Automation/readme.md)

### 🔹 Gün 15 | Git — Branch, Merge ve Gerçek Bir Push Çakışması

_Bu faz, planlanandan çok daha zengin geçti. Branch/merge kısmı sorunsuzdu, ama gerçek push çakışması senaryosunu (ayrı bir test reposunda) kurarken `git clone` sırasında VM'in interneti tamamen kesildi. Teşhis beni `NetworkManager`'ın çökmüş olduğunu görmeye, onu yeniden başlatmayı denerken de VM'in "emergency mode"a düşmesine götürdü. `journalctl -xb` ile kök nedeni buldum: 10. fazdan (Depolama Yönetimi) kalma bir loop device mount girdisi hâlâ `/etc/fstab`'daydı — loop device'lar kalıcı olmadığı için VM her yeniden başladığında sistem olmayan bir UUID'yi bağlamaya çalışıp tüm boot sürecini kilitliyordu. `fstab`'dan temizleyip sistemi kurtardıktan sonra asıl Git senaryosuna devam ettim: aynı README.md'yi hem yerelde hem GitHub'da değiştirip gerçek bir push reddi ve merge çakışması yaşadım, elle çözdüm, ve GitHub'ın artık şifre değil Personal Access Token istediğini ilk elden öğrendim._

- **Görevler & Hedefler:**
  - `git checkout -b test-branch` ile branch oluşturuldu, `main`'in aslında `master` olduğu keşfedildi.
  - Branch izolasyonu gerçek dosya testiyle kanıtlandı, `git merge` ile fast-forward birleştirme yapıldı.
  - Ayrı bir test reposu (`git-test-lab`) oluşturulup clone edildi.
  - `git clone` sırasında internet tamamen kesildi — `NetworkManager` servisinin çöktüğü, ardından VM'in emergency mode'a düştüğü tespit edildi.
  - `journalctl -xb` ile kök neden bulundu: 10. fazdan kalma bir loop device `/etc/fstab` girdisi boot'u kilitliyordu — `sed` ile temizlenip sistem kurtarıldı.
  - Aynı `README.md` hem VM'de hem GitHub web arayüzünden bağımsız değiştirilip gerçek bir `git push` reddi tetiklendi.
  - `git config pull.rebase false` ile pull stratejisi belirlendi, gerçek bir merge çakışması elle çözüldü.
  - GitHub Personal Access Token oluşturulup `git remote set-url` ile kimlik doğrulaması tamamlandı, push başarıyla gerçekleşti.
- **Kilometre Taşları & Çıktılar:**
  - 🔧 Git Notları: [16-Linux-Git-Basics](./16-Linux-Git-Basics/readme.md)

### 🔹 Gün 16 | Mini Proje: Nginx, Docker, Git & SSH Bir Arada

_Kaynak müfredat bu fazı gerçek kiralık bir sunucuda anlatıyordu; elimde kiralık sunucu olmadığı için ortamı WSL2 üzerinde Ubuntu 26.04 ile simüle ettim (WSL, Windows'un arkasında NAT'landığı için gerçek bir public IP deneyimi yaşanmadı, bunun yerine yerel IP üzerinden test edildi). Üç gerçek hatayla karşılaştım: `~/ssh.` yazıp `~/.ssh` demek istediğimi fark etmemek, `sudo su -egeadmin` yazınca `-` ile kullanıcı adı arasında boşluk unutup sistemin bunu geçersiz bir seçenek sanması, ve en öğreticisi — `git clone`'u yanlışlıkla Windows dosya sisteminde (`/mnt/c/WINDOWS/system32`) çalıştırınca alınan `chmod ... Operation not permitted` hatası. Kaynak repodaki hazır sayfa yerine, kendi bu journal reponun içine bir `17-Mini-Project/index.html` ekleyip onu sunucuya klonlayıp yayınladım — böylece "canlıya alınan" içerik kendi projeme ait oldu._

- **Görevler & Hedefler:**
  - Windows üzerinde WSL2 + Ubuntu 26.04 sıfırdan kuruldu.
  - En Düşük Yetki Prensibi doğrultusunda ayrı bir sudo kullanıcısı (`egeadmin`) oluşturuldu ve doğrulandı.
  - `egeadmin` için SSH anahtar tabanlı erişim kuruldu (Windows tarafındaki mevcut `id_ed25519` anahtar çifti kullanıldı), `openssh-server` kurulup etkinleştirildi.
  - Windows PowerShell'den WSL'e şifresiz SSH bağlantısı doğrulandı.
  - Nginx kuruldu, `curl localhost` ve tarayıcı üzerinden doğrulandı.
  - Docker, resmi Ubuntu kurulum adımlarıyla (`.sources` formatı) kuruldu ve `hello-world` ile doğrulandı.
  - Git kurulup bu repo (`devops-learning-journal`) sunucuya klonlandı.
  - `17-Mini-Project/index.html`, `cp` ile Nginx'in servis ettiği `/var/www/html/index.html` konumuna kopyalanıp canlıya alındı.
- **Kilometre Taşları & Çıktılar:**
  - 🚀 Mini Proje Notları: [17-Mini-Project](./17-Mini-Project/readme.md)

---

## 🛠️ Ortam

- **Hypervisor:** VirtualBox 7.2.14
- **Guest OS:** Rocky Linux 10.2 (Red Quartz)
- **VM Kaynakları:** 4096 MB RAM, 4 vCPU, 20 GB Disk
- **Klavye Düzeni:** Türkçe (TR)
- **Ek Ortam (Faz 17):** WSL2 üzerinde Ubuntu 26.04 LTS (gerçek kiralık sunucu simülasyonu için)

---

## 📚 Kaynak

Bu öğrenme yolculuğu [alifurkan-altuntas/devops-internship](https://github.com/alifurkan-altuntas/devops-internship) reposundaki müfredat sırası takip edilerek hazırlanmıştır.
