# 🛡️ Fail2Ban Kurulum ve Yapılandırma Rehberi
### Mainnet Node Sunucuları için Güvenli Kurulum

> **Bu rehber kimler için?**
> Sunucusunda mainnet node çalıştıran ve Fail2Ban'ı hiç kurmamış, komut satırında tecrübesiz kullanıcılar için sıfırdan, adım adım hazırlanmıştır. Her adım açıklamalıdır. Herhangi bir komutu kör kör kopyalayıp yapıştırmadan önce ne yaptığını anlaman için açıklamalar eklenmiştir.

---

## ⚠️ EN ÖNEMLİ UYARI — OKUMADAN GEÇME

> **Fail2Ban yanlış yapılandırılırsa kendi IP adresini engelleyebilir ve sunucuna bir daha giremezsin.**
>
> Bu rehberde bu durumu önlemek için alınması gereken tüm tedbirler adım adım anlatılmıştır. Adımları sırayla uygula, hiçbirini atlama.

---

## 📋 İçindekiler

1. [Fail2Ban Nedir?](#1-fail2ban-nedir)
2. [Kuruluma Başlamadan Önce](#2-kuruluma-başlamadan-önce)
3. [Fail2Ban Kurulumu](#3-fail2ban-kurulumu)
4. [Temel Yapılandırma](#4-temel-yapılandırma)
5. [SSH Koruması](#5-ssh-koruması)
6. [Beyaz Liste — Kendi IP Adresini Koru](#6-beyaz-liste--kendi-ip-adresini-koru)
7. [Dinamik IP ve Farklı Cihazlardan Erişim](#7-dinamik-ip-ve-farklı-cihazlardan-erişim)
8. [Fail2Ban'ı Başlatma ve Test Etme](#8-fail2banı-başlatma-ve-test-etme)
9. [Günlük Kullanım Komutları](#9-günlük-kullanım-komutları)
10. [Sorun Giderme](#10-sorun-giderme)
11. [Gelişmiş Ayarlar (İsteğe Bağlı)](#11-gelişmiş-ayarlar-isteğe-bağlı)
12. [Ubuntu 24.04 (Noble) — Önemli Farklar](#12-ubuntu-2404-noble--önemli-farklar)

---

## 1. Fail2Ban Nedir?

**Fail2Ban**, sunucuna sürekli bağlanmaya çalışan kötü niyetli kişileri otomatik olarak engelleyen bir güvenlik yazılımıdır.

**Nasıl çalışır?**

```
Saldırgan → Yanlış şifre dener → Fail2Ban fark eder → IP'yi engeller
```

Örneğin biri SSH üzerinden şifreni kırmaya çalışıyorsa ve 5 kez üst üste yanlış giriş yaparsa, Fail2Ban o kişinin IP adresini otomatik olarak güvenlik duvarına ekler ve bağlantısını keser.

**Mainnet node çalıştıranlar için neden önemli?**
- Node'unun port'larına yetkisiz erişimi engeller
- Brute-force (kaba kuvvet) saldırılarını durdurur
- Sunucu kaynaklarını gereksiz trafikten korur

---

## 2. Kuruluma Başlamadan Önce

### 2.1 Kendi IP Adresini Öğren

> **Bu adım kritiktir.** Kendi IP adresini bilmeden Fail2Ban konfigürasyonuna başlama.

**Kendi bilgisayarında** (sunucuda değil) bir tarayıcı aç ve şu siteye git:

```
https://www.whatismyip.com
```

Ya da terminalde şu komutu çalıştır:

```bash
curl ifconfig.me
```

Çıktı şöyle görünecek:
```
123.45.67.89
```

Bu sayıyı not al. Bir sonraki adımlarda kullanacaksın.

---

### 2.2 Sunucuna Bağlan

Sunucuna SSH ile bağlan:

```bash
ssh kullanici_adin@sunucu_ip_adresin
```

Örnek:
```bash
ssh root@192.168.1.100
```

---

### 2.3 Sistemi Güncelle

Fail2Ban kurmadan önce sistemi güncelle. Bu adım, kurulumun sorunsuz gitmesi için önemlidir.

```bash
sudo apt update && sudo apt upgrade -y
```

> **Ne yapar bu komut?**
> - `apt update` → Yüklenebilecek paketlerin listesini günceller
> - `apt upgrade -y` → Tüm güncel olmayan paketleri yükseltir
> - `-y` → "Evet mi devam edeyim?" sorularına otomatik "Evet" der

Tamamlanması birkaç dakika sürebilir.

---

## 3. Fail2Ban Kurulumu

### 3.1 Fail2Ban'ı Kur

```bash
sudo apt install fail2ban -y
```

> **Ne yapar?** Fail2Ban yazılımını indirip kurar.

### 3.2 Kurulumu Doğrula

```bash
fail2ban-client --version
```

Buna benzer bir çıktı göreceksin:
```
Fail2Ban v1.0.2
```

Versiyon numarası görünüyorsa kurulum başarılıdır.

---

## 4. Temel Yapılandırma

### 4.1 Yapılandırma Dosyaları Hakkında

Fail2Ban'ın iki tür yapılandırma dosyası vardır:

| Dosya | Açıklama |
|-------|----------|
| `/etc/fail2ban/jail.conf` | Orijinal ayar dosyası — **ASLA düzenleme** |
| `/etc/fail2ban/jail.local` | Senin özel ayar dosyan — **Bu dosyayı düzenle** |

> **Neden orijinal dosyayı düzenlemiyoruz?**
> Fail2Ban güncellendiğinde `jail.conf` dosyası üzerine yazılır ve tüm ayarların silinir. `jail.local` dosyası ise güncellemeye karşı korumalıdır.

### 4.2 Yapılandırma Dosyasını Oluştur

```bash
sudo nano /etc/fail2ban/jail.local
```

> **Ne yapar?** `nano` adlı metin düzenleyiciyi açar. Dosya yoksa otomatik oluşturur.

Açılan boş dosyaya aşağıdaki içeriği yapıştır:

```ini
[DEFAULT]
# Kaç dakika boyunca log dosyası geriye doğru taranacak
findtime = 10m

# Kaç başarısız denemede IP engellenecek
maxretry = 5

# IP kaç saniye boyunca engellenecek (3600 = 1 saat)
bantime = 3600

# Beyaz listedeki IP'ler asla engellenmez
# KENDİ IP ADRESİNİ BURAYA YAZMAYI UNUTMA!
ignoreip = 127.0.0.1/8 ::1

# Engelleme yöntemi (iptables kullanıyoruz)
banaction = iptables-multiport
```

> **Ayarların anlamı:**
> - `findtime = 10m` → Son 10 dakikaya bak
> - `maxretry = 5` → 5 başarısız deneme sonrası engelle
> - `bantime = 3600` → 1 saat boyunca engelle (saniye cinsinden)
> - `ignoreip` → Bu IP'ler hiçbir zaman engellenmez

### 4.3 Dosyayı Kaydet

`nano` düzenleyicisinde kaydetmek için:

1. `Ctrl + O` → Kaydet (Write Out)
2. `Enter` → Onay
3. `Ctrl + X` → Çıkış

---

## 5. SSH Koruması

### 5.1 SSH Jail Yapılandırması

`jail.local` dosyasına SSH koruması ekleyeceğiz. Dosyayı tekrar aç:

```bash
sudo nano /etc/fail2ban/jail.local
```

Dosyanın **sonuna** şunları ekle:

```ini
[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 86400
findtime = 10m
```

> **Bu ayarların anlamı:**
> - `enabled = true` → Bu kuralı aktif et
> - `port = ssh` → SSH portunu izle (varsayılan: 22)
> - `filter = sshd` → SSH için özel filtreyi kullan
> - `logpath` → Başarısız girişlerin yazıldığı log dosyası
> - `maxretry = 3` → 3 başarısız denemede engelle
> - `bantime = 86400` → 24 saat boyunca engelle

> **⚠️ SSH portunu değiştirdiysen:** `port = ssh` yerine `port = 2222` şeklinde kendi portunu yaz.

---

## 6. Beyaz Liste — Kendi IP Adresini Koru

> **Bu adım en kritik adımdır. Atlama!**

Kendi IP adresini beyaz listeye eklemezsen, yanlışlıkla kendin de engellenebilirsin.

### 6.1 IP Adresini Ekle

`jail.local` dosyasındaki `[DEFAULT]` bölümündeki `ignoreip` satırını bul ve kendi IP adresini ekle:

```ini
ignoreip = 127.0.0.1/8 ::1 BURAYA_KENDI_IP_ADRESINI_YAZ
```

Örnek (IP adresin 123.45.67.89 ise):
```ini
ignoreip = 127.0.0.1/8 ::1 123.45.67.89
```

Birden fazla IP eklemek istersen aralarına boşluk koy:
```ini
ignoreip = 127.0.0.1/8 ::1 123.45.67.89 98.76.54.32
```

### 6.2 IP Adresin Değişiyorsa Ne Yapmalısın?

Eğer internet servis sağlayıcın sana **dinamik IP** veriyorsa (her bağlandığında IP değişiyor), şu seçenekleri değerlendir:

1. **Statik IP satın al** → İnternet sağlayıcından sabit bir IP adresi al
2. **VPN kullan** → Sabit IP'li bir VPN servisi kullan ve o IP'yi beyaz listeye ekle
3. **IP bloğu ekle** → Örneğin hep Türkiye'den bağlanıyorsan `/24` bloğunu ekleyebilirsin (daha az güvenli)

---

## 7. Dinamik IP ve Farklı Cihazlardan Erişim

> **Bu bölüm senin için:** Ev interneti dinamik IP veriyorsa, telefondan veya farklı ağlardan sunucuna bağlanman gerekiyorsa bu bölümü dikkatle oku.

### Sorunun Özeti

```
Ev bilgisayarı   → IP her an değişebilir
Telefon (mobil)  → Farklı bir IP adresi
Kafeden bağlanma → Yine farklı bir IP

Fail2Ban beyaz listeye aldığın IP'yi tanır.
Peki IP değişirse? → Engelleme riski!
```

**Çözüm:** IP adresinden bağımsız bir kimlik doğrulama yöntemi kullanmak. Bunun en güvenli ve pratik yolu **SSH Anahtar (Key) Doğrulaması**'dır.

---

### 7.1 SSH Anahtar Doğrulaması Nedir?

Normal SSH girişinde şifre kullanırsın. Şifre tahmin edilebilir, bu yüzden saldırganlar defalarca dener.

SSH key ile girişte ise iki parça vardır:

```
Private Key (Özel Anahtar) → Sende kalır, kimseyle paylaşma
Public Key  (Açık Anahtar) → Sunucuya yüklenir
```

Sunucuya bağlanmaya çalıştığında sistem sorar: "Bu public key'in sahibi misin?" Bilgisayarın private key ile "Evet benim" der. Şifre gerekmez, tahmin edilecek bir şey yoktur.

**Fail2Ban açısından önemi:** Şifre denemesi olmadığı için Fail2Ban devreye girmez. IP adresin ne olursa olsun, anahtarın varsa girersin.

---

### 7.2 SSH Key Oluşturma — Bilgisayar (Windows / Mac / Linux)

> Bu adımları **kendi bilgisayarında** yap, sunucuda değil.

**Mac ve Linux için:**

```bash
ssh-keygen -t ed25519 -C "sunucu-erisim"
```

**Windows için (PowerShell veya CMD):**

```bash
ssh-keygen -t ed25519 -C "sunucu-erisim"
```

> Windows 10/11'de OpenSSH varsayılan olarak yüklü gelir. Çalışmazsa Ayarlar → Uygulamalar → İsteğe Bağlı Özellikler → OpenSSH İstemcisi'ni kur.

Komutu çalıştırdıktan sonra şu sorular gelecek:

```
Enter file in which to save the key: (Enter'a bas, varsayılan konuma kaydeder)
Enter passphrase: (İsteğe bağlı — ek güvenlik için şifre ekleyebilirsin)
Enter same passphrase again: (Aynı şifreyi tekrar gir)
```

Oluşturulan iki dosya:

| Dosya | Açıklama |
|-------|----------|
| `~/.ssh/id_ed25519` | **Private key** — Asla paylaşma, yedekle |
| `~/.ssh/id_ed25519.pub` | **Public key** — Sunucuya yüklenecek olan |

---

### 7.3 Public Key'i Sunucuya Yükle

**Mac / Linux'tan:**

```bash
ssh-copy-id kullanici_adin@sunucu_ip_adresin
```

Bu komut yoksa manuel olarak yapabilirsin:

```bash
cat ~/.ssh/id_ed25519.pub | ssh kullanici_adin@sunucu_ip "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

**Windows'tan (PowerShell):**

```powershell
type $env:USERPROFILE\.ssh\id_ed25519.pub | ssh kullanici_adin@sunucu_ip "mkdir -p ~/.ssh && cat >> ~/.ssh/authorized_keys"
```

---

### 7.4 Sunucuda Dosya İzinlerini Ayarla

Sunucuna bağlan ve şu komutları çalıştır:

```bash
chmod 700 ~/.ssh
chmod 600 ~/.ssh/authorized_keys
```

> **Ne yapar?** SSH sadece doğru izinlere sahip dosyaları kabul eder. Bu komutlar izinleri doğru şekilde ayarlar.

---

### 7.5 Key ile Bağlantıyı Test Et

> **⚠️ Eski SSH bağlantını kapatma!** Yeni bir terminal aç ve test et:

```bash
ssh kullanici_adin@sunucu_ip_adresin
```

Şifre sormadan giriyorsan key doğrulaması çalışıyor demektir.

---

### 7.6 Şifre ile Girişi Kapat (Opsiyonel ama Önerilen)

Key ile girişin çalıştığını doğruladıktan sonra, şifre ile girişi tamamen kapatabilirsin. Bu en güvenli yapılandırmadır.

Sunucuda SSH ayar dosyasını aç:

```bash
sudo nano /etc/ssh/sshd_config
```

Bu satırı bul ve değiştir:

```
# Bul:
#PasswordAuthentication yes

# Bunu yap (başındaki # işaretini kaldır ve yes'i no yap):
PasswordAuthentication no
```

Kaydet ve SSH servisini yeniden başlat:

```bash
sudo systemctl restart sshd
```

> **Bu adımdan sonra şifre ile giriş tamamen kapanır. Key olmadan bağlanamazsın.**

---

### 7.7 Telefona SSH Key Kurulumu

Telefonundan da aynı key ile bağlanabilirsin. Bunun için telefonuna bir SSH uygulaması kurman gerekir.

**Önerilen uygulamalar:**

| Platform | Uygulama | Ücretsiz mi? |
|----------|----------|-------------|
| iOS | [Termius](https://termius.com) | Evet (temel özellikler) |
| iOS | [Prompt 3](https://apps.apple.com/app/prompt-3/id1594420480) | Ücretli |
| Android | [Termius](https://termius.com) | Evet |
| Android | [JuiceSSH](https://juicessh.com) | Evet |

**Telefona key aktarma adımları (Termius örneği):**

1. Bilgisayarındaki `id_ed25519` dosyasını (private key) güvenli bir şekilde telefona aktar
   - AirDrop (iPhone) veya USB kablo kullanabilirsin
   - **WhatsApp veya e-posta ile gönderme** — güvenli değil
2. Termius → Keychain → Add Key → Import from file
3. Key'i import et
4. Yeni host eklerken bu key'i seç

---

### 7.8 Hangi Yöntemi Seçmelisin?

| Durum | Önerilen Çözüm |
|-------|---------------|
| Her yerden (ev, telefon, kafe) bağlanıyorum | **SSH Key** — IP'den bağımsız |
| Sadece evden bağlanıyorum, IP sabit | Statik IP + beyaz liste |
| Sadece evden bağlanıyorum, IP dinamik | DDNS veya SSH Key |
| VPN kullanıyorum | VPN IP'sini beyaz listeye ekle |

---

### 7.9 DDNS Alternatifi (SSH Key Kullanmak İstemeyenler İçin)

DDNS (Dinamik DNS), değişen IP adresini sabit bir alan adına bağlar.

**Ücretsiz DDNS servisleri:** [DuckDNS](https://www.duckdns.org), [No-IP](https://www.noip.com)

**DuckDNS kurulumu (örnek):**

1. [duckdns.org](https://www.duckdns.org) adresine git ve ücretsiz kayıt ol
2. Bir subdomain oluştur: `senin-adin.duckdns.org`
3. **Kendi bilgisayarına** DuckDNS güncelleme scriptini kur:

```bash
# Her 5 dakikada bir DuckDNS'e güncel IP'ni bildirir
echo "*/5 * * * * curl -s 'https://www.duckdns.org/update?domains=senin-adin&token=TOKEN_BURAYA&ip='" | crontab -
```

4. `jail.local` dosyasında hostname kullan:

```ini
ignoreip = 127.0.0.1/8 ::1 senin-adin.duckdns.org
```

> **Önemli not:** Fail2Ban, hostname'i yalnızca başlangıçta çözümler. IP değişirse Fail2Ban'ı yeniden başlatman gerekir: `sudo systemctl restart fail2ban`

---

### 7.10 VPN Alternatifi

Sabit çıkış IP'sine sahip bir VPN servisi kullanıyorsan, o IP'yi beyaz listeye eklemen yeterlidir.

```ini
ignoreip = 127.0.0.1/8 ::1 VPN_CIKIS_IP_ADRESI
```

VPN çıkış IP'ni öğrenmek için VPN'e bağlıyken ziyaret et:

```
https://ifconfig.me
```

---

## 8. Fail2Ban'ı Başlatma ve Test Etme

### 7.1 Servisi Başlat

```bash
sudo systemctl enable fail2ban
sudo systemctl start fail2ban
```

> - `enable` → Sunucu her açıldığında Fail2Ban otomatik başlasın
> - `start` → Şu an hemen başlat

### 7.2 Servis Durumunu Kontrol Et

```bash
sudo systemctl status fail2ban
```

Çıktıda şunu görmelisin:
```
● fail2ban.service - Fail2Ban Service
   Active: active (running) ...
```

`active (running)` yazıyorsa her şey yolunda.

### 7.3 Jail'lerin Aktif Olduğunu Kontrol Et

```bash
sudo fail2ban-client status
```

Çıktı şöyle görünmeli:
```
Status
|- Number of jail: 1
`- Jail list: sshd
```

SSH jail'i aktif görünüyorsa kurulum başarılı.

### 7.4 SSH Jail Detaylarını Gör

```bash
sudo fail2ban-client status sshd
```

Çıktı:
```
Status for the jail: sshd
|- Filter
|  |- Currently failed: 0
|  |- Total failed: 0
|  `- File list: /var/log/auth.log
`- Actions
   |- Currently banned: 0
   |- Total banned: 0
   `- Banned IP list:
```

---

## 9. Günlük Kullanım Komutları

### Engellenen IP'leri Listele

```bash
sudo fail2ban-client status sshd
```

### Bir IP'nin Engelini Kaldır

Kendi IP'ni yanlışlıkla engellediysen:

```bash
sudo fail2ban-client set sshd unbanip IP_ADRESI
```

Örnek:
```bash
sudo fail2ban-client set sshd unbanip 123.45.67.89
```

### Fail2Ban Loglarını İzle

Gerçek zamanlı olarak neler olduğunu görmek için:

```bash
sudo tail -f /var/log/fail2ban.log
```

Çıkmak için `Ctrl + C`.

### Fail2Ban'ı Yeniden Başlat

Ayarları değiştirdikten sonra yeniden başlatmak için:

```bash
sudo systemctl restart fail2ban
```

### Fail2Ban'ı Durdur / Başlat

```bash
sudo systemctl stop fail2ban
sudo systemctl start fail2ban
```

---

## 10. Sorun Giderme

### 🔴 Sunucuya Giremiyorum — Kendimi Engelledim!

**Panik yapma.** Şu seçenekleri dene:

**Seçenek 1: VPS Sağlayıcısının Konsolunu Kullan**
Hetzner, DigitalOcean, OVH vb. sağlayıcıların web panelinde "Console" veya "VNC" seçeneği vardır. Buradan doğrudan sunucuya bağlanabilirsin. Bağlandıktan sonra:

```bash
sudo fail2ban-client set sshd unbanip KENDI_IP_ADRESIN
```

**Seçenek 2: Farklı Bir IP'den Bağlan**
Mobil veri kullanarak (telefon hotspot'u) veya farklı bir ağdan bağlanmayı dene.

**Seçenek 3: Fail2Ban'ı Geçici Olarak Durdur (Konsol üzerinden)**
```bash
sudo systemctl stop fail2ban
```
Bu komut tüm engellemeleri kaldırır.

---

### 🟡 Fail2Ban Başlamıyor

Hata mesajını görmek için:

```bash
sudo journalctl -u fail2ban -n 50
```

Yapılandırma dosyasında yazım hatası olabilir. Dosyayı kontrol et:

```bash
sudo fail2ban-client -t
```

`OK: configuration test is successful` görüyorsan sorun yok.

---

### 🟡 Log Dosyası Bulunamıyor Hatası

`logpath` olarak belirttiğin dosya sistemde olmayabilir. Ubuntu/Debian'da kontrol et:

```bash
ls /var/log/auth.log
```

Dosya yoksa:
```bash
sudo touch /var/log/auth.log
```

---

### 🟡 Ayarları Değiştirdikten Sonra Geçerli Olmadı

Her yapılandırma değişikliğinden sonra Fail2Ban'ı yeniden başlatman gerekir:

```bash
sudo systemctl restart fail2ban
```

---

## 11. Gelişmiş Ayarlar (İsteğe Bağlı)

### 11.1 Daha Uzun Engelleme Süresi

Tekrar eden saldırganlar için kalıcı engelleme:

```ini
[sshd]
bantime = -1
```

> `-1` değeri IP'yi kalıcı olarak engeller. Dikkatli kullan.

---

### 11.2 Progressif (Artan) Engelleme Süresi

Her engellemede süreyi 2 katına çıkar:

```ini
[DEFAULT]
bantime.increment = true
bantime.multiplier = 2
bantime.maxtime = 1w
```

---

### 11.3 E-posta Bildirimi

Bir IP engellendiğinde e-posta almak için önce `sendmail` kur:

```bash
sudo apt install sendmail -y
```

Sonra `jail.local` dosyasına ekle:

```ini
[DEFAULT]
destemail = senin@email.com
sender = fail2ban@sunucu
mta = sendmail
action = %(action_mwl)s
```

---

### 11.4 Tüm `jail.local` Dosyasının Tam Hali

Referans olması için eksiksiz bir yapılandırma örneği:

```ini
[DEFAULT]
findtime = 10m
maxretry = 5
bantime = 3600
ignoreip = 127.0.0.1/8 ::1 KENDI_IP_ADRESIN
banaction = iptables-multiport
bantime.increment = true
bantime.multiplier = 2

[sshd]
enabled = true
port = ssh
filter = sshd
logpath = /var/log/auth.log
maxretry = 3
bantime = 86400
findtime = 10m
```

---

## 📊 Hızlı Başvuru Tablosu

| Komut | Ne Yapar |
|-------|----------|
| `sudo systemctl status fail2ban` | Servis durumunu göster |
| `sudo systemctl restart fail2ban` | Servisi yeniden başlat |
| `sudo fail2ban-client status` | Aktif jail'leri listele |
| `sudo fail2ban-client status sshd` | SSH jail detaylarını göster |
| `sudo fail2ban-client set sshd unbanip X.X.X.X` | IP engelini kaldır |
| `sudo tail -f /var/log/fail2ban.log` | Logları canlı izle |
| `sudo fail2ban-client -t` | Yapılandırmayı test et |

---

## ✅ Kurulum Kontrol Listesi

Kurulum tamamlandıktan sonra şunları doğrula:

- [ ] Kendi IP adresimi `ignoreip` listesine ekledim
- [ ] `fail2ban-client -t` ile yapılandırmayı test ettim
- [ ] `systemctl status fail2ban` ile servisin çalıştığını doğruladım
- [ ] `fail2ban-client status sshd` ile SSH jail'inin aktif olduğunu gördüm
- [ ] Farklı bir terminalden SSH bağlantısı hâlâ çalışıyor (eski terminali kapatmadım)
- [ ] SSH key oluşturdum ve sunucuya yükledim
- [ ] Key ile bağlantıyı test ettim, şifresiz giriyor
- [ ] Telefona SSH uygulaması kurdum ve key'i aktardım
- [ ] Telefondan test bağlantısı yaptım

---

## 12. Ubuntu 24.04 (Noble) — Önemli Farklar

> Ubuntu 22.04 kullananlar bu bölümü atlayabilir. Ubuntu 24.04 (Noble Numbat) kullananlar için rehberin bazı adımları farklı çalışır.

---

### 12.1 SSH Servis Adı Değişti

Ubuntu 24.04'te SSH servisi `sshd` değil `ssh` olarak adlandırılır.

| İşlem | Ubuntu 22.04 | Ubuntu 24.04 |
|-------|-------------|-------------|
| Yeniden başlat | `sudo systemctl restart sshd` | `sudo systemctl restart ssh` |
| Durdur | `sudo systemctl stop sshd` | `sudo systemctl stop ssh` |
| Başlat | `sudo systemctl start sshd` | `sudo systemctl start ssh` |
| Durum | `sudo systemctl status sshd` | `sudo systemctl status ssh` |

> `sshd.service not found` hatası alıyorsan Ubuntu 24.04 kullanıyorsun demektir. `ssh` komutunu kullan.

---

### 12.2 Log Dosyası Değişti

Ubuntu 24.04, geleneksel `/var/log/auth.log` yerine varsayılan olarak **systemd journal** kullanır. Bu dosya bazen oluşmaz veya boş gelir.

**Kontrol et:**

```bash
ls -la /var/log/auth.log
```

**Dosya yoksa** iki seçeneğin var:

**Seçenek A — rsyslog kur (auth.log'u geri getir, önerilen):**

```bash
sudo apt install rsyslog -y
sudo systemctl enable rsyslog
sudo systemctl start rsyslog
```

Kurulumdan sonra `/var/log/auth.log` otomatik oluşur. Fail2Ban'ı yeniden başlat:

```bash
sudo systemctl restart fail2ban
```

**Seçenek B — journal'ı logpath olarak kullan:**

`jail.local` dosyasında `[sshd]` bölümünü şöyle değiştir:

```ini
[sshd]
enabled = true
port = ssh
filter = sshd
backend = systemd
maxretry = 3
bantime = 86400
findtime = 10m
```

> `logpath` satırını kaldır, yerine `backend = systemd` ekle. Fail2Ban doğrudan systemd journal'ını okur.

---

### 12.3 PubkeyAuthentication Varsayılan Olarak Kapalı Geliyor

> Ubuntu 22.04'te bu adım **gerekmiyordu** çünkü `PubkeyAuthentication yes` zaten aktifti. Ubuntu 24.04'te aynı satır yorum olarak (`#` ile) geliyor, yani devre dışı. SSH key ile bağlanamıyorsan ilk bakacağın yer burasıdır.

**Neden farklı?** Ubuntu 24.04, `sshd_config` dosyasını daha kısıtlayıcı varsayılanlarla sunuyor. Yorum satırı olan ayarlar devre dışı sayılır.

**Kontrol et:**

```bash
grep PubkeyAuthentication /etc/ssh/sshd_config
```

Eğer çıktı şöyleyse (`#` ile başlıyorsa) → **devre dışı:**

```
#PubkeyAuthentication yes
```

**Düzeltmek için:**

```bash
sudo nano /etc/ssh/sshd_config
```

`#PubkeyAuthentication yes` satırını bul, başındaki `#` işaretini ve yanındaki boşluğu sil:

```
# Önce (devre dışı):
#PubkeyAuthentication yes

# Sonra (aktif):
PubkeyAuthentication yes
```

Kaydet (`Ctrl+O` → `Enter` → `Ctrl+X`) ve SSH'yi yeniden başlat:

```bash
sudo systemctl restart ssh
```

---

### 12.5 sshd_config Dosyası Bölündü

Ubuntu 24.04'te SSH yapılandırması tek dosya yerine klasör yapısına taşındı.

```
/etc/ssh/sshd_config          → Ana dosya (dokunma)
/etc/ssh/sshd_config.d/       → Özel ayarlar buraya
```

`PasswordAuthentication no` ayarını yaparken (Bölüm 7.6) ana dosyayı düzenleme. Bunun yerine yeni bir dosya oluştur:

```bash
sudo nano /etc/ssh/sshd_config.d/99-custom.conf
```

İçine şunu yaz:

```
PasswordAuthentication no
```

Kaydet ve SSH'yi yeniden başlat:

```bash
sudo systemctl restart ssh
```

> Bu yöntem Ubuntu güncellemelerinde ayarlarının üzerine yazılmasını engeller.

---

### 12.6 Ubuntu 24.04 için Tam jail.local Örneği

```ini
[DEFAULT]
findtime = 10m
maxretry = 5
bantime = 3600
ignoreip = 127.0.0.1/8 ::1 KENDI_IP_ADRESIN
banaction = iptables-multiport
bantime.increment = true
bantime.multiplier = 2

[sshd]
enabled = true
port = ssh
filter = sshd
backend = systemd
maxretry = 3
bantime = 86400
findtime = 10m
```

---

### 12.7 Ubuntu 24.04 Kontrol Listesi

- [ ] `grep PubkeyAuthentication /etc/ssh/sshd_config` ile kontrol ettim, `#` yoksa aktif
- [ ] `PubkeyAuthentication yes` satırındaki `#` işaretini kaldırdım
- [ ] `sudo systemctl restart ssh` ile (sshd değil) SSH'yi yeniden başlattım
- [ ] `/var/log/auth.log` var mı kontrol ettim, yoksa `rsyslog` kurdum
- [ ] Şifre kapatmayı `/etc/ssh/sshd_config.d/99-custom.conf` dosyasına ekledim
- [ ] `sudo systemctl status ssh` ile SSH'nin çalıştığını doğruladım

---

## 🔗 Kaynaklar

- [Fail2Ban Resmi Dokümantasyon](https://www.fail2ban.org/wiki/index.php/Main_Page)
- [Fail2Ban GitHub Sayfası](https://github.com/fail2ban/fail2ban)

---

> **Katkı ve geri bildirim:** Herhangi bir adımda sorun yaşarsan bir Issue açabilirsin.
