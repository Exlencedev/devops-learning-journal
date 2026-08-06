# 🤖 Vagrant Otomasyonu

Bu belge, Vagrant kullanarak sanal makine kurulumunu otomatikleştirme sürecini kapsıyor. Daha önce `00-VM-Setup`'ta VirtualBox'ı manuel kurup ISO'dan Rocky Linux kurmuştum — bu faz, aynı sonuca (çalışan bir Linux VM'i) tek bir komutla nasıl ulaşılacağını gösterdi.

---

## 1. Vagrant Nedir?

Vagrant, bir `Vagrantfile` (metin tabanlı yapılandırma dosyası) ve `vagrant up` komutu ile sanal makine kurulumunu otomatikleştiren bir araç. Bir "box" (önceden yapılandırılmış OS kalıbı) seçilir, `Vagrantfile`'da kaynak ayarları tanımlanır, ve `vagrant up` ile VM otomatik kurulur — Anaconda kurulum ekranından elle geçmeye gerek kalmaz.

**Önemli mimari detay:** Vagrant, VM'in **içine değil**, VirtualBox'ın çalıştığı **host makineye** (benim durumumda Windows) kurulur — çünkü Vagrant, VirtualBox'ı dışarıdan yönetir. VM'in içine kurulsa, dışarıdaki VirtualBox'a ulaşamaz.

---

## 2. Kurulum (Windows Host'ta)

```powershell
# developer.hashicorp.com/vagrant/install adresinden
# Windows AMD64 .msi indirildi ve kuruldu

vagrant --version
# Çıktı: Vagrant 2.4.9
```

---

## 3. Provider Seçimi: VirtualBox

Orijinal müfredatta Vagrant, **VMware** provider'ı ile kullanılmış ve iki spesifik sorun yaşanmış:
1. VMware provider bulunamadı hatası (`vagrant-vmware-desktop` plugin'i gerekiyormuş)
2. Yanlış box adı (`rocky Linux/9` yerine `generic/rocky9` olması gerekiyormuş) → 404 hatası

**Ben zaten VirtualBox kullandığım için**, VMware'i ayrıca kurmak yerine Vagrant'ın varsayılan/yerleşik **VirtualBox** provider'ını kullandım — ekstra plugin gerekmedi, ve doğru box adını (`generic/rocky9`) baştan kullanarak 404 hatasını hiç yaşamadım.

---

## 4. VM'i Ayağa Kaldırma

```powershell
cd $HOME\Desktop
mkdir vagrant-test
cd vagrant-test

vagrant init generic/rocky9
```
**Çıktı:** `A 'Vagrantfile' has been placed in this directory...`

```powershell
vagrant up
```

### Süreç Adımları (Vagrant'ın otomatik yaptıkları)

1. Box indirildi (`generic/rocky9` v4.3.12, virtualbox amd64) — checksum doğrulandı.
2. VirtualBox'a import edildi, VM adı otomatik oluşturuldu.
3. NAT ağ arayüzü hazırlandı, port yönlendirmesi yapıldı (`22 (guest) → 2222 (host)`).
4. VM boot edildi, SSH anahtarı otomatik değiştirildi (güvenlik için "insecure key" yerine yeni bir keypair üretildi).
5. **`Machine booted and ready!`**

### ⚠️ Küçük Uyarı (Zararsız)

```
The guest additions on this VM do not match the installed version of VirtualBox!
Guest Additions Version: 6.1.48
VirtualBox Version: 7.2
```

Vagrant'ın kendisi de belirttiği gibi, bu çoğu durumda sorun değil — sadece paylaşımlı klasörlerde (shared folders) sorun çıkarabilir. Paylaşımlı klasör kullanılmadığı için görmezden gelindi.

---

## 5. VM'e Bağlanma ve Doğrulama

```powershell
vagrant ssh
```
Hiçbir şifre sormadan, otomatik SSH anahtarıyla doğrudan bağlantı sağlandı: `[vagrant@rocky9 ~]$`

```bash
hostnamectl
cat /etc/os-release | grep PRETTY_NAME
```
**Çıktı:** `PRETTY_NAME="Rocky Linux 9.3 (Blue Onyx)"`

Böylece elimde artık iki farklı VM var:
- **`Rocky9-Test`** (VirtualBox, manuel kurulum, Rocky Linux **10.2**)
- **Vagrant VM** (otomatik, `vagrant up` ile, Rocky Linux **9.3**) — orijinal müfredatın kullandığı major sürümle birebir eşleşiyor.

---

## 📊 Komut Referansı

| Komut | Amacı | Örnek |
|-------|-------|-------|
| **`vagrant init <box>`** | Klasöre bir `Vagrantfile` yerleştirir | `vagrant init generic/rocky9` |
| **`vagrant up`** | Box'ı indirir (gerekirse), VM'i oluşturur ve başlatır | `vagrant up` |
| **`vagrant ssh`** | VM'e otomatik SSH anahtarıyla bağlanır | `vagrant ssh` |
| **`vagrant halt`** | VM'i durdurur (silmez) | `vagrant halt` |
| **`vagrant destroy`** | VM'i tamamen siler | `vagrant destroy` |

---

## 🧠 Öğrenilen Kavram

Manuel kurulumda yaptığım tüm adımlar (ISO indirme, Anaconda ekranından geçme, kullanıcı oluşturma, ağ yapılandırma) Vagrant'ta **tek bir `vagrant up` komutuna** indirgendi. Bu, "Infrastructure as Code" (IaC) mantığının somut bir örneği — altyapı elle değil, bir yapılandırma dosyasıyla (`Vagrantfile`) tanımlanıyor, tekrarlanabilir ve paylaşılabilir hale geliyor.

---

ℹ️ _Tüm adımlar Windows host üzerinde, VirtualBox provider'ı ile test edilmiştir._
