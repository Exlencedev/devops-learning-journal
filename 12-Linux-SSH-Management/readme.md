# 🔒 Linux SSH Yönetimi

Bu belge, SSH servis yönetimini kapsar: key-based authentication, `sshd_config` üzerinden güvenlik sertleştirmesi, firewalld/SELinux doğrulaması, ve fail2ban ile brute-force koruması.

---

## 1. SSH Servis Durumu

```bash
sudo systemctl status sshd
sudo ss -tlnp | grep ssh
```

`sshd.service` **enabled** + **active (running)** durumda, hem IPv4 (`0.0.0.0:22`) hem IPv6 (`[::]:22`) üzerinden dinliyor, PID 874.

### ℹ️ Not: Dağıtımlar Arası Servis Adı Farkı

Debian/Ubuntu'da servis adı `ssh`, RHEL tabanlı dağıtımlarda (Rocky Linux dahil) `sshd` — küçük ama karıştırılabilecek bir dağıtım farkı.

---

## 2. VirtualBox Ağ Yapılandırması

VM'in ağ modu **NAT** olduğu için (`ip a` çıktısında `10.0.2.15`), host makineden VM'e doğrudan bu IP ile bağlanmak mümkün değil — VM, host'un arkasında NAT edilmiş durumda ve host'a görünür değil.

**Çözüm:** VirtualBox → Ayarlar → Ağ → Port Forwarding kuralı eklendi:

| Ad  | Protokol | Host IP   | Host Port | Guest IP  | Guest Port |
|-----|----------|-----------|-----------|-----------|------------|
| SSH | TCP      | 127.0.0.1 | 2222      | 10.0.2.15 | 22         |

Bağlantı komutu (Windows host, PowerShell):

```powershell
ssh -p 2222 ege@127.0.0.1
```

---

## 3. Key-Based Authentication Kurulumu

Host (Windows) tarafında key üretildi:

```powershell
ssh-keygen -t ed25519 -C "ege-devops-journal"
```

### 🐛 Hata & Çözüm: Windows'ta `ssh-copy-id` Yok

`ssh-copy-id`, Linux/Mac'e özgü bir script olduğu için Windows'un yerleşik OpenSSH istemcisinde bulunmuyor. Onun yerine public key manuel olarak aktarıldı:

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 2222 ege@127.0.0.1 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

**Doğrulama:**

```powershell
ssh -p 2222 ege@127.0.0.1
```

Şifre sormadan direkt `[ege@localhost ~]$` prompt'una düşüldü. ✅

---

## 4. sshd_config Sertleştirme

Değişiklik öncesi yedek alındı:

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

### 🐛 Hata & Çözüm: `nano` Bulunamadı

Rocky Linux minimal kurulumda `nano` yüklü gelmiyor (`sudo: nano: komut bulunamadı`). `vi` ile manuel düzenleme yerine, tek seferde ve hatasız uygulamak için `sed` tercih edildi:

```bash
sudo sed -i \
  -e 's/^#\?PermitRootLogin.*/PermitRootLogin no/' \
  -e 's/^#\?PasswordAuthentication.*/PasswordAuthentication no/' \
  -e 's/^#\?PubkeyAuthentication.*/PubkeyAuthentication yes/' \
  -e 's/^#\?MaxAuthTries.*/MaxAuthTries 3/' \
  -e 's/^#\?ClientAliveInterval.*/ClientAliveInterval 300/' \
  -e 's/^#\?ClientAliveCountMax.*/ClientAliveCountMax 2/' \
  -e 's/^#\?X11Forwarding.*/X11Forwarding no/' \
  /etc/ssh/sshd_config
```

Doğrulama:

```bash
sudo grep -E "^PermitRootLogin|^PasswordAuthentication|^PubkeyAuthentication|^MaxAuthTries|^ClientAliveInterval|^ClientAliveCountMax|^X11Forwarding" /etc/ssh/sshd_config
```

```
PermitRootLogin no
MaxAuthTries 3
PubkeyAuthentication yes
PasswordAuthentication no
X11Forwarding no
ClientAliveInterval 300
ClientAliveCountMax 2
```

Syntax kontrolü ve restart:

```bash
sudo sshd -t
sudo systemctl restart sshd
```

### ⚠️ Ders: Self-Lockout Riskine Karşı Güvenli Test

`PasswordAuthentication no` uygulandıktan sonra key çalışmazsa VM'e SSH üzerinden erişim tamamen kaybedilir. Bu riske karşı mevcut SSH oturumu **kapatılmadan**, restart sonrası doğrulama **ayrı bir pencereden** yapıldı:

```powershell
ssh -p 2222 ege@127.0.0.1
```

Şifre sormadan bağlandı, sertleştirme başarıyla doğrulandı. ✅

---

## 5. Firewall (firewalld) Kontrolü

```bash
sudo systemctl status firewalld
sudo firewall-cmd --list-services
```

`firewalld` aktif, çıktı: `cockpit dhcpv6-client ssh` — **ssh servisi zaten açık geliyor**, Rocky Linux'ta varsayılan davranış. Ekstra işlem gerekmedi.

---

## 6. SELinux Durumu

```bash
getenforce
sudo sestatus
```

```
SELinux status:                 enabled
Current mode:                   enforcing
Loaded policy name:             targeted
```

SELinux **Enforcing** modda, varsayılan 22 portunda kaldığımız için (`ssh_port_t` zaten tanımlı) ekstra işlem gerekmedi. Port değiştirilseydi gerekli komut:

```bash
sudo semanage port -a -t ssh_port_t -p tcp <yeni_port>
```

---

## 7. fail2ban ile Brute-Force Koruması

```bash
sudo dnf install epel-release -y
```

### 🐛 Not: CRB Deposu Gerekliliği

EPEL kurulum çıktısında uyarı: *"Many EPEL packages require the CodeReady Builder (CRB) repository."* fail2ban'ın bazı bağımlılıkları için önce CRB etkinleştirildi:

```bash
sudo /usr/bin/crb enable
sudo dnf install fail2ban -y
```

SSH jail'i yapılandırıldı:

```bash
sudo tee /etc/fail2ban/jail.local <<EOF
[sshd]
enabled = true
port = ssh
maxretry = 3
bantime = 3600
findtime = 600
EOF

sudo systemctl enable --now fail2ban
sudo systemctl restart fail2ban
```

Doğrulama:

```bash
sudo fail2ban-client status
sudo fail2ban-client status sshd
```

**Sonuç:**

```
Status
|- Number of jail:      1
`- Jail list:   sshd

Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed:     0
|  `- Journal matches:  _SYSTEMD_UNIT=sshd.service + _COMM=sshd + _COMM=sshd-session
`- Actions
   |- Currently banned:  0
   |- Total banned:      0
   `- Banned IP list:
```

`sshd` jail aktif, journal üzerinden `sshd.service` loglarını izliyor. Kural: 10 dakika içinde 3 başarısız denemeden sonra 1 saat ban.

---

## 📊 Komut Referansı

| Komut | Amacı |
|-------|-------|
| **`ssh-keygen -t ed25519`** | Modern, güvenli bir SSH key çifti üretir |
| **`sshd -t`** | `sshd_config` dosyasının syntax'ını, servisi yeniden başlatmadan doğrular |
| **`sudo sed -i -e 's/.../.../'`** | Config dosyasında toplu, editörsüz bul-değiştir yapar |
| **`firewall-cmd --list-services`** | firewalld'de açık olan servisleri listeler |
| **`getenforce` / `sestatus`** | SELinux'un aktif modunu ve policy'sini gösterir |
| **`semanage port -a -t ssh_port_t -p tcp <port>`** | SELinux'a özel bir portu SSH için tanıtır |
| **`fail2ban-client status <jail>`** | Bir fail2ban jail'inin anlık durumunu ve ban listesini gösterir |

---

## 🧠 Quiz (Gerçek Sonuçlar)

| # | Soru | Cevap | Sonuç |
|---|------|-------|-------|
| 1 | `sshd_config`'te `PasswordAuthentication no` yapmadan önce neden mutlaka key-based authentication'ın çalıştığını test etmemiz gerekiyordu? | Test edilmezse VM'e şifreyle de key ile de giriş yapılamayabilir (self-lockout riski) | ✅ Doğru |
| 2 | VirtualBox'ta VM'in ağ modu NAT olduğunda host'tan VM'e doğrudan IP ile bağlanamayız. Bu sorunu hangi yöntemle çözdük? | VirtualBox Port Forwarding ile host:2222 portunu VM:22 portuna yönlendirerek | ✅ Doğru |

---

ℹ️ _Tüm komutlar yerel Rocky Linux VM'inde (Rocky9-Test), Windows host üzerinden VirtualBox NAT + Port Forwarding ile test edilmiştir. Bu fazda 2 pratik engelle karşılaşıldı: Windows'ta `ssh-copy-id` bulunmayışı ve Rocky Linux minimal kurulumda `nano` bulunmayışı — ikisi de dağıtım/platform bağımlı alternatiflerle (`type | ssh ... cat >>` ve `sed -i`) çözülmüştür._
