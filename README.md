# Server Doku
1. Debian 12 VPS auf Digital ocean erstellen
  - mit ssh login
2. Erster login
  - System updaten, weitere public schlüssel installieren
  - ufw und fail2ban installieren
  - ufw ssh freigeben, und ufw starten
```
ufw allow 22
ufw enable
```
- fail2ban configurieren
```jail.local
[sshd]
backend=systemd
enabled=true
port=ssh
filter=sshd
logpath=/var/log/auth.log
maxretry=3
```
3. A-Record DNS eintrag verknüpfen zu der IP des VPS
4. Grundlagen fuer Moodle installieren
```
postgresql nginx php php-fpm php-ctype php-curl php-dom php-gd php-iconv php-intl php-json php-mbstring php-simplexml php-xml php-zip php-pgsql php-soap 
```
5. php.ini editieren wie beschrieben in [moodle dev doc](https://moodledev.io/general/releases/4.5)
6. Neuen Nutzer erstellen fuer moodle die lokalen rechte hat
```
useradd -m -U -s /bin/bash -G users,sudo new_user && passwd new_user
cp .ssh dir to new_user
su -u new_user
sudo chown -R new_user:new_user ~/.ssh
sudo chmod 700 ~/.ssh
sudo chmod 600 ~/.ssh/authorized_keys
```
7. letsencrypt aufsetzen und starten
```
sudo apt install certbot python3-certbot-nginx
ufw allow 'Nginx Full'
certbot --nginx -d example.com
```
8. nginx config fuer moodle und /etc/nginx/sites-available/server.dns.net editieren und als einziges in sites-enabled verlinken
```
server {
    server_name server.dns.net;

    root /var/www/moodle;
    index index.php index.html;

    location / {
          try_files $uri $uri/ =404;
    }

    location ~ [^/]\.php(/|$) {
        fastcgi_split_path_info  ^(.+\.php)(/.+)$;
        fastcgi_param   PATH_INFO       $fastcgi_path_info;
        include snippets/fastcgi-php.conf;
        fastcgi_pass unix:/var/run/php/php8.2-fpm.sock;
        fastcgi_param SCRIPT_FILENAME $document_root$fastcgi_script_name;
        include fastcgi_params;
    }
    # This passes 404 pages to Moodle so they can be themed
    error_page 404 /error/index.php;
    error_page 403 =404 /error/index.php;

    # Hide all dot files but allow "Well-Known URIs" as per RFC 5785
    location ~ /\.(?!well-known).* {
                return 404;
    }

    # This should be after the php fpm rule and very close to the last nginx ruleset.
    # Don't allow direct access to various internal files. See MDL-69333
        location ~ (/vendor/|/node_modules/|composer\.json|/readme|/README|readme\.txt|/upgrade\.txt|/UPGRADING\.md|db/install\.xml|/fixtures/|/behat/|phpunit\.xml|\.lock|environment\.xml) {
        deny all;
        return 404;
        }

    listen 443 ssl; # managed by Certbot
    ssl_certificate /etc/letsencrypt/live/server.dns.net.net/fullchain.pem; # managed by Certbot
    ssl_certificate_key /etc/letsencrypt/live/server.dns.net/privkey.pem; # managed by Certbot
    include /etc/letsencrypt/options-ssl-nginx.conf; # managed by Certbot
    ssl_dhparam /etc/letsencrypt/ssl-dhparams.pem; # managed by Certbot

}
server {
    if ($host = server.dns.net) {
        return 301 https://$host$request_uri;
    } # managed by Certbot
    
    listen 80;
    server_name server.dns.net;
    return 404; # managed by Certbot
}
```
9. moodle in /var/www/ installieren
```
git clone https://github.com/moodle/moodle.git
git checkout MOODLE_405_STABLE
```
10. Permissions siehe  [Link](https://www.internalpointers.com/post/right-folder-permission-website)
```
chown -R new_user /var/www/moodle
chmod -R 0755 /path/to/moodle

mkdir /var/moodledata
chmod -R 777 /var/moodledata
```
11. Datenbank aufsetzen `sudo -u postgres psql`
```
CREATE USER moodleuser WITH PASSWORD '<password>';
CREATE DATABASE moodle WITH OWNER <user>;
```
12. Moodle installation starten im web und admin hinzufügen, davor und danach moodle order für www-data user freigen und wieder sperren um config.php zu erstellen
```
chown www-data /var/www/moodle
... Installation ...
chown new_user /var/www/moodle
```
13. Add cronjob
```
sudo -u www-data crontab -e
    Add
* * * * * /usr/bin/php  /var/www/moodle/admin/cli/cron.php >/dev/null
```
14. Mit Nextcloud als Oauth2 Provider verbinden und als repository verbinden [Doku](https://docs.moodle.org/405/de/Nextcloud_Repository)

Jetzt sollte alles funktionieren!

# Dev Doku
**Anpassungen des Moodle-Cores**

- **Anforderungen:**
	- [x] Hinzufügen eines Share Icons neben Nextcloud Ordnern
	- [?] Öffnen eines gesonderten Dialogs
	- [x]  Hinzufügen des Ordnerlinks als Dateiressource
- **Technische Details:**
    - Die Nextcloud spezifischen interaktion des backends ist in `repository/nextcloud/lib.php` definiert, bei der ich nextcloud Ordnern einen zusätzlichen wert vergebe sowie die Funktionen um zu prüfen, ob es sich um einen Ordner handelt, und um für diesen einen Link zu generieren.
    - Die Frontend interaktion findet in `repository/filepicker.js` statt, 
	- Dort habe ich die rendering funktion bisher vom IconView, als auch vom Treeview so angepasst, dass ein share icon neben dem Ordnernamen erscheint
    - Beim klicken wird die funktion `select_file` aufgerufen, die bisher den gleichen Dialog wie zum speichern von Dateien aufruft, nicht alle multiple choice Elemente sind valide
- **Limitations:** 
    - Bisher wird lediglich ein Link zum komprimierten Download des Ordners auf Moodle angezeigt, es sollte aber der gesamte Dateibaum auch Syncronisiert werden, laut den Anforderungen


# Moodle

<p align="center"><a href="https://moodle.org" target="_blank" title="Moodle Website">
  <img src="https://raw.githubusercontent.com/moodle/moodle/main/.github/moodlelogo.svg" alt="The Moodle Logo">
</a></p>

[Moodle][1] is the World's Open Source Learning Platform, widely used around the world by countless universities, schools, companies, and all manner of organisations and individuals.

Moodle is designed to allow educators, administrators and learners to create personalised learning environments with a single robust, secure and integrated system.

## Documentation

- Read our [User documentation][3]
- Discover our [developer documentation][5]
- Take a look at our [demo site][4]

## Community

[moodle.org][1] is the central hub for the Moodle Community, with spaces for educators, administrators and developers to meet and work together.

You may also be interested in:

- attending a [Moodle Moot][6]
- our regular series of [developer meetings][7]
- the [Moodle User Association][8]

## Installation and hosting

Moodle is Free, and Open Source software. You can easily [download Moodle][9] and run it on your own web server, however you may prefer to work with one of our experienced [Moodle Partners][10].

Moodle also offers hosting through both [MoodleCloud][11], and our [partner network][10].

## License

Moodle is provided freely as open source software, under version 3 of the GNU General Public License. For more information on our license see

[1]: https://moodle.org
[2]: https://moodle.com
[3]: https://docs.moodle.org/
[4]: https://sandbox.moodledemo.net/
[5]: https://moodledev.io
[6]: https://moodle.com/events/mootglobal/
[7]: https://moodledev.io/general/community/meetings
[8]: https://moodleassociation.org/
[9]: https://download.moodle.org
[10]: https://moodle.com/partners
[11]: https://moodle.com/cloud
[12]: https://moodledev.io/general/license

