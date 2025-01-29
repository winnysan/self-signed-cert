# self-signed-certificate

## WSL

### vytvorenie konfiguračného súboru `ssl.conf`

```bash
[req]
default_bits       = 2048    # Nastavuje veľkosť RSA kľúča na 2048 bitov
default_md         = sha256  # Používa SHA-256 ako hashovaciu funkciu na podpis certifikátu
distinguished_name = dn      # Odkazuje na sekciu [dn], kde sú uvedené údaje o organizácii
x509_extensions    = v3_req  # Odkazuje na sekciu [v3_req], ktorá definuje rozšírenia pre certifikát (vrátane SAN)
prompt             = no      # Zabezpečuje, že OpenSSL nebude interaktívne pýtať údaje, ale použije hodnoty z konfigurácie

[dn]
C  = SK           # Krajina
ST = Bratislava   # Štát/kraj
L  = Bratislava   # Mesto
O  = MyLocal      # Organizácia (napr. názov projektu)
CN = mylocal.com  # Common Name (CN) – hlavná doména pre SSL certifikát

[v3_req]
subjectAltName = @alt_names  # Odkazuje na sekciu [alt_names], kde sú definované alternatívne domény

[alt_names]
DNS.1 = mylocal.com
DNS.2 = mylocal.sk
DNS.3 = sub.mylocal.com
```

### vytvorenie certifikátu

```bash
openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout mylocal.key -out mylocal.crt -config ssl.conf

# -x509 Vytvorí self-signed certifikát.
# -nodes Bez hesla na privátny kľúč.
# -days 365 Platnosť na 1 rok.
# -newkey rsa:2048 Vytvorí nový 2048-bitový RSA kľúč.
# -keyout mylocal.key Privátny kľúč.
# -out mylocal.crt Výstupný certifikát.
# -config ssl.conf Použitie vlastnej konfigurácie.
```

### skopírovanie do `nginx`

```bash
# ak adresár `certs` ešte neexistuje
sudo mkdir -p /etc/nginx/certs
sudo chmod 700 /etc/nginx/certs        # Obmedzí prístup len na vlastníka adresára
sudo chown root:root /etc/nginx/certs  # Nastaví vlastníka a skupinu adresára na systémového superužívateľa (root)

sudo cp mylocal.crt mylocal.key /etc/nginx/certs
```

### nastavenie `nginx`

pridať do súboru: `sudo nano /etc/nginx/sites-available/default`

```bash
server {
        # Tento server bude počúvať na port 443 a používať SSL/TLS na zabezpečenie komunikácie cez HTTPS
        listen 443 ssl;
        # Tento server blok bude spracovávať požiadavky pre mylocal.com a mylocal.sk. NGINX bude kontrolovať, či sa požiadavka zhoduje s jednou z týchto domén
        server_name mylocal.com mylocal.sk;

        # Cesta k SSL certifikátu pre zabezpečenie pripojenia
        ssl_certificate /etc/nginx/certs/mylocal.crt;
        # Cesta k privátnemu kľúču pre tento certifikát.
        ssl_certificate_key /etc/nginx/certs/mylocal.key;

        # Tento blok je definícia spracovania všetkých požiadaviek, ktoré prídu na / (t.j. akékoľvek URL začínajúce s /)
        location / {
                # NGINX bude fungovať ako reverse proxy a všetky požiadavky presmeruje na lokálny server bežiaci na portu 3000
                proxy_pass http://127.0.0.1:3000;
                # proxy_set_header direktívy zabezpečujú, že NGINX odošle správne hlavičky na backend server
                # Nastaví hlavičku Host na hodnotu domény, na ktorú bola požiadavka zaslaná (napr. mylocal.com)
                proxy_set_header Host $host;
                # Odosiela IP adresu klienta, ktorý urobil požiadavku
                proxy_set_header X-Real-IP $remote_addr;
                # Udržiava si zoznam všetkých IP adries, cez ktoré požiadavka prešla
                proxy_set_header X-Forwarded-For $proxy_add_x_forwarded_for;
                # Posiela protokol (http alebo https) ako hlavičku na backend
                proxy_set_header X-Forwarded-Proto $scheme;
        }
}

server {
        # Tento server blok počúva na port 80, čo je štandardný port pre HTTP
        listen 80;
        # Opäť spracováva požiadavky pre domény mylocal.com a mylocal.sk
        server_name mylocal.com mylocal.sk;

        # Tento riadok zabezpečuje, že ak príde požiadavka na HTTP (port 80), NGINX ju automaticky presmeruje na HTTPS (port 443)
        return 301 https://$host$request_uri;

        # 301 Je permanentné presmerovanie (permanent redirect)
        # $host Sa postará o to, že sa použije rovnaká doména, ktorá bola uvedená v pôvodnej požiadavke (napr. mylocal.com alebo mylocal.sk)
        # $request_uri Zachová pôvodnú URL cestu (napr. /page1)   
}
```

reštartovať `nginx`

```bash
sudo nginx -t
sudo systemctl reload nginx
```

### nainštalovanie certifikátu

skopírovanie z `WSL` na disk `c`

```bash
cp mylocal.crt /mnt/c/users/marekvi/mylocal.crt
```

inštalácia a overenie

```ps
# Powershell ako správca

Import-Certificate -FilePath "C:\Users\marekvi\mylocal.crt" -CertStoreLocation "Cert:\LocalMachine\Root"  # Naištaluje certifikát medzi dôveryhodné pre celý systém
Get-ChildItem -Path "Cert:\LocalMachine\Root" | Where-Object { $_.Subject -match "mylocal.com" }
```

vyčistenie cache

```ps
certutil -urlcache * delete
```

upraviť `/etc/hosts`

```ps
# Powershell ako správca
notepad.exe C:\Windows\System32\drivers\etc\hosts

127.0.0.1  mylocal.com
127.0.0.1  mylocal.sk
```

vyčistenie DNS cache

```ps
ipconfig /flushdns
```
