# Faz 12: Linux SSH Management

**Ortam:** Rocky Linux 10.2 (VirtualBox VM) · Host: Windows · VM kullanıcısı: `ege`
**Kaynak:** [alifurkan-altuntas/devops-internship](https://github.com/alifurkan-altuntas/devops-internship)
**Tarih:** Ağustos 2026

## Amaç

SSH servisini yönetmek, key-based authentication kurmak, `sshd_config`
üzerinden güvenlik sertleştirmesi yapmak ve brute-force saldırılarına karşı
fail2ban ile koruma sağlamak.

## Yapılanlar

### 1. SSH Servis Durumu

```bash
sudo systemctl status sshd
sudo ss -tlnp | grep ssh
```

`sshd.service` enabled + active (running) durumda, 22 portunda hem IPv4
(`0.0.0.0:22`) hem IPv6 (`[::]:22`) üzerinden dinliyor.

**Not:** Debian/Ubuntu'da servis adı `ssh`, RHEL tabanlı dağıtımlarda
(Rocky dahil) `sshd`.

### 2. VirtualBox Ağ Yapılandırması

VM'in ağ modu **NAT** olduğu için host makineden VM'in IP'sine
(`10.0.2.15`) doğrudan bağlanmak mümkün değil — VM, host'un arkasında NAT
edilmiş durumda ve host'a görünür değil.

**Çözüm:** VirtualBox → Ayarlar → Ağ → Port Forwarding kuralı eklendi:

| Ad  | Protokol | Host IP   | Host Port | Guest IP  | Guest Port |
|-----|----------|-----------|-----------|-----------|------------|
| SSH | TCP      | 127.0.0.1 | 2222      | 10.0.2.15 | 22         |

Bağlantı komutu:

```powershell
ssh -p 2222 ege@127.0.0.1
```

### 3. Key-Based Authentication

Host (Windows) tarafında key üretildi:

```powershell
ssh-keygen -t ed25519 -C "ege-devops-journal"
```

Windows'ta `ssh-copy-id` komutu bulunmadığı için (Linux/Mac'e özgü bir
script), public key manuel olarak aktarıldı:

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh -p 2222 ege@127.0.0.1 "mkdir -p ~/.ssh && chmod 700 ~/.ssh && cat >> ~/.ssh/authorized_keys && chmod 600 ~/.ssh/authorized_keys"
```

Doğrulama: `ssh -p 2222 ege@127.0.0.1` şifre sormadan bağlandı ✅

### 4. sshd_config Sertleştirme

Değişiklik öncesi yedek alındı:

```bash
sudo cp /etc/ssh/sshd_config /etc/ssh/sshd_config.bak
```

Rocky Linux minimal kurulumda `nano` bulunmadığı için `sed` ile toplu
değişiklik yapıldı:

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

Uygulanan ayarlar:

```
PermitRootLogin no
PasswordAuthentication no
PubkeyAuthentication yes
MaxAuthTries 3
ClientAliveInterval 300
ClientAliveCountMax 2
X11Forwarding no
```

Syntax kontrolü ve servis restart:

```bash
sudo sshd -t
sudo systemctl restart sshd
```

**Kritik nokta:** `PasswordAuthentication no` uygulanmadan önce key ile
girişin çalıştığı doğrulandı ve mevcut SSH oturumu kapatılmadan yeni bir
pencereden test edildi — böylece bir hata durumunda geri dönüş imkânı
korunmuş oldu. Restart sonrası yeni pencereden şifresiz giriş doğrulandı ✅

### 5. Firewall (firewalld)

```bash
sudo systemctl status firewalld
sudo firewall-cmd --list-services
```

`firewalld` aktif, `ssh` servisi Rocky Linux'ta varsayılan olarak zaten
açık geliyor (`cockpit dhcpv6-client ssh`). Ekstra işlem gerekmedi.

### 6. SELinux Kontrolü

```bash
getenforce
sudo sestatus
```

SELinux `Enforcing` modda, `targeted` policy aktif. Varsayılan 22 portunda
kaldığımız için (`ssh_port_t` zaten tanımlı) ekstra bir SELinux işlemi
gerekmedi. Port değiştirilseydi gerekli komut:

```bash
sudo semanage port -a -t ssh_port_t -p tcp <yeni_port>
```

### 7. fail2ban Kurulumu

EPEL ve CRB depoları etkinleştirildi:

```bash
sudo dnf install epel-release -y
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

Sonuç: `sshd` jail aktif, journal üzerinden `sshd.service` loglarını
izliyor. Kural: 10 dakika içinde 3 başarısız denemeden sonra 1 saat ban.

## Öğrenilenler / Notlar

- RHEL tabanlı dağıtımlarda servis adı `sshd`, Debian tabanlılarda `ssh`.
- VirtualBox NAT modunda host-VM arası doğrudan IP erişimi yok; port
  forwarding (veya Bridged/Host-only adapter) gerekiyor.
- Windows'un yerleşik OpenSSH istemcisinde `ssh-copy-id` yok, manuel
  `cat >> authorized_keys` yöntemi kullanılmalı.
- Rocky Linux minimal kurulumda `nano` yok, `vi` veya `sed` kullanılmalı.
- `PasswordAuthentication no` gibi riskli değişikliklerden önce her zaman
  mevcut oturumu açık tutup yeni bağlantıyı ayrı pencereden test etmek
  gerekir (self-lockout riskine karşı).
- SELinux `Enforcing` modda özel port kullanılacaksa `semanage port`
  komutu şart, aksi halde bağlantı reddedilir.

## Quiz

**S1:** `sshd_config` dosyasında `PasswordAuthentication no` yapmadan önce
neden mutlaka key-based authentication'ın çalıştığını test etmemiz
gerekiyordu?

**S2:** VirtualBox'ta VM'in ağ modu NAT olduğunda, host makineden VM'e
doğrudan IP üzerinden SSH ile neden bağlanamıyoruz ve bu sorunu hangi
yöntemle çözdük?
