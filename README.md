# Voi Node Taşıma
# Bu rehber sadece nodunu başka bir sunucuya taşımak isteyenler içindir.

Rues'in (kendisine sonsuz teşekkür ederek) rehberinin ilgili yerlerini aynen alıp taşıma işleminde yapılacak işlemleri göstereceğim.
> Başlamadan önce Eski Sunucunuzda Nodu Durdurun 

```sudo systemctl stop voi ```
> Sonra sunucunuza SFTP ile bağlanıp ```var/lib/algorand``` dizininin içindeki ```logging.config```  adlı dosyası pc ye indirip yedekleyin.  Aşağıda yeri geldiğinde yeni sunucunuza aktaracaksınız

> Bir de Wallet oluşturma aşamasında cüzdanınıza ait daha önce aldığınız 24 kelimeyi hazırlayın. İmport edeceğiz

#Node Kurulumuna sıfırdan başlar gibi başlıyoruz.

```console
# Güncellemeler:
sudo apt update && sudo apt-get upgrade -y
sudo systemctl start unattended-upgrades && sudo systemctl enable unattended-upgrades
```

<h1 align="center">Node Kurulumu</h1>

```console
# Hep Cosmosu yükleyecek değiliz, Algorandı yüklüyoruz:
sudo apt install -y jq gnupg2 curl software-properties-common
curl -o - https://releases.algorand.com/key.pub | sudo tee /etc/apt/trusted.gpg.d/algorand.asc

# Çıktısına ENTER diyebilirsiniz.
sudo add-apt-repository "deb [arch=amd64] https://releases.algorand.com/deb/ stable main"

# Tekrar güncelleyelim ve nodeun otomatik başlamaması için durduralım:
sudo apt update && sudo apt install -y algorand && echo OK
sudo systemctl stop algorand && sudo systemctl disable algorand && echo OK

# goal setupı yapalım
echo -e "\nexport ALGORAND_DATA=/var/lib/algorand/" >> ~/.bashrc && source ~/.bashrc && echo OK
sudo adduser $(whoami) algorand && echo OK

# yapılandırma işlem(satırlar tek tek girilecek):
sudo algocfg set -p DNSBootstrapID -v "<network>.voi.network" -d /var/lib/algorand/ &&\
sudo algocfg set -p EnableCatchupFromArchiveServers -v true -d /var/lib/algorand/ &&\
sudo chown algorand:algorand /var/lib/algorand/config.json &&\
sudo chmod g+w /var/lib/algorand/config.json &&\
echo OK

# Genesis (satırlar tek tek girilecek)
sudo curl -s -o /var/lib/algorand/genesis.json https://testnet-api.voi.nodly.io/genesis &&\
sudo chown algorand:algorand /var/lib/algorand/genesis.json &&\
echo OK
```

<h1 align="center">Nodeu çalıştıralım</h1>

```console
# Algorandı Voi olarak yapılandıralım (satırlar tek tek girilecek):
sudo cp /lib/systemd/system/algorand.service /etc/systemd/system/voi.service &&\
sudo sed -i 's/Algorand daemon/Voi daemon/g' /etc/systemd/system/voi.service &&\
echo OK

# ve nodeu çalıştralım:
sudo systemctl start voi && sudo systemctl enable voi && echo OK

# nodeu kontrrol edelim status ile:
goal node status

# ==> Genesis ID: voitest-v1
# ==> Genesis hash: IXnoWtviVVJW5LGivNFc0Dq14V3kqaXuK2u5OQrdVZo=
# Çıktının sonu bu şekilde olmalı (hash değişebilir)

# Hızlı sync olalım(satırlar tek tek girilecek):
goal node catchup $(curl -s https://testnet-api.voi.nodly.io/v2/status|jq -r '.["last-catchpoint"]') &&\
echo OK
# Burada bir kaç dakika bekleyelim

# Yine status yapalım ama bu sefer loglarda Catchpoint göreceğiz:
goal node status

# Bu komutla kontrol ettiğimizde Sync Timeın sıfırlanmasını ve loglarda Catchpointin gitmesini bekleyelim.
goal node status -w 1000
# Yukarda ki şartlar gerçekleince CTRL + C
```

<h1 align="center">Ödül alabilmek için Telemtry yapalım</h1>


# BURADA YİNE SUNUCUNUZA SFTP İLE BAĞLANIP ```var/lib/algorand``` DİZİNİNE girip ```logging.config``` dosyasını silin. 
#  eski sunucudan yedeklediğiniz dosyayı buraya aktarın
>devam edin
```console
sudo ALGORAND_DATA=/var/lib/algorand diagcfg telemetry enable &&\
sudo systemctl restart voi
```

<h1 align="center">Cüzdan oluşturma işlemleri</h1>

```console
# Cüzdan oluşturalım:

goal wallet new voi
```
># burda size verilen 24 kelimeyi almanıza gerek yok önceki cüzdanınızın kelimelerinini import edeceksiniz.
```console
goal account import
```
># elinizde var olan cüzdanınızın kelimelerinini import edin.
```console



# Şimdi bu kodları girelim ve bizden Imported adresimizi isteyecek.
# hepsini tek seferde  kopyala yapıştır yapabilirsiniz bu kodu
echo -ne "\nEnter your voi address: " && read addr &&\
echo -ne "\nEnter duration in rounds [press ENTER to accept default (2M)]: " && read duration &&\
start=$(goal node status | grep "Last committed block:" | cut -d\  -f4) &&\
duration=${duration:-2000000} &&\
end=$((start + duration)) &&\
dilution=$(echo "sqrt($end - $start)" | bc) &&\
goal account addpartkey -a $addr --roundFirstValid $start --roundLastValid $end --keyDilution $dilution
# Imported adresinden sonra ki soruda ENTER diyip varsayılanı tercih edebiliriz.
# Import işleminin tamamlanmasını bekleyin ve Participation IDinizi saklayın.

```

```console
# Son olarak Bu komutu yine hepsini tek seferde kopyalayıp yapıştın
getaddress() {
  if [ "$addr" == "" ]; then echo -ne "\nEnter your voi address: " && read addr; else echo ""; fi
}
getaddress &&\
goal account changeonlinestatus -a $addr -o=1 &&\
sleep 1 &&\
goal account dump -a $addr | jq -r 'if (.onl == 1) then "You are online!" else "You are offline." end'
```


Bu yolla node sağlık skorunuzun korunarak taşıma işlemi yaptınız.

1-2 saat sonra node sağlık skorunuzun aynen taşındığını https://cswenor.github.io/voi-proposer-data/health.html  burdan bakıp kontrol edebilirsiniz.

># RUES'E SONSUZ TEŞEKKÜR EDERİM. HERŞEY ONUN SAYESİNDEDİR.



