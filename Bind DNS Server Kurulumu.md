# Debian 10/11 Bind DNS Server Kurulumu

## Başlangıç

Bu yazımızda Debian 10/11 tabanlı Pardus 19.5 ve 21 sunucularda **bind9 DNS Server** kurulumunu ve ayarlamalarını göstereceğiz. Başlamadan önce değinmemiz gereken birkaç nokta var.

- Yazıdaki IP ve makine adresleri tamamen sizlere konuyu kavratmak adına oluşturulmuştur. Lütfen kurulum ve kullanım boyunca kendi adreslerinizi kullanmayı ihmal etmeyiniz.
- Resimlerin bazı kısımlarında üstü kapatılan bölümler kullanılan makinelerin IP bloklarını gizlemek adına kapatılmıştır. Beyaz renkle kapatılan bölümlerin büyük bir kısmını aşağıda da belirtileceği üzere `14.8.18` (reverse zone kısımlar için `18.8.14`) olarak kabul edebilirsiniz.
- Kurulumda kullanılan konfigürasyon dosyalarını [GitHub reposunda](https://github.com/fthrslntgy/debianBindDNS) bulabilirsiniz.

## Kurulum

Öncelikle olası bir hatadan kaçınmak için sistemimizi güncelleyelim.

```bash
sudo apt update -y
sudo apt upgrade -y
```

Ardından ilgili paketleri yükleyerek sistemimize **bind9** kurulumu yapalım.

```shell
sudo apt install bind9 bind9utils
```

Kurulum tamamlandıktan sonra BIND'ı **IPv4** moduna ayarlayacağız. Bunun için bir text editor ile (tercihen nano veya vi) `/etc/default/bind9` dosyasının içeriğini aşağıdaki hale getiriyoruz.

```bash
sudo nano /etc/default/bind9
```

```bash
# run resolvconf?
RESOLVCONF=no

# startup options for the server
OPTIONS="-4 -u bind"
```



## DNS Server Yapılandırma

**NOT: Yapılandırma sırasında içeriği değiştirilen dosyalardaki syntax çok önemlidir. En ufak noktalı virgül, space veya tab hatası hatalara sebep olabilmektedir. Bu sebeple lütfen siz de komutları ve dosyaların içeriğini örneklerdeki gibi uygulayınız.**

Kullanacağımız iki makinenin bilgilerini aşağıdaki tabloda görebilirsiniz. Birçok konfigürasyon yapacağımızdan ve bu konfigürasyonlarda makinelerin hostname, IP gibi bilgilerini kullanacağımızdan konfigürasyon dosyalarının hangi değişkenine ne yazmamız gerektini aşağıdaki tabloya bakarak anlayabiliriz.

| Rol                     | Hostname      | IP              | Oluşacak FQND*               |
| ----------------------- | ------------- | --------------- | ---------------------------- |
| DNS Server (nameserver) | testdnsserver | **14.8.18.121** | testdnsserver.testzone.local |
| DNS Client              | testdnsclient | **14.8.18.120** | testdnsclient.testzone.local |

**FQDN = Fully-Qualified Domain Name*

**Oluşturacağımız DNS bölgesinin (zone) ismi tamamen bize bağlıdır. Lokalde denemeler yapacağımız için kendi isteğimize bağlı olarak "testzone.local" adında bir zone oluşturacağız.*

- Öncelikle konfigürasyonlarımızı yapacağımız dizine gidelim (bundan sonraki işlemleri bu klasör içerisinden yapacağız yani yazacağımız komutlar da buradaki pathe göre olacak). Daha sonra ilk dosyamızı düzenlemeye başlıyoruz. **named.conf.options** dosyasında güvenilir ağ üyelerini (host) ve forwarderları belirleyeceğiz.

```bash
cd /etc/bind/
sudo nano named.conf.options
```

```bash
### DOSYA ICERIGI ###
acl "trusted" {
        14.8.18.121; # dns serverim
        14.8.18.120; # dns clientim
};

options {
        directory "/var/cache/bind";
        recursion yes;
        allow-recursion { trusted; }; # yukarida trusted olarak tanimladigim hostlara recursive query izni veriyorum
        listen-on { 14.8.18.121; }; # nameserver ozel IP adresi

        forwarders {
                8.8.8.8; # google dns
                8.8.4.4;
        };

        dnssec-validation auto;
        listen-on-v6 { any; };
};
```

*Recursive modda clientlardan domain talebi geldiğinde ve istenen domain nameserverda bulunamadığında nameserver diğer nameserverlara talebi iletir, bulunamadığında ise öneride bulunur. Döngü halinde çalışır. Iterative modda ise nameserverda istenen domain varsa size IP'yi verir, yoksa bilmediğini söyler. Profosyonel DNS Serverlarında recursive modun kapatılması önerilir çünkü kötü amaçlı aşırı sayıda yapılan request saldırıları ile DNS Server meşgul edileceğinden kendi işlevini de yerine getiremez hale gelir. Fakat biz lokalde çalıştığımızdan açık bırakabiliriz.*

* Ardından alan yani zoneları belirleyeceğiz. **named.conf.local** dosyası boş durumda olabilir. Biz örnek bir dosya oluşturacağız. İsterseniz kopyala yapıştır yaparak, isterseniz de **named.conf.default-zones** dosyasından şablonu kopyalayarak gerekli konfigürasyonlarla kendi dosyanızı oluşturabilirsiniz. 

  - Yukarıda DNS alan adımızın **testzone.local** olacağından bahsetmiştik. Buradaki ayarlarımızı da bu alan adına göre yapıyoruz. Oluşturacağımız db dosyasının adı da **db.testzone.local** oluyor. *(/etc/bind dizininin altında zones/ klasörü oluşturacağız ve burada zoneların db dosyalarını düzenleyeceğiz. Şimdilik yalnızca file pathi veriyoruz.)*

  * Reverse Lookup Zone adımızı **18.8.14.in-addr.arpa**, db dosyamızın adını ise **db.14.8.18** olarak verdik. Örneğin IP bloğumuz *192.168.1.0* olsaydı zone adımız **1.168.192.arpa**, db dosyamızın adı ise **db.192.168.1** olacaktı.

```bash
sudo nano named.conf.local
```

```bash
### DOSYA ICERIGI ###
zone "testzone.local" {
        type master;
        file "/etc/bind/zones/db.testzone.local";
};

zone "18.8.14.in-addr.arpa" {
        type master;
        file "/etc/bind/zones/db.14.8.18";
};
```



## Forward Lookup Zone

İlk olarak zonelar için ayrı bir klasör oluşturuyoruz. Daha sonra ise **db.local** dosyasını (oluşturacağımız dosyaya syntax olarak benzediğinden bu şablon üzerinde oynama yapmak daha kolay olacaktır) Forward Lookup Zone adımızı vererek buraya kopyalayalım. Bu isim **named.conf.local** dosyasında verdiğimizle aynı, yani **db.testzone.local** olacak. Ardından bu dosyanın konfigürasyonuna geçebiliriz. Birkaç temel bilgi vermek gerekirse:

```bash
sudo mkdir zones
cp db.local zones/db.testzone.local
```

```bash
### DOSYA ICERIGI ###
;
; BIND data file for local loopback interface
;
$TTL    604800
@       IN      SOA     testdnsserver.testzone.local. root.testzone.local. (
                              3         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
; name servers - NS records
        IN      NS      testdnsserver.testzone.local.

; name servers - A records
testdnsserver.testzone.local.   IN      A       14.8.18.121

; 14.8.18.0/24 - A records
testdnsclient.testzone.local.   IN      A       14.8.18.120
;
```



## Reverse Lookup Zone

Reverse Lookup Zone dosyası, ters DNS aramaları için DNS PTR kayıtlarını tanımladığımız yerdir. Bu zone dosyasını da yine zones klasörünün altında oluşturacağız. Yine db.local dosyasını Reverse Lookup Zone adımızı vererek buraya kopyalayalım. Bu isim **named.conf.local** dosyasında verdiğimizle aynı, yani **db.14.8.18** olacak. Ardından bu dosyanın konfigürasyonuna geçebiliriz.

```bash
cp db.local zones/db.14.8.18
```

```bash
### DOSYA ICERIGI ###
;
; BIND reverse data file for local loopback interface
;
$TTL    604800
@       IN      SOA     testdnsserver.local. root.testdnsserver.local. (
                              2         ; Serial 
                         604800         ; Refresh
                          86400         ; Retry 
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
; name servers  
        IN      NS      testdnsserver.testzone.local.
; PTR records
121     IN      PTR     testdnsserver.testzone.local.
120     IN      PTR     testdnsclient.testzone.local.
```



## Server Konfigürasyonu Kontrolleri

**named-checkconf** ve **named-checkzone** komutları işe oluşturduğumuz dosyaları kontrol edebiliriz. 

- Eğer hata yoksa **named-checkconf** komutunda herhangi bir hata ile karşılaşmayız, varsa bize hatanın ne olduğunu ve hangi dosyada olduğunu göstermektedir.
- **named-checkzone** komutunu ise `named-checkzone <ZONE_ADI> <ZONE_DOSYASI>` şeklinde çalıştırdığımızda **OK** çıktısını almalıyız.

Aşağıdaki örnekte yazdığımız dosyaları öncelikle **named-checkconf** ile kontrol ettik. Daha sonra named.conf.local dosyasında bir yeri bozup aynı komutla tekrar kontrol ettiğimizde bize hatayı gösteriyor. Daha sonra düzeltip tekrar **named-checkconf** dediğimizde ise hatayla karşılaşmıyoruz. En son ise iki zone dosyamızı da kontrol ederek **OK** çıktısını alıyoruz.

![1](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/mddq8nttmzpg9z0l9dvu.png)


Kontrolleri sağladıktan sonra bind9 servisimi yeniden başlatarak statusumu kontrol ediyorum.

```bash
sudo systemctl restart bind9
sudo systemctl status bind9
```

![2](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/qpr0gki6dan4n82zcg13.png)



## Client Konfigürasyonu

Client tarafında ise DNS Nameserver'imizi DNS sunucularımızın listesine eklememiz gerekiyor. Bu sunucular sunucumuzun `/etc/resolv.conf` dosyasında tutulmaktadır ve maksimum 3 adet nameserver verilebilir. Bu dosyayı kendi nameserverimize göre düzenleyelim.

```bash
sudo nano /etc/resolv.conf
```

```bash
### DOSYA ICERIGI ###
search testzone.local   
nameserver 14.8.18.121
#Diger (onceden bulunan) nameserverlerimizi buraya yazabiliriz
```

Daha sonrasında ise **nslookup <IP\>** veya **nslookup <HOSTNAME\>** komutlarıyla DNS Serverimizden doğru yanıt alıp alamadığımıza bakıyoruz. Doğru IP ve hostnameler verdiğimizde yanıtı alabiliyoruz fakat yanlış bir IP veya hostname denediğimizde **"server can't find"** çıktısı alıyoruz.

![3](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/rhv7e2lmij47y0bhbxd7.png)


Ayrıca **dig** komutuyla daha da ayrıntılı çıktı alabiliriz.

![4](https://dev-to-uploads.s3.amazonaws.com/uploads/articles/lmea2ngi2adsynmcfg0x.png)
