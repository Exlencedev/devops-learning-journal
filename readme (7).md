# 🔧 Git — Branch, Merge ve Gerçek Bir Push Çakışması

Bu belge, `git branch`, `git merge`, `git clone`, `git pull`/`git push`
komutlarını — ve bunları test ederken yaşanan biri beklenmedik, biri de
tam kaynak senaryonun kendisi olan iki gerçek debug sürecini kapsar.

---

## 1. Git Kimliğini Yapılandırma

VM'de daha önce hiç Git kimliği tanımlanmamıştı:

```bash
git config --global user.name "ege"
git config --global user.email "egeaylicioglu@hotmail.com"
```

---

## 2. Branch Alma ve İzolasyon (Canlı Test)

Mevcut yerel repo (`~/.git`, 14. fazdan kalma) üzerinde test edildi.

```bash
git checkout -b test-branch
echo "bu bir test dosyasi" > test-file.txt
git add test-file.txt
git commit -m "add test file on test-branch"
```

### 🐛 Not: Varsayılan Branch Adı `main` Değil, `master`

```bash
git checkout main
```

```
hata: yol belirteci 'main' git'in tanıdığı herhangi bir dosya ile eşleşmedi
```

Bu repo `git init` ile oluşturulduğunda varsayılan branch adı `master`
olmuştu (Git'in eski varsayılanı) — `main` diye bir branch hiç
oluşmamıştı. Doğru isimle devam edildi:

```bash
git checkout master
ls -la
```

`test-file.txt` bu listede **görünmedi** — iki branch'in birleştirilene
kadar bağımsız olduğu kanıtlandı.

### Fast-Forward Merge

```bash
git merge test-branch
```

```
Güncelleniyor: ab2fc76..7e7be8d
Fast-forward
test-file.txt | 1 +
1 file changed, 1 insertion(+)
```

`test-branch` oluşturulduğundan beri `master` hiç değişmediği için Git
gerçek bir merge commit oluşturmadı, sadece pointer'ı ilerletti. Doğrulama:

```bash
git log --oneline -3
```

```
7e7be8d (HEAD -> master, test-branch) add test file on test-branch
ab2fc76 Add disk usage check script
```

### Temizlik

```bash
git branch -d test-branch
git rm test-file.txt
git commit -m "clean up test branch demo file"
```

---

## 3. Gerçek Push Çakışması — Ayrı Bir Test Reposunda

Gerçek `devops-learning-journal` reposunu riske atmamak için GitHub'da
ayrı, sıfırdan bir test reposu (`Exlencedev/git-test-lab`, README'li)
oluşturuldu ve clone edildi:

```bash
git clone https://github.com/Exlencedev/git-test-lab.git
cd git-test-lab
```

**Yerel değişiklik (henüz push edilmedi):**

```bash
echo "local degisiklik" >> README.md
git add README.md
git commit -m "local update"
```

**Aynı anda, GitHub web arayüzünden** `README.md`'ye bağımsız bir satır
eklenip doğrudan `main`'e commit edildi — gerçek bir çatallanma senaryosu
kuruldu.

### 🐛 Hata & Çözüm #1: Push Reddedildi

```bash
git push origin main
```

```
! [rejected]        main -> main (fetch first)
hata: bazı başvurular '...' konumuna itilemedi
ipucu: Güncellemeler reddedildi: çünkü uzak konumda henüz yerelde
       sizde olmayan değişiklikler var.
ipucu: itmeden önce 'git pull' yapın.
```

### 🐛 Hata & Çözüm #2: Pull Stratejisi Belirsizliği

```bash
git pull origin main
```

```
ipucu: Iraksak dallarınız var ve onların nasıl uzlaştırılacağını
       belirtmeniz gerekiyor.
```

Modern Git (2.52.0), merge/rebase/fast-forward-only arasında hangi
stratejinin kullanılacağını artık otomatik varsaymıyor, açıkça
belirtilmesini istiyor:

```bash
git config pull.rebase false
git pull origin main
```

### 🐛 Hata & Çözüm #3: Gerçek Merge Çakışması

```
README.md kendiliğinden birleştiriliyor
ÇAKIŞMA (içerik): README.md içinde birleştirme çakışması
Otomatik birleştirme başarısız; çakışmaları çözün ve sonucu işleyin.
```

Hem yerel hem uzak, `README.md`'nin aynı satırlarını bağımsız olarak
değiştirdiği için Git otomatik birleştiremedi. Dosyanın çakışma
işaretli hâli:

```
<<<<<<< HEAD
# git-test-lablocal degisiklik
=======
# git-test-lab
"remote degisiklik"
>>>>>>> c9dca1bf...
```

**Elle çözüldü** — `nano` ile çakışma işaretleri temizlenip her iki
değişikliği de içeren temiz bir sürüm yazıldı:

```
# git-test-lab
local degisiklik
remote degisiklik
```

```bash
git add README.md
git commit -m "resolve merge conflict in README.md"
```

### 🐛 Hata & Çözüm #4: GitHub Token ile Kimlik Doğrulama

```bash
git push origin main
```

```
remote: Invalid username or token. Password authentication is not
        supported for Git operations.
```

GitHub, HTTPS üzerinden düz şifre kabul etmiyor — bir **Personal Access
Token (classic, `repo` yetkili)** gerekiyor. Panodan VM'e yapıştırma
(VirtualBox Shared Clipboard / Guest Additions eksikliği) çalışmadığı
için token, URL'ye gömülerek elle girildi:

```bash
git remote set-url origin https://Exlencedev:<TOKEN>@github.com/Exlencedev/git-test-lab.git
git push origin main
```

Push başarıyla tamamlandı. Doğrulama:

```bash
git log --oneline -5
git status
```

```
nothing to commit, working tree clean
```

---

## 4. Beklenmedik Ekstra Kriz: `/etc/fstab` Kaynaklı Emergency Mode

`git clone` denemesi sırasında VM'in interneti tamamen kesikti
(`Could not resolve host: github.com`). Teşhis zinciri:

1. `ping`/`dig` → `network unreachable`
2. `ip a` → `enp0s3` arayüzü **`state DOWN`**
3. `ip link set enp0s3 up` sonrası IPv6 geldi ama IPv4 gelmedi
4. `nmcli connection up` → `NMClient nesnesi oluşturulamadı`
5. `systemctl status NetworkManager` → **`inactive (dead)`**
6. `systemctl start NetworkManager` → **VM emergency mode'a düştü**

### 🐛 Kök Neden

```bash
journalctl -xb | grep -i "failed"
```

```
Dependency failed for dev-disk-by-uuid-d0294aa3...device
mnt-test-storage.mount: Job mnt-test-storage.mount/start failed with result 'dependency'
```

**10. fazdan (Depolama Yönetimi) kalma bir loop device mount girişi**
hâlâ `/etc/fstab`'daydı. Loop device'lar kalıcı değildir — VM yeniden
başlayınca (`losetup` ile elle yeniden bağlanmadıkça) kaybolur. Sistem
boot sırasında o UUID'ye sahip bir disk bulamayınca `local-fs.target`
tamamlanamadı, bu da tüm boot sürecini emergency mode'a düşürdü.

### ✅ Çözüm

```bash
cp /etc/fstab /etc/fstab.bak-before-fix
sed -i '/d0294aa3-7748-4525-b3d9-319a617b0f74/d' /etc/fstab
systemctl daemon-reload
systemctl default
```

Sistem normal boot'a döndü, `systemctl --failed` → `0 loaded units
listed`, `NetworkManager` → `active (running)`, internet tam olarak
geri geldi.

> ⚠️ **Ders:** Bir loop device üzerinde test amaçlı kurulan bir mount,
> `/etc/fstab`'dan kaldırılmazsa, VM'in bir sonraki yeniden başlatmasında
> **tüm sistemin boot etmesini engelleyebilir**. Geçici test mount'ları
> işi bitince fstab'dan mutlaka temizlenmeli.

---

## 5. Temel Çıkarımlar

- Reddedilen push genellikle uzak reponun yerelde olmayan commit'ler
  içerdiği anlamına gelir — önce `git pull`, sonra tekrar `push`.
- Her merge basit bir fast-forward değildir — her iki taraf bağımsız
  olarak değiştiyse, Git gerçek bir merge commit'e ihtiyaç duyar.
- Modern Git, pull stratejisini (`merge`/`rebase`/`ff-only`) artık
  otomatik varsaymıyor — `git config pull.rebase false/true` ile
  açıkça belirtilmesi gerekiyor.
- `git branch` (argümansız) yalnızca mevcut branch'leri listeler — bir
  tane oluşturmaz. Oluşturmak için isim gerekir: `git branch <isim>`.
- GitHub artık HTTPS üzerinden düz şifre kabul etmiyor, Personal Access
  Token gerekiyor.
- **Bir loop device mount'unu `/etc/fstab`'da kalıcı bırakmak, bir
  sonraki reboot'ta tüm sistemi emergency mode'a düşürebilir.**

---

## 📊 Komut Referansı

| Komut | Amacı |
|-------|-------|
| **`git clone <url>`** | Uzak bir reponun tam kopyasını yerel olarak indirir |
| **`git checkout -b <isim>`** | Tek adımda yeni branch oluşturur ve geçer |
| **`git merge <branch>`** | Belirtilen branch'i aktif branch'e birleştirir |
| **`git config pull.rebase false`** | Pull stratejisini "merge" olarak sabitler |
| **`git pull`** | Uzak değişiklikleri getirir ve yerel branch'e birleştirir |
| **`git remote set-url origin <url>`** | Remote URL'sini değiştirir (token gömme dahil) |
| **`journalctl -xb \| grep -i failed`** | Son boot loglarında başarısız birimleri filtreler |
| **`systemctl --failed`** | O an başarısız durumdaki tüm servisleri listeler |
| **`sed -i '/pattern/d' <dosya>`** | Belirli bir satırı dosyadan kalıcı olarak siler |

---

## 🧠 Quiz (Çoktan Seçmeli)

**S1:** `git push` komutu "rejected (fetch first)" hatasıyla reddedildiğinde bunun en yaygın sebebi nedir?

- A) Yerel commit mesajı çok uzun yazılmıştır
- B) Uzak repoda (GitHub'da), yerel repoda henüz bulunmayan commit'ler vardır ✅
- C) Git kimliği (`user.name`/`user.email`) tanımlanmamıştır
- D) `.gitignore` dosyası eksiktir

**S2:** Bir loop device üzerine kurulan test mount'u `/etc/fstab`'dan kaldırılmadan VM yeniden başlatılırsa ne olabilir?

- A) Hiçbir şey olmaz, loop device otomatik olarak yeniden oluşturulur
- B) Yalnızca o mount noktasına erişilemez, sistemin geri kalanı normal çalışır
- C) Sistem, olmayan bir UUID'yi bağlamaya çalışırken `local-fs.target`'ı tamamlayamaz ve emergency mode'a düşebilir ✅
- D) SELinux otomatik olarak mount girişini fstab'dan siler

---

ℹ️ _Tüm komutlar yerel Rocky Linux VM'inde (Rocky9-Test) test edilmiştir.
Bu fazda planlanan senaryonun (push çakışması, merge, token auth) yanı
sıra, beklenmedik bir gerçek kriz (10. fazdan kalma bir fstab girdisinin
VM'i emergency mode'a düşürmesi) debug edilip başarıyla çözülmüştür._
