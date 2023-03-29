# Matomo På Svenska - Del 3 - Arkiveringar

![Matomo På Svenska - Del 3 - Arkiveringar](../../images/MPS-3-Arkiveringar.png)

I den här videon visar jag hur du konfigerar Matomo för optimeringar med förakriverade rapporter.

Jag går även igenom varför vissa rapporter inte förarkiveras.

Länkar:
* [How to Set up Auto-Archiving of Your Reports](https://matomo.org/faq/on-premise/how-to-set-up-auto-archiving-of-your-reports/)
* [Crontab Guru - ifall du vill ha snabb hjälp med en specifik körnings tid](https://crontab.guru/)

I videon så skapar jag ett bash script som exekveras en gång i timman:
```bash
sudo vim /etc/cron.hourly/mps.sh
```

Kopiera in texten:
```sh
#!/bin/bash

/var/www/matomopasvenska/console core:archive
```

Eller om du vill spara utskriften i en log fil:

```sh
#!/bin/bash

/var/www/matomopasvenska/console core:archive > /var/log/matomo.log
```

Enligt matomo så är `--url` parametern obligatorisk, men det funkar för mig utan:
```vi
#!/bin/bash

/var/www/matomopasvenska/console core:archive --url=http://example.org/matomo/ > /var/log/matomo.log
```

Kom ihåg att göra filen exekverbar:
```bash
sudo chmod +x mps.sh
```

Alternativ sätt att göra det på, är att lägga till i en generell crontab:
```bash
crontab -e
```

Lägg in en oneliner med definierat intervall(t.ex 5 över varje heltimma):
```bash
5 * * * * /usr/bin/php /path/to/matomo/console core:archive --url=http://example.org/matomo/ > /dev/null
```



