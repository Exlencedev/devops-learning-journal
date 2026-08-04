# 🖥️ VirtualBox Üzerinde Rocky Linux Kurulumu

Bu belge, sıfırdan VirtualBox kurup, üzerine Rocky Linux 10.2 kurma sürecini kapsıyor. Orijinal müfredatta bu adım Vagrant ile otomatikleştiriliyor (bkz. 02-Vagrant-Automation), ancak bu süreci manuel yaparak sanal makine kurulumunun her adımını birebir görmeyi tercih ettim.

---

## 1. VirtualBox Kurulumu

Resmi siteden (`virtualbox.org/wiki/Downloads`) işletim sistemine uygun sürüm indirildi.

- **Sürüm:** VirtualBox 7.2.14

---

## 2. Rocky Linux ISO İndirme

`rockylinux.org/download` üzerinden minimal ISO indirildi.

- **İndirilen sürüm:** Rocky-10.2-x86_64-minimal.iso
- **Not:** Minimal ISO tercih edildi çünkü sadece komut satırı içeriyor (GUI yok), staj müfredatındaki kullanım da bu şekilde.

---

## 3. Sanal Makine Oluşturma

VirtualBox'ta "Yeni Sanal Makine" sihirbazı ile aşağıdaki ayarlarla VM oluşturuldu:

| Ayar | Değer |
|------|-------|
| VM Adı | Rocky9-Test |
| ISO Kalıbı | Rocky-10.2-x86_64-minimal.iso |
| İş Dağıtımı | Red Hat (64-bit) |
| Ana Bellek (RAM) | 4096 MB |
| İşlemci Sayısı | 4 |
| Disk Boyutu | 20 GB |
| EFI | Kapalı |

### 🐛 Küçük Not

İndirme sayfasından otomatik olarak Rocky Linux **10** indi (repo müfredatında 9.8 kullanılmış). Aradaki fark, `dnf`, `systemctl`, `firewalld` gibi temel komutlar açısından pratik olarak önemsiz olduğu için Rocky Linux 10 ile devam edildi.

---

## 4. Kurulum Ekranı (Anaconda Installer)

Kurulum özetinde 3 zorunlu adım tamamlandı:

1. **Kurulum Hedefi** → Disk otomatik partition'landı
2. **Kök Hesabı (root)** → Şifre belirlendi
3. **Kullanıcı Oluşturma** → `ege` kullanıcısı oluşturuldu, yönetici (sudo) yetkisi verildi

Kurulum tamamlandıktan sonra `sudo reboot` ile sistem yeniden başlatıldı (yeni kurulan kernel'in aktif olması için).

---

## 5. İlk Giriş ve Sistem Güncellemesi

```bash
sudo dnf update -y
```

Kernel dahil tüm sistem paketleri güncellendi. Kernel güncellemesi olduğu için tekrar `sudo reboot` ile yeniden başlatıldı.

### 🔍 Gözlem

`hostnamectl` çıktısında `Virtualization: oracle` ve `Chassis: vm` bilgileri, sistemin VirtualBox üzerinde çalıştığını doğru şekilde tespit ettiğini gösterdi. `Hardware Vendor: innotek GmbH` de VirtualBox'ın sanal donanım imzası.

---

## 📊 Kurulum Özeti Referans Tablosu

| Adım | Amaç |
|------|------|
| ISO indirme | Kurulacak işletim sisteminin kalıbını edinmek |
| VM oluşturma | Sanal donanımı (RAM/CPU/Disk) tanımlamak |
| Anaconda kurulumu | OS'i diske kurmak, kullanıcı/root belirlemek |
| `dnf update -y` | Sistemi güncel paket ve kernel ile senkronize etmek |

---

ℹ️ _Tüm adımlar yerel VirtualBox ortamında test edilmiştir._
