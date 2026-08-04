# ⏭️ Vagrant Otomasyonu — Atlandı

Bu faz, orijinal müfredatta Vagrant + VMware provider kullanılarak sanal makine kurulumunu otomatikleştirmeyi kapsıyor.

## Neden Atlandı?

Sanal makinemi (Rocky Linux 10.2) zaten **VirtualBox üzerinde manuel olarak** kurmuştum (bkz. `00-VM-Setup`). Vagrant'ın amacı da tam olarak bu süreci otomatikleştirmek olduğu için, manuel kurulumla aynı sonuca farklı bir yoldan ulaşılmış oldu.

## Vagrant Nedir? (Özet)

Vagrant, bir `Vagrantfile` (metin tabanlı yapılandırma dosyası) ve `vagrant up` komutu ile sanal makine kurulumunu otomatikleştiren bir araç:

- Bir "box" (hazır, önceden yapılandırılmış OS kalıbı) seçilir
- `Vagrantfile`'da RAM, CPU, port gibi ayarlar tanımlanır
- `vagrant up` ile VM otomatik kurulur — kurulum ekranından elle geçmeye gerek kalmaz

**DevOps'taki önemi:** "Infrastructure as Code" (IaC) mantığının basit bir örneği — altyapı elle değil, kod/config dosyasıyla tanımlanır, tutarlı ve tekrarlanabilir hale gelir.

## İleride Denemek İstersem

```bash
# Vagrant kurulumu (örnek)
sudo dnf install vagrant -y
vagrant init generic/rocky9
vagrant up --provider=virtualbox
```

Bu adımlar ileride ayrı bir pratik olarak denenebilir.

---

ℹ️ _Bu faz aktif olarak uygulanmadı, sadece kavramsal olarak belgelenmiştir._
