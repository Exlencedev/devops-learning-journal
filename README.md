# 🚀 DevOps & Linux Öğrenme Yolculuğu

Bu repo, Linux ve DevOps temellerini öğrenirken tuttuğum günlük notları belgeleyen bir öğrenme günlüğüdür. [alifurkan-altuntas/devops-internship](https://github.com/alifurkan-altuntas/devops-internship) reposundaki müfredatı takip ederek, VirtualBox üzerinde kurduğum Rocky Linux VM'inde adım adım ilerliyorum.

## 📍 Şu An Neredeyim

VirtualBox üzerinde Rocky Linux 10.2 kurdum ve sistemi güncel tuttum. Linux temellerinde `hostname`/`hostnamectl` ile sistem kimliğini, `grep`/`cut`/`awk`/`tr` ile pipe tabanlı metin işlemeyi öğrendim. Dosya sistemi yönetiminde `dd` ile test dosyası oluşturdum, bu sırada XFS'in "speculative preallocation" davranışını keşfettim ve `stat` ile doğruladım — ardından `find`/`sort`/`head` pipeline'ı ile en büyük dosyaları listelemeyi öğrendim.

Vagrant fazını, VM'i zaten manuel kurduğum için atladım (bkz. aşağıdaki not).

Sırada kullanıcı/sudo yetki yönetimi (Least Privilege Prensibi) var.

---

## 📁 Repo Yapısı

- [00-VM-Setup](./00-VM-Setup/): VirtualBox kurulumu ve Rocky Linux 10.2'nin ISO'dan manuel kurulumu.
- [01-Linux-Basics](./01-Linux-Basics/): Sistem kimliği komutları (`hostname`, `hostnamectl`) ve pipe tabanlı metin işleme (`grep`, `cut`, `awk`, `tr`).
- [02-Vagrant-Automation](./02-Vagrant-Automation/): Atlandı — VM zaten manuel kurulduğu için. Kavramsal özet içeriyor.
- [03-File-System-Management](./03-File-System-Management/): `dd` ile test dosyası oluşturma, XFS speculative preallocation keşfi, ve en büyük dosyaları bulma pipeline'ı.

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

---

## 🛠️ Ortam

- **Hypervisor:** VirtualBox 7.2.14
- **Guest OS:** Rocky Linux 10.2 (Red Quartz)
- **VM Kaynakları:** 4096 MB RAM, 4 vCPU, 20 GB Disk
- **Klavye Düzeni:** Türkçe (TR)

---

## 📚 Kaynak

Bu öğrenme yolculuğu [alifurkan-altuntas/devops-internship](https://github.com/alifurkan-altuntas/devops-internship) reposundaki müfredat sırası takip edilerek hazırlanmıştır.
