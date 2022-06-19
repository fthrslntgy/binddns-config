**Bind DNS Server Kurulumu** dosyasındaki konfigürasyon dosyalarını içerir. Kurulumları yaptıktan sonra bu repodaki aynı konfigürasyonları koyup üzerinde değişiklikler yapabilirsiniz. Repoyu zip olarak indirdikten sonra (Code -> Download ZIP) indirdiğiniz yere çıkartınız (default olarak $HOME/Downloads). Dosyaları ilgili konumlara atmak için

**Server Tarafında:**

```bash
cd $HOME/Downloads/binddns-config-main
sudo cp -r dnsserver/etc/* /etc/
```

**Client Tarafında:**

```bash
cd $HOME/Downloads/binddns-config-main
sudo cp -r dnsclient/etc/* /etc/
```

komutlarını kullanabilirsiniz. Sonrasında ise kurulum dokümanındaki yönergelere göre kendi makinelerinize göre değişiklikler yapınız.
