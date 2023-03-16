# Matomo Installation

I den här handledningen så kommer jag beskriva dem stegen jag gör för att installera Matomo på en helt vanlig server.

Vill man gå lite djupare och leverera en distribuerad Matomo lösning i molnet med Kubernetes så kan jag gå närmre in på det i en framtida vloggserie. Eller om man vill förstå bakomliggande mekanismer så tveka inte att höra av er.


Förarbete:
[Requirements](https://matomo.org/docs/requirements/requirements/)


## Steg 1 - Konfigurera server:

Se till att du har PHP och Mysql installerat. Det finns guider på nätet, här använder jag mig av en Ubuntu server och jag installerar minimalt med paket för demot's skull. Så innan ni lanserar i produktion, se till att ni härdar era miljöer och brandväggar ordentligt. Det krävs nog en hel separat serie för det så jag går inte in på sådant här.

Om du vill använda `PHP7.4` gör följande:
```bash
sudo apt install php
```

För `PHP8.0` så krävs det att lägga till ytterligare ett repository(depå på svenska kanske🤔). Jag använder Ondrej, eftersom jag känner till det sedan tidigare och är bekväm med det. Vill man läsa mer om hans paket och bidrag så kan man kolla in hans profil på [Launchpad](https://launchpad.net/~ondrej).

Jag föredrar att använda nyare PHP versioner, så att jag kan vara snabb på bollen med buggar som uppstår. Medans andra föredrar att vänta tills dem är mer stabilare. Däremot så har jag stött på en del buggar med PHP8.2, så jag avvaktar nog med den ett tag till.

```bash
sudo apt install software-properties-common
sudo add-apt-repository ppa:ondrej/php
sudo apt-get update
sudo apt install php8.0
# Se till att vi har dem PHP paket som behövs.
sudo apt install php8.0-common php8.0-mysql php8.0-xml php8.0-xmlrpc php8.0-curl php8.0-gd php8.0-imagick php8.0-cli php8.0-dev php8.0-imap php8.0-mbstring php8.0-opcache php8.0-soap php8.0-zip php8.0-intl -y
```

Aktivera:
```bash
sudo a2dismod php7.4
sudo a2enmod php8.0
sudo systemctl restart apache2
php -v
```
Om du fortfarande får fel version, så finns det schysst knep:
```bash
sudo update-alternatives --config php
```
O så är det bara o välja.



### Installera Mysql:

```bash
sudo apt update
sudo apt install mysql-server
sudo systemctl start mysql.service
```
Kom ihåg att det här startar upp din Mysql utan användare och lösenord, så se till att säkra din Mysql installation innan du går ut i produktion. En bra guide hittar du [här](https://www.digitalocean.com/community/tutorials/how-to-install-mysql-on-ubuntu-20-04).

#### TL;DR

Snabbversionen:
```bash
sudo mysql
```
```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH mysql_native_password BY 'password';
exit
```
```bash
sudo mysql_secure_installation
mysql -u root -p
```
```sql
ALTER USER 'root'@'localhost' IDENTIFIED WITH auth_socket;
exit
```

Logga in och skapa en ny användare:
```bash
mysql -u root -p
```

#### Skapa en Databas

Jag använder `utf8mb4` eftersom Matomo stödjer special tecken med 4 bytes kodning, som t.ex. Emoji's. Läs mer [här](https://matomo.org/faq/how-to-update/how-to-convert-the-database-to-utf8mb4-charset/)!
Det sparar er tid o möda att installera korrekt så här från början, för om man missar det så kommer man till en punkt när man inser att man behöver konvertera hela databasen(been there, done that😅) och beroende på hur mycket data man har så kan det ta väldigt lång tid, med massa moment som kan korrumpera din DB.

```sql
CREATE DATABASE `matomopasvenska` /*!40100 DEFAULT CHARACTER SET utf8mb4 COLLATE utf8mb4_0900_ai_ci */ /*!80016 DEFAULT ENCRYPTION='N' */
```
Ibland kan man få fel med sortering med svenska tecken. Jag har inga större bekymmer med det just nu. Men om man vill optimera sin Databas för svenska språket, så rekommenderar jag att man använder rätt `COLLATE`(kollationering). Annars kan vissa listor hamna i fel bokstavsordning med `ÅÄÖ` till exempel.
För svenska använd:
```
utf8mb4_sv_0900_ai_ci
```
Om det inte funkar, kör på default:
```
utf8mb4_0900_ai_ci
```

#### Skapa en dedikerad användare

```bash
sudo mysql
```
```sql
CREATE USER 'matomo'@'matomopasvenska.demo' IDENTIFIED WITH authentication_plugin BY 'BytUtMot3ttSäkertLösenord!';
```

Eller om du får problem med `ERROR 1524 (HY000): Plugin 'authentication_plugin' is not loaded`, använd default lösenordshanteraren:
```sql
CREATE USER 'matomo'@'matomopasvenska.demo' IDENTIFIED BY 'BytUtMot3ttSäkertLösenord!';
```

Tilldela behörigheter till Databasen:
```
GRANT ALL PRIVILEGES ON matomopasvenska.* TO 'matomo'@'matomopasvenska.demo';
FLUSH PRIVILEGES;
exit
```
För exemplets skull så använder jag här `GRANT ALL PRIVILEGES`, men du bör noga överväga vilka privilegier din användare ska ha.

Enligt [Matomo](https://matomo.org/faq/troubleshooting/how-do-i-check-if-my-mysql-user-has-all-required-grants/):
```sql
GRANT SELECT, INSERT, UPDATE, DELETE, CREATE, INDEX, DROP, ALTER, CREATE TEMPORARY TABLES, LOCK TABLES ON matomo_db_name_here.* TO 'matomo'@'localhost';
```

### Installera Apache(om du inte redan gjort det)

Installera Apache server:
```bash
sudo apt update
sudo apt install apache2
```

Lägg till en vHost(jag kopierar default och lägger till i efterhand):
```bash
cd /etc/apache2/sites-available
sudo cp 000-default.conf matomopasvenska.conf
sudo vim matomopasvenska.conf
# Redigera så att du får rätt konfiguration
# Jag använder vim, du kanske föredrar någon annan
sudo vim apache2.conf
```

vHosten ser ut så här(minimum):
```conf
<VirtualHost *:80>
	ServerName matomopasvenska.demo

	ServerAdmin jorge@localhost
	DocumentRoot /var/www/matomopasvenska

    # Vi slår på debug
	LogLevel info debug

	ErrorLog ${APACHE_LOG_DIR}/error.log
	CustomLog ${APACHE_LOG_DIR}/access.log combined
</VirtualHost>
```

### Starta om Apache
```bash
sudo apache2ctl -t
sudo apache2ctl restart
sudo tail -f /var/log/apache2/access.log /var/log/apache2/error.log
```


## Steg 2 - Installation:

För enkelhetens skull så har jag byggt ett enkelt program som automatiserar dem flesta flödena i mina videos. Men för utbildningssyfte så går vi igenom dem steg för steg.

Hitta mina scripter på Github: [github.com/jorgeuos/matomo-setup](https://github.com/jorgeuos/matomo-setup)

Ladda ner Matomo och flytta till rätt mapp:
```bash
cd /var/www
curl -L https://github.com/matomo-org/matomo/releases/download/4.13.3/matomo-4.13.3.tar.gz | tar -xz
mv matomo matomopasvenska
cd /var/www/matomopasvenska
```

Du kan behöva starta om servern innan nästa steg:
```bash
sudo apache2ctl restart
```

### Navigera till din Matomo URL

I mitt fall så har jag installerat Matomo på matomopasvenska.demo.

Du får koppla din Matomo domän till din Matomo installation som du har konfigurerat i din vHost(`ServerName`).


#### Et voilà!

Nu tänker du säkert: "Meh, det är inte alls klart."

Nej, det är det kanske inte, men Matomo hjälper till lite på traven. Det är bara att följa instruktionerna.

```bash
chown -R www-data:www-data /var/www/matomopasvenska
find /var/www/matomopasvenska/tmp -type f -exec chmod 644 {} \;
find /var/www/matomopasvenska/tmp -type d -exec chmod 755 {} \;
find /var/www/matomopasvenska/tmp/assets/ -type f -exec chmod 644 {} \;
find /var/www/matomopasvenska/tmp/assets/ -type d -exec chmod 755 {} \;
find /var/www/matomopasvenska/tmp/cache/ -type f -exec chmod 644 {} \;
find /var/www/matomopasvenska/tmp/cache/ -type d -exec chmod 755 {} \;
find /var/www/matomopasvenska/tmp/logs/ -type f -exec chmod 644 {} \;
find /var/www/matomopasvenska/tmp/logs/ -type d -exec chmod 755 {} \;
find /var/www/matomopasvenska/tmp/tcpdf/ -type f -exec chmod 644 {} \;
find /var/www/matomopasvenska/tmp/tcpdf/ -type d -exec chmod 755 {} \;
find /var/www/matomopasvenska/tmp/templates_c -type f -exec chmod 644 {} \;
find /var/www/matomopasvenska/tmp/templates_c -type d -exec chmod 755 {} \;
```