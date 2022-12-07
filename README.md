# Bitcoin/LN Node Dienste über eigene Domain aus dem Clearnet erreichbar machen (inkl. SSL-Zertifikate)

Mein vorheriges [Tutorial](https://github.com/Surenic/oracle-vps-wireguard-server-LN) hat euch gezeigt, wie ihr eure Node mittels VPS und Wireguard Server im Clearnet mittels fester Public IP Adresse erreichbar macht. Die jeweiligen Dienste konnten hierbei mittels Portweiterleitung erreichbar gemacht werden. Bspw. konnte LNbits via `https://PUBLIC_IP:5001` aufgerufen werden (sofern zuvor ein SSL-Zertifikat hinterlegt war.)

Dank eines Web Servers namens nginx (gesprochen engine-x), der direkt auf dem VPS installiert wird, können wir uns zum Einen die Portweiterleitung sparen, direkt SSL-Zertifikate hinterlegen, eigene Domains nutzen und dadurch nach außen anschaulichere Aufrufe generieren wie z.B https://lnbits.meine-domain.de

### Voraussetzungen

- Eine Bitcoin (und Lightning) Node mit den entsprechenden Diensten
- Ein Server mit erreichbarer IP Adresse, der mittels VPN mit der Node verbunden ist.
- Eine eigene Domain mit erstellbaren Subdomains inklusive Eintragung von A- und TXT-Records (zu finden in den DNS-Einstellungen eures Domain-Anbieters). Alternativ: ein dyndns-Anbieter wie duckdns.org, der TXT-Records setzen kann.

## Domain erstellen und konfigurieren

Macht euch Gedanken darüber, welche Services ihr erreichbar machen wollt und unter welchem Domainnamen diese erreichbar sein soll. Folgender Ablauf zeigt euch die Einrichtung von LNbits. Ihr könnt jeden beliebigen Domainnamen und Service wählen den ihr wollt. nginx ist in der Lage, Webanfragen nach Domainnamen zu filtern und somit mehrere Subdomains mit unterschiedlichen Services zu kombinieren. Heißt: auch wenn lnbits.meine-domain.de und mempool.meine-domain.de via A-Record an die selbe IP-Adresse geleitet werden, leitet nginx die Anfrage gemäß eurer Settings um.

Wir wählen nun beispielhaft `lnbits.meine-domain.de`

Ruft die Verwaltungsseite eures Domain-Anbieters auf und erstellt die entsprechende Subdomain. Sucht nun nach den DNS-Einstellungen. Ihr findet die Möglichkeit einen A-Record zu setzen. Hier tragt ihr die PUBLIC_IP eures Servers ein, auf dem der VPN Service läuft. Je nach Provider kann es einige Zeit dauern, bis die Weiterleitung gesetzt ist. 

## SSL Zertifikat

Zum Installieren eines SSL-Zertifikat, das durch [Let's Encrypt](https://letsencrypt.org/) ausgestellt wird, nutzen wir eine Software namens certbot. Im Terminalfenster eures VPS Servers gebt ihr folgende Befehle zum Installieren und starten der Software ein

```
sudo snap install core; sudo snap refresh core
sudo snap install --classic certbot
sudo ln -s /snap/bin/certbot /usr/bin/certbot
sudo snap set certbot trust-plugin-with-root=ok
sudo certbot certonly --manual --preferred-challenges dns
```

Certbot erfragt neben einigen anderen Dingen nach eurer gewählten Domain. Gebt diese (`lnbits.meine-domain.de`) entsprechend ein.

Es folgt die Frage nach einem TXT-Record, den ihr auf der DNS-Verwaltungsseite eures Domain-Anbieters setzen könnt. Certbot verlangt einen TXT-Record für `_acme-challenge.lnbits.meine-domain.de`. Kopiert den angezeigten Wert und fügt in gemeinsam mit dem Präfix ein. Auf strato sieht das beispielsweise so aus:

<img src=https://raw.githubusercontent.com/Surenic/reverse-proxy-node-services/main/TXT_Record.png width="400">

Speichert das Ganze und wartet einen Moment. Der Eintrag eines TXT-Records kann länger dauern, ist aber in der Regel in wenigen Sekunden aktiv.

Wie man den TXT-Record für duckdns.org setzt, hat [HODLmeTight](https://github.com/TrezorHannes) [hier](https://github.com/TrezorHannes/vps-lnbits-wg#vps-ssl-certificate) gut erklärt.

Drückt im Terminalfenster nun auf Enter und bestätigt dem certbot damit, dass diese Domain auch tatsächlich eure ist. Das Zertifikat und der Key werden daraufhin unter `/etc/letsencrypt/live/lnbits.meine-domain.de` abgelegt

## Installieren und einrichten von nginx

Auf eurem VPS Server führt ihr folgende Befehle aus:

```
sudo apt update
sudo apt install nginx
```

Nachdem die Installation abgeschlossen ist, erzeugt ihr eine Konfigurationsdatei für eure neue Seite

```
sudo nano /etc/nginx/sites-available/lnbits.conf
```

und füllt sie wie folgt

```
server {
        # Binds the TCP port 80
        listen 80;
        # Defines the domain or subdomain name
        server_name lnbits.meine-domain.de;
        # Redirect the traffic to the corresponding 
        # HTTPS server block with status code 301
        return 301 https://$host$request_uri;
       }

server {
        listen 443 ssl; # tell nginx to listen on port 443 for SSL connections
        server_name lnbits.meine-domain.de; # tell nginx the expected domain for requests

        access_log /var/log/nginx/lnbits-access.log; # Your first go-to for troubleshooting
        error_log /var/log/nginx/lnbits-error.log; # Same as above

        location / {
                proxy_pass https://10.8.0.2:5001; # Dies ist die VPN IP eurer Node sowie der Port des LNbits Dienstes
                proxy_set_header Upgrade $http_upgrade;
                proxy_set_header Connection 'upgrade';
                proxy_set_header X-Forwarded-Proto https;
                proxy_set_header Host $host;
                proxy_http_version 1.1; # headers to ensure replies are coming back and forth through your domain
       }

        ssl_certificate /etc/letsencrypt/live/lnbits.meine-domain.de/fullchain.pem; # Point to the fullchain.pem from Certbot 
        ssl_certificate_key /etc/letsencrypt/live/lnbits.meine-domain.de/privkey.pem; # Point to the private key from Certbot
}
```

Schaut euch jede Zeile genauestens an und achtet beim Erstellen weiterer Dienste auf das Ändern der Domain, log-Files, lets-encrypt keys usw.

Die erste Zeile im location-Bereich zeigt die URL des Services. In diesem Fall ist 10.8.0.2 die IP der Node in der Wireguard VPN Umgebung und 5001 der Port, auf dem LNbits läuft.

Speichert die Datei mit STRG-x, Y/J und Enter.

Nun überprüfen wir die Datei, spiegeln sie in den aktiven sites-enabled-Ordner und starten nginx neu

```
sudo nginx -t
nginx: the configuration file /etc/nginx/nginx.conf syntax is ok
nginx: configuration file /etc/nginx/nginx.conf test is successful
sudo ln -s /etc/nginx/sites-available/paymeinsats.conf /etc/nginx/sites-enabled/
sudo systemctl restart nginx
```

## Was haben wir getan

`https://lnbits.meine-domain.de > PUBLIC_IP > nginx > VPN_NODE_IP:PORT`

Der Aufruf von `https://lnbits.meine-domain.de` sollte nun unter Berücksichtigung eines gültigen SSL-Zertifikats über den VPN Service des VPS auf die Node und über den Port den entsprechenden Dienst erreichen und diesen damit vollständig für das Clearnet verfügbar machen.

## Zu guter Letzt

Die SSL-Zertifikate von Let's Encrypt sind lediglich 30 Tage gültig. Um diese automatisch zu erneuern, richten wir auf dem VPS noch einen Cronjob ein

```
sudo crontab -e
```

Folgende Zeile ans untere Ende einfügen

```
0 12 * * * /usr/bin/certbot renew --quiet
```

STRG-X, Y/J und Enter speichert den cronjob.



Wenn euch das Tutorial gefallen hat und alles funktioniert, wie es soll, freue ich mich, wenn ihr meinen LNurlp-Link mal ausprobiert ;)

[<img src=https://raw.githubusercontent.com/Surenic/oracle-vps-wireguard-server-LN/main/QR.png width="200" height="200">](https://lnbits.surenic.net/lnurlp/2)

Folgt mir auf Twitter!

[<img src=https://upload.wikimedia.org/wikipedia/commons/4/4f/Twitter-logo.svg width="50" height="50">](https://twitter.com/surenic)

Ansonsten freue ich mich auf Verbesserungen, Anregungen und ähnliches
