
https://github.com/Kioshi/SPOS
https://github.com/jindrichskupa/kiv-spos/

Obecné poznámky:
zprovoznění ssh:
	`apt-get install ssh`
	`apt-get install openssh-server openssh-client`
/etc/apt/sources.list - seznam zdrojů pro apt-get
/etc/resolv.conf - seznam dns serverů
/etc/hosts - seznam pevně překládaných doménových jmen

nastavení ip interfacu:
	`ifconfig eth0 192.168.0.104 netmask 255.255.0.0`
nastavení ip subinterfacu:
	`ifconfig eth0:0 192.168.0.104 netmask 255.255.0.0`
přidání ip:
	`ip addr add 192.168.0.255/16 dev eth0`
odebrání ip:
	`ip addr del 192.168.0.255/16 dev eth0`
zobrazení všech ip adres:
	`ip addr show`

restart služby:
	service bind9 restart
zastavení služby:
	service bind9 stop

netstat -tupln - vypíše, co poslouchá na jaké adrese a portu

adduser user - přidání uživatele

#!/bin/bash - na začátku každého bash scriptu
`apt-get remove --purge apache2` - odinstaluje apache včetně všech konfiguráků



SSH klíče
==============================================================================
na stroji, ze kterého se připojujeme:
* `ssh-keygen` - vygeneruje klíče, defaultně rsa, pro dsa: ssh-keygen -t dsa, pro rsa2048: ssh-keygen -t rsa -b 2048
* `cd /root/.ssh`
* `scp id_rsa.pub root@147.228.0.51:~/id_rsa.pub`

na stroji, na který se připojujeme:
* `cat id_rsa.pub >> ~/.ssh/authorized_keys` - .ssh v home adresáři uživatele, na kterého se přes ssh připojujeme, pokud neexistuje, vytvoříme

```bash
/etc/ssh/sshd_config
	#PasswordAuthentication yes - zakomentovat, defaultně je už zakomentováno
```

pro připojení bez zadávání adresy a username:
zadáváme na stroji, ze kterého se připojujeme

```bash
~/.ssh/config
Host spos - pomocí jakého slova se připojujeme
	user root - na jakého uživatele
	HostName 192.168.1.3 - na jakou adresu
	IdentityFile ~/.ssh/id_rsa - pomocí jakého klíče
```

Typically you want the .ssh directory permissions to be 700 (drwx------) and the public key (.pub file) to be 644 (-rw-r--r--). Your private key (id_rsa) should be 600 (-rw-------). Lastly, your home directory should not be writeable by the group or others (at most 755 (drwxr-xr-x)).


Firewall
==============================================================================

```bash
iptables -A INPUT -s 147.228.0.0/16 -p tcp --dport 22 -j ACCEPT - přidání pravidla
iptables -D INPUT -s 147.228.0.0/16 -p tcp --dport 22 -j ACCEPT - odebrání pravidla
iptables -A INPUT -p tcp --dport 22 -j DROP - zakázání všeho na portu 22 přes TCP
```

* `iptables-save > iptab` - uložení iptables
* `iptables-restore < iptab` - obnovení ip tables

* `iptables -L -n -v` - vypsání aktuálních pravidel

Obnovení iptables po rebootu:

```bash
iptables-save > /etc/network/iptables
/etc/network/interfaces
	up /sbin/iptables-restore /etc/network/iptables
```

Odříznutí uživatele po několika neúspěšných přihlášeních:
`apt-get install fail2ban`

```bash
/etc/fail2ban/jail.conf
	v sekcích [DEFAULT] a [ssh]
	maxretry = 3 - odřízne uživatele po 3 neúspěšných pokusech o přihlášení
	bantime = 20 - odřízne ho na 20 sekund
```

Disky
==============================================================================

* `lsblk` - Vypsání dostupných disků
* `fdisk -l` - vypsání všech disků k dispozici

```bash
sdb    	8:16	0    	200M	0  	disk
sdc    	8:32	0    	200M  	0 	disk
sdd    	8:48	0	200M  	0  	disk
∟ssd1	8:49	0	199M  	0 	part
```

* `cfdisk /dev/sdd` - Smazání oblasti sdd1
Zvolit [Delete] → [Write] → yes → [Quit]
* `cfdisk /dev/sdd` - Vytvoreni oddilu
Zvolit [New] → [Size: 200M] → [Primary] → [Write] → yes → [Quit]
 type -> Linux raid autodetect

* `apt install mdadm` - Instalace balíčku mdadm

* `mdadm -Cv /dev/md0 -l5 -n3 /dev/sd[bcd]` - Založení raidu 5
* `mdadm --detail /dev/md0` - Vypíše stav raidu (kapacita, typ, počet zařízení…)

* Format with vFat File System `sudo mkfs.vfat /dev/sd[bcd]`.
* Format with NTFS File System `sudo mkfs.ntfs /dev/sd[bcd]`.
* Format with EXT4 File System `sudo mkfs.ext4 /dev/sd[bcd]`.

Vytvoření LVM nad raidem
==============================================================================
* `apt install lvm2` - Instalace balíčku lvm2
* `vgcreate data /dev/md0` - Vytvoření skupiny pojmenované data
* `vgdisplay` - Vypsání existujících skupin
* `lvcreate -L 396M -n share data` - Vytvoření logického svazku pojmenovaného share
* `lvdisplay` - Vypsání existujících svazků

Naformátování svazku a automatické připojení jako /home
==============================================================================

* `mkfs.ext4 /dev/data/share` - Naformátování share na ext4

* `nano /etc/fstab` - Nastavení připojení share jako /home

```bash
/dev/data/share	/home	ext4	defaults		0	2
```
* `mount -a` - Připojení všech oddílů z fstab

* `lsblk -f` - Vypsání diskových oddílů včetně použitých FS a přípojných bodů

* `e2label /dev/data/share share` - Nastavení názvu svazku na share

* `nano /etc/fstab` - Nastavení připojení tak, aby se provedlo podle názvu svazku

```bash
LABEL=share	/home	ext4	defaults		0	2
```
tzn. jen se /dev/data/share přepíše na LABEL=share

* `apt-get install jfsutils` - instalace JFS filesystému
* `apt-get install xfsprogs` - instalace XFS filesystému
* `apt-get install reiserfsprogs` - instalace ReiserFS filesystému

```
mkfs.ext4 -F /dev/md0   #vytvori souborovy system
mount /dev/md0 /mnt/md0   #mountne 

/proc/mdstat - obsahuje stav raidů

`mkfs.ext4 /dev/sdb` - vytvoření filesystému ext4 na zařízení sdb, může být i raid
`mkfs.ext4 -L label /dev/sda` - přidá label k disku
`blkid` - zjištění filesystemu disku nebo raidu a UUID disků

mount /dev/sdb /mnt/dir - mount disku sdb do adresáře /mnt/dir
mount - bez parametrů vypíše co je kam namounotváno
umount /mnt/dir - odmountuje disk z adresáře /mnt/dir

/etc/fstab - obsahuje co se má mountovat po nabootování
	co		kam		filesystem
	/dev/md1        /mnt/raid5	ext4	defaults        0       0 #místo /dev/md1 může být UUID=..., nebo LABEL=...

mount -a - namountuje všechno v /etc/fstab

mdadm --stop /dev/md1 - zastaví raid
mdadm --remove /dev/md1 - odstraní raid
mdadm /dev/md0 -f /dev/sdb - nastaví disk sdb v raidu md0 jako failed
mdadm /dev/md0 -r /dev/sdb - odstraní disk sdb z raidu md0
mdadm /dev/md0 -a /dev/sdb - vrátí disk sdb do raidu md0


LVM:
apt-get install lvm2
pvcreate /dev/sdb - připraví disk pro LVM
vgcreate vgdata /dev/sdb - vytvoří skupinu s názvem vgdata s jedním diskem /dev/sdb
vgextend vgdata /dev/sdc - přidá /dev/sdc do skupiny vgdata
lvcreate -L 30M -n test vgdata - vytvoří LVM s názvem test nad skupinou vgdata o velikosti 30MB

pvdisplay - zobrazí všechny fyzické jednotky vytvořené pomocí pvcreate
vgdisplay - zobrazí všechny skupiny
lvdisplay - zobrazí všechna LVM

lvremove /dev/vgdata/test - odstraní LVM /dev/vgdata/test
vgremove vgdata - odstraní skupinu vgdata
pvremove /dev/sdb - ostraní přípravu pro LVM z disku /dev/sdb
```

DNS
==============================================================================
* `apt-get install bind9`

```bash
/etc/bind/named.conf.local - soubor se zonami
zone "test.spos" { //vytvoreni zony test.spos
        type master;
        allow-transfer { any; };
        file "/etc/bind/db.test.spos"; //zaznamy dane zony budou v souboru /etc/bind/db.test.spos
};
```

* `cd /etc/bind`
* `cp db.local db.test.spos` - vytvoříme náš soubor se záznamy a jako šablobu použijeme db.local

```bash
/etc/bind/db.test.spos
$TTL    604800
@       IN      SOA     test.spos. root.localhost. ( ;v hlavicce nazev nasi domeny misto localhost
                              2         ; Serial
                         604800         ; Refresh
                          86400         ; Retry
                        2419200         ; Expire
                         604800 )       ; Negative Cache TTL
;
@       IN      NS      localhost.
@       IN      A       192.168.0.104  ;zaznam pro vsechno co konci test.spos, preklada se na 192.168.0.104
```

* `/etc/resolv.conf` - na první místo přidat adresu našeho DNS serveru
* `host test.spos` - ověření funkčnosti


POHLEDY (VIEWS):
==============================================================================
obsah všech zónových souborů uzavřeme do view, každý soubor jiný view, většinou jen `named.conf.local` a `named.conf.default-zones`

```bash
příklad pro /etc/bind/db.test.spos
view "mynet" { //view s nazvem mynet
	match-clients { 192.168.0.0/16; }; //pro koho view plati, muze byt { any; }
	recursion yes;  //povolit rekurzi yes/no                                  e
		zone "test.spos" {
		        type master;
		        file "/etc/bind/db.test.spos";
		};
};
```
...obdobně pro named.cond.default-zones (jiný název než mynet), případně další

REVERZNÍ ZÁZNAMY
==============================================================================

```bash
/etc/bind/named.conf.local
zone "168.192.in-addr.arpa" { //prvni 1 az 3 hodnoty jsou cast site, z niz prekladame adresy
        type master;
        file "/etc/bind/db.168.192"; //soubor se zaznamy
};
```
```bash
/etc/bind/db.168.192 - soubor se zaznamy
	do hlavicky - 168.192.in-addr.arpa místo localhost. (na konci není tečka)
	zaznamy pridavat ve tvaru
	104.0	IN	PTR	test.spos. ;prvni hodnota je konec adresy, ktera ma byt prekladana v opacnem poradi bytu
				   	   ;posledni je domena, na kterou ma byt adresa prekladana
```
* `host 192.168.0.104` - test funkcnosti

WEBSERVER:
==============================================================================

* `apt-get install apache2`
* `apt-get install curl`

```bash
/etc/apache2/ports.conf
	#na jakych portech a adresach muze apache poslouchat
	Listen 80 #poslouchani vsude na portu 80
	Listen 127.0.0.1:1111 #poslouchani jen na loclahost na portu 1111
	<IfModule ssl_module> #na jakych portech muze poslouchat pro sity s SSL
        	Listen 443 
	</IfModule>

	<VirtualHost *:80>
    		<Location />
     		 Require ip 192.168.0.0/24    #bude poslouchat na danem rozmezi IP
   		</Location>
    		...
	</VirtualHost>
```

1. `cd /etc/apache2/sites-available`
2. `cp 000-default.conf www1.conf` - použijeme 000-default.conf jako šablonu pro naši stránku

```bash
/etc/apache2/sites-available/www1.conf
	<VirtualHost *:80> #stranka je pro port 80
	ServerName www1.test.spos
	DocumentRoot /var/www/www1
```

3. `/var/www/www1/` - adresář www1 nutno vytvořit, do něj dáme obsah stránky (index.html, index.php...)

4. `a2ensite www1` - přidá www1.conf do /etc/apache2/sites-enabled

do DNS přidat záznam:

```bash
www1    IN      A       192.168.0.104
```

* `curl www1.test.spos` - ověření funkčnosti
* `apache2ctl -S` - ukáže stránky které běží a na jakých portech běží

* `a2enmod status` - server status
* `curl www1.test.spos/server-status` - zobrazit pomocí 

```bash
/etc/apache2/mods-available/status.conf
	Require ip 192.168.0.0/16 #odkud lze status zobrazit
```

PHP:
==============================================================================
* `apt-get install php5`
* `apt-get install libapache2-mod-php5`
* `a2enmode php5`

```bash
/etc/apache2/mods-available/dir.conf
	index.php před index.html
```

výpis IP adresy klienta v php:

```php
<?php
echo $_SERVER['REMOTE_ADDR']
?>
```

Omezení adresáře pro php:
```bash
/etc/php5/apache2/php.ini - pokud je jinde, lze vypsat phpinfo() a grepnout php.ini
	open_basedir = /var/www
```

Ověřeni:
```php
<?php
echo file_get_contents("/etc/passwd");
?>
```

SSL:
==============================================================================
* `cd /etc/apache2/`
* `mkdir certs`
* `cd certs`
* `openssl genrsa -des3 -out server.key 1024`
* `openssl req -new -key server.key -out server.csr`
* `cp server.key server.key.org`
* `openssl rsa -in server.key.org -out server.key`
* `openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt` - vytvoří soubory server.crt a server.key

`cd /etc/apache2/sites-available`
`cp default-ssl.conf www1ssl.conf`

```bash
/etc/apache2/sites-available/www1ssl.conf
	<VirtualHost _default_:443> #stranka je pro port 443
	DocumentRoot /var/www/www1ssl
	ServerName www1.test.spos
	SSLCertificateFile    /etc/apache2/certs/server.crt
        SSLCertificateKeyFile /etc/apache2/certs/server.key
```

* `/var/www/www1ssl/ `- adresář www1ssl nutno vytvořit, do něj dáme obsah stránky (index.html, index.php...)

* `a2ensite www1ssl`
* `a2enmod ssl`
* `curl -k https://www1.test.spos`

Ballancer:
==============================================================================
```
Pound:
apt-get install pound

/etc/default/pound
	startup = 1

/etc/pound/pound.cfg
	ListenHTTP
        	Address 127.0.0.1 #tady adresu serveru, ktery dela ballance (nesmi byt localhost, ale adresa, na kterou je zaznam v DNS)
        	Port    8080 	#tady port, na kterem ma poslouchat

        	## allow PUT and DELETE also (by default only GET, POST and HEAD)?:
        	xHTTP           0

        	Service
                	BackEnd     #back endy jsou ty servery, mezi kterymi se ma balancovat
                        	Address 127.0.0.1  #adresa serveru
                        	Port    80
                	End
        	End
	End

NginX:
apt-get install nginx - nesmí nic poslouchat na portu 80

/etc/nginx/sites-available/www.conf - vytvořit
	upstream www1.test.spos { #tady je jmeno stranky, kterou chceme balancovat
		ip_hash; #rika, ze klient bude vzdy komunikovat se stejnym server jako minule (nemusi tam byt)
	        server  192.168.0.104:8008      weight=3        max_fails=2     fail_timeout=10s; #adresy s portem serveru, mezi kterymi se ma balancovat
	        server  192.168.0.105:8080      weight=1        max_fails=2     fail_timeout=10s;
	}
	server {
 	       listen  80; #port, na kterem ma nginx poslouchat
 	       server_name www1.test.spos; #jmeno stranky, kterou chceme balancovat
		
		location /static {
			root /var/www/web01; #adresar static bude brat ze zadaneho adresare
		}

 	       location / {
 	               proxy_pass http://www1.test.spos; #jmeno stranky, kterou chceme balancovat
	       }
 	}
	#weigh - server s wight=3 bude prebirat 3x vice trafficu nez server s weight=1
	#max_fails - kolikrat muze selhat pripojeni k danemu serveru nez ho ngnix prohlasi za offline
	#fail_timeout - udava, po jak dlouhych intervalech se bude nginx snazit pripojit k serveru, ktery je down
ln -s /etc/nginx/sites-available/www.conf /etc/nginx/sites-enabled/ - přidá stránku do sites-enabled
```

Maily
==============================================================================

* `apt-get install postfix`
* `apt-get install mailutils`
* `apt-get install mutt`
* `apt-get install dovecot-imapd dovecot-pop3d`

```bash
poslání mailu:
	echo "Obsah mailu" | mail -s "Subject" user@test.spos
	echo "Obsah mailu" | mail -s "Subject" user@test.spos -a "From: odesilatel@domena.spos"
```

* `mutt -f Mailbox` - umožní prohlížení mailů, Mailbox je cesta k mailboxu nebo maildiru

```bash
/etc/postfix/main.cf
	mynetworks = ... #z jakych siti muze chodit posta
	myhostname = test.spos #domena pro mail
	mydestination = test.spos, localhost
	mailbox_command = #musi byt prazdne
	home_mailbox = Maildir/ #posta bude chodit do adresare Maildir v home adresari daneho uzivatele
	(home_mailbox = Mailbox #pokud na konci neni lomitko, jedna se o mailbox, nikoliv maildir)
```

* `usermod -G mail user` - každý uživatel, kterému má být možno poslat mail, musí být ve skupině mail

!!!! pokud je v souboru `/etc/aliases` řádek typu root: user, pak pošta pro roota bude chodit uživateli user a bude v jeho Maildiru !!!!
po modifikaci `/etc/aliases` je potřeba použít příkaz newaliases

```bash
Modifikace DNS pro maily (domena test.spos, adresa 192.168.0.104):
mail    IN      A       192.168.0.104
@       IN      MX 10   mail
```

Virtuální domény:
```bash
/etc/postfix/main.cf
	virtual_alias_domains_map = hash:/etc/postfix/virtual_domains #muze byt i jiny soubor
	virtual_alias_maps = hash:/etc/postfix/virtual #muze byt i jiny soubor
```

```bash
/etc/postfix/virtual_domains
	domena1.spos ok
	domena2.spos ok
	domena3.spos ok
	... #seznam virualnich domen
```
```bash
/etc/postfix/virtual
	user1@domena1.spos user1
	user2@domena2.spos user2
	user3@domena3.spos user3
	... #rika, jaky mail ma jaky uzivatel
```

* `postmap /etc/postfix/virtual_domains` - aplikování změn
* `postmap /etc/postfix/virtual` - aplikování změn

* potřeba přidat zóny pro všechny virtuální domény do DNS

Přečtení pošty přes IMAP, telnet:

```bash
/etc/dovecot/conf.d/10-mail.conf
	mail_location = mbox:~/mail:INBOX=~/Mailbox #za INBOX= nasleduje cesta k mailboxu
	(mail_location = maildir:~/Maildir #pro Maildir pouze cesta k maildiru)
```

```bash
/etc/dovecot/conf.d/10-logging.conf
	log_path = /var/log/dovecot.log
```

```bash
~/.muttrc:
set folder = imap://jmeno:heslo@localhost:143/
set spoolfile = imap://jmeno:heslo@localhost:143/
```

* `chmod 777 .muttrc`

```bash
telnet localhost 143
1	LOGIN	user	pass
2	LIST	""	"*"
3	STATUS	INBOX	(MESSAGES)
4	EXAMINE	INBOX
5	FETCH	1	BODY[]	- misto 1 bude ID dane zpravy
6	LOGOUT
```

Čtení pošty šifrovaně přes IMAP (nebo POP3), mutt:
* `10-mail.conf` - stejně jako pro telnet 
* `cd /etc/dovecot`
* `mkdir certs`
* `cd certs`
* `openssl genrsa -des3 -out server.key 1024`
* `openssl req -new -key server.key -out server.csr`
* `cp server.key server.key.org`
* `openssl rsa -in server.key.org -out server.key`
* `openssl x509 -req -days 365 -in server.csr -signkey server.key -out server.crt` - vytvoří soubory server.crt a server.key (stejné jako pro SSL u webu)

```bash
/etc/dovecot/conf.d/10-ssl.conf
	ssl = yes
	ssl_cert = </etc/dovecot/certs/server.crt
	ssl_key = </etc/dovecot/certs/server.key
```

```bash
~/.muttrc (v home adresáři uživatele, ze kterého se přihlašujeme)
	- pro imap
	set folder = imap://user:pass@localhost:143/
	set spoolfile = imap://user:pass@localhost:143/
	- pro pop3
	set folder = pop://user:pass@localhost:110/
	set spoolfile = pop://user:pass@localhost:110/
```

* `user` - uživatel, na kterého se přihlašujeme
* `mutt` - spustit bez parametrů	

Přidání virtuálních mailů s virtuálními uživateli
--------------------------------------------------

* `addgroup --gid 1003 vmail`
* `adduser --gid 1003 --uid 1003 vmail`

```bash
/etc/postfix/main.cf:

	#pro maildir
	home_mailbox = Maildir/
	mailbox_command =

	#define how many virtual domains you want
	virtual_mailbox_domains = domena.spos, ...dalsi

	#tell the postfix to use the new mail base
	virtual_mailbox_base = /var/spool/mail

	virtual_mailbox_maps = hash:/etc/postfix/virtual

	virtual_uid_maps = static:1003
	virtual_gid_maps = static:1003
```

```bash
/etc/postfix/virtual:
	info@domena.spos	domena.spos/info
```


* `mkdir /var/spool/mail/domena.spos`
* `chmod 777 /var/spool/mail/domena.spos`

* `postmap /etc/postfix/virtual`

domena domena.spos:
```bash
/etc/bind/named.conf.local:

zone "domena.spos" {
	type master;
	file "/etc/bind/db.domena.spos";
	allow-transfer {"any";};
};
```

```bash
/etc/bind/db.domena.spos:
$TTL	604800
@	IN	SOA	domena.spos. root.localhost. (
			      3		; Serial
			 604800		; Refresh
			  86400		; Retry
			2419200		; Expire
			 604800 )	; Negative Cache TTL
;
@	IN	NS	domena.spos.
@	IN	A	192.168.0.104
	IN	MX	6	domena.spos.
```
* `service bind9 restart`
* `service postfix restart`


spatny sendername:
--------------------------------------------------

* `vim /etc/postfix/sender_canonical`
* `sender@oldomain.nx  sender@newexistingdomain.com`

* `postmap /etc/postfix/sender_canonical`


* `vim /etc/postfix/main.cf:`
* `sender_canonical_maps = hash:/etc/postfix/sender_canonical`

* `service postfix reload`


Databáze [MYSQL]
==============================================================================

MySql:
* `apt-get install mysql-server`

* `mysql -u root -p` - přihlášení do databáze pod uživatele root
* `mysql -u root -p -h 192.168.0.100 -P 3309` - přihlášení z jiného stroje na 192.168.0.100:3309

```bash
~/.my.cnf - obsahuje defaultní přihlašovací údaje, které jsou použity pokud je příkaz mysql použit bez parametrů
	[client]
	user=user
	password=pass
```

```bash
/etc/mysql/my.cnf
	bind-adress = 10.0.0.1 #adresa na které mysql poslouchá
	[mysqld]
	port = 1234    #pro port pod 1024 musí být ještě user = root
	[client]
	port = 1234
```

```sql
mysql> SHOW DATABASES; - ukáže všechny databáze, na které má daný uživatel práva
mysql> CREATE USER 'db01'@'localhost' IDENTIFIED BY 'password'; - vytvoří uživatele db01 s heslem password, pokud místo 'locahost' '%', pak lze k uživateli připojit odkudkoliv se stejnými právy, také lze síť: '192.168.0.0/255.255.0.0'
mysql> CREATE DATABASE db DEFAULT CHARACTER SET utf8 DEFAULT COLLATE utf8_general_ci; - stačí CREATE DATABASE db; - vytvoří databázi db
mysql> use db; - přepne na databázi db
mysql> GRANT ALL PRIVILEGES ON db.* to 'db01'@'localhost' identified by 'password'; - přidá všechna práva na databázi db uživateli db01 (resp. 'db01'@'localhost'), * - platí pro všechny tabulky (lze specifikovat)
mysql> GRANT SELECT, INSERT, UPDATE, DELETE ON db.* to 'db01'@'localhost' identified by 'password';
mysql> CREATE TABLE table01(id INT(6) UNSIGNED AUTO_INCREMENT PRIMARY KEY, firstname VARCHAR(30) NOT NULL); - vytvoří tabulku table01
mysql> insert into table01 (firstname) values ('Pepa');
```

```sql
#tabulka z vystupu ke cviceni k mysql:
CREATE TABLE table01(id INT(6) UNSIGNED AUTO_INCREMENT PRIMARY KEY, name VARCHAR(30) NOT NULL, description TEXT NOT NULL, created_at timestamp default now() NOT NULL, price FLOAT(9,4) NOT NULL);
```

```bash
#script na generovani pro mysql, db ve scriptu je databaze
#!/bin/bash
for i in `seq 1 50`; do
	echo "INSERT INTO table01 (name, description, price) VALUES ('name"$i"', 'desc"$i"', '"$i"');"
done | mysql db
```

```sql
#prava uzivatelovi na konkretnim rozsahu ip
GRANT ALL PRIVILEGES ON database.* TO 'user'@'81.10.20.1/255.255.255.240' [IDNETIFIED BY 'password'];
```
```sql
změna hesla uživatele:
mysql> use mysql;
mysql> update user set password=PASSWORD('heslo') where user='root'; - změna hesla uživatele root
```

* `mysqldump spos -u root --password='heslo'` - zálohuje databázi spos pod uživatelem root
* `mysql --user=db01 --password=password db -Bse "INSERT INTO... "` - spustí příkaz v databázi db pod uživatelem db01 z terminálu

Přpojení z PHP:
* `apt-get install php5-mysql`
* `apt install php-mysqli`


index.php:
```php
<?php
$servername = "localhost";
$username = "db01";
$password = "password";
$dbname = "db";

// Create connection
$conn = new mysqli($servername, $username, $password, $dbname);    //za dbname muze byt jeste port jako int
// Check connection
if ($conn->connect_error) {
    die("Connection failed: " . $conn->connect_error);
}

$sql = "SELECT id, firstname from table01";
$result = $conn->query($sql);

if ($result->num_rows > 0) {
    // output data of each row
    while($row = $result->fetch_assoc()) {
        echo "id: " . $row["id"]. " - Name: " . $row["firstname"]. " " . "<br>";
    }
} else {
    echo "0 results";
}
$conn->close();
?>
```
_________________________________________
upraveny script podle output ze cviceni

```php
<?php
$servername = "localhost";
$username = "db01";
$password = "heslo";
$dbname = "db";

$conn = new mysqli($servername, $username, $password, $dbname);

if ($conn->connect_error) {
    die("Connection failed: " . $conn->connect_error);
}

else {
	echo "conn ok";
}

     $sql = "SELECT id, name, description, created_at, price FROM table01";

     if(isset($_GET["sort"]))
	     $sql = "SELECT id, name, description, created_at, price FROM table01 ORDER BY price DESC";

     $result = $conn->query($sql);

     if ($result->num_rows > 0) {
             while($row = $result->fetch_assoc()) {
                     echo "id: " . $row["id"]. " - Name: " . $row["name"]. " - Description: " . $row["description"] . " - Created at: " . $row["created_at"] . " - Price: ". $row["price"] . "<br>\n";
     	     }
     } else {
	echo "0 results";
     }
       $conn->close();
?>
```

___
Dump script:
```bash
#!/bin/bash

mysqldump -u root -pspos. spos > /mnt/db.dump
```
___





Pgsql:
==============================================================================
* `apt-get	install	postgresql-9.4`
* `su - postgres`

```bash
/etc/postgresql/9.4/main/pg_hba.conf
	host    db              db01            192.168.0.0/16          md5 - lze se přihlásit na uživatele db01 pro databázi db ze sítě 192.168.0.0/16
	local    db              db01                                   md5 - lze ----------------------- || ------------------ z localhostu přes psql - dát na začátek
```

* `service postgresql restart`

* `psql -U postgres` - přihlášení do postgresu

```sql
postgres=# CREATE USER db01 WITH PASSWORD 'password';
postgres=# CREATE DATABASE db WITH OWNER=db01;
postgres=# \l - list databází
postgres=# \c db - přepnout do databáze db
db=# CREATE TABLE table01(id SERIAL PRIMARY KEY, firstname VARCHAR(30) NOT NULL);

CREATE TABLE table01(id SERIAL PRIMARY KEY, name VARCHAR(30) NOT NULL, description TEXT NOT NULL, created_at timestamp default now() NOT NULL, price FLOAT(8) NOT NULL);

db=# GRANT ALL PRIVILEGES ON DATABASE db to db01; - všechna práva pro uživatele db01 na databázi db
db=# GRANT SELECT, INSERT, UPDATE, DELETE ON DATABASE db to db01; - specifikována jen některá práva
db=# GRANT SELECT ON TABLE table01 to db01; - práva nad konkrétní tabulkou
GRANT USAGE, SELECT ON ALL SEQUENCES IN SCHEMA public to www; #prava aby slo pridavat aby se inkrementovalo id
db=# insert into table01 (firstname) values ('Pepa');
postgres=# \q - konec
```

byt ve spravne db a potom:
```sql
GRANT SELECT ON ALL TABLES IN SCHEMA public TO spos_test;
GRANT SELECT ON ALL SEQUENCES IN SCHEMA public TO spos_test;
GRANT ALL ON ALL TABLES IN SCHEMA public TO spos_test;
GRANT ALL ON ALL SEQUENCES IN SCHEMA public TO spos_test;
```

po vytvoreni db:
`GRANT ALL PRIVILEGES ON DATABASE db01 to db01;`

potom ve spravne db:

```sql
SQL> GRANT CONNECT ON DATABASE dbname TO username;
SQL> GRANT ALL PRIVILEGES ON TABLE tabulka TO user1;
grant usage on all sequences in schema public to spos_test;
```

Připojení z PHP:
* `apt-get install php5-pgsql`
* `service apache2 restart`


```bash
pg_hba.conf:

local   all             postgres                                peer

# TYPE  DATABASE        USER            ADDRESS                 METHOD
local	all		db01					md5
local	all		root					peer
local 	spos_test	spos_test				md5
local	spos_test	spos_ro					md5
host	spos_test	spos_ro		192.168.255.0/24	md5
host	spos_test	spos_test	::1/128			md5
host 	spos_test	spos_ro		::1/128			md5
```


insert script:
--------------------------------------------------

```bash
#!/bin/bash

for i in `seq 1 10`; do
	echo "INSERT INTO table01 (name, description, price) VALUES ('name"$i"', 'desc"$i"', '"$i"');"
done | psql spos_test -U spos_test
```
--------------------------------------------------

```php
index.php:
<?php
$dbconn = pg_connect("host=localhost dbname=db user=db01 password=password")
    or die('Could not connect: ' . pg_last_error());

$query = 'SELECT * FROM table01';
$result = pg_query($query) or die('Query failed: ' . pg_last_error());

echo "<table>\n";
while ($line = pg_fetch_array($result, null, PGSQL_ASSOC)) {
    echo "\t<tr>\n";
    foreach ($line as $col_value) {
        echo "\t\t<td>$col_value</td>\n";
    }
    echo "\t</tr>\n";
}
echo "</table>\n";

pg_free_result($result);

pg_close($dbconn);
?>
```

* `alter user postgres with password 'heslo';` - dump


```bash
#!/bin/bash

pg_dump --dbname=postgresql://postgres:heslo@localhost:5432/spos_test > zaloha.sql
```

Sdílení:
==============================================================================

NFS:
* `apt-get install nfs-server`

chmod na sdilene slozky v `/srv/` a i na pripojne misto v `/mnt/` !!!!

```bash
/etc/exports
	/srv/share1	*(rw) #pro všechny čtení i zápis
	/srv/share2     192.168.0.0/16(ro) #pro síť 192.168.0.0/16 pouze čtení
	/srv/share3     192.168.0.0/16(ro,no_root_squash) #uživatel nebude připojen jako root
	/srv/share4     192.168.0.0/16(rw,no_subtree_check,all_squash,anonuid=6000,anongid=6000) #připojený uživatel bude připojen jako user s UID a GID 6000
												 #musí být přidán do skupiny:
												 #addgroup --gid 6000 nfs
												 #adduser --gid 6000 --uid 6000 nfs
												 #pro roota anonuid=0, anongid=0
```

`service nfs-kernel-server restart`
`chmod 777 share1`
`chmod 777 share2`
`chmod 777 share3`
...

Ze stroje na který sdílím:
`mount -t nfs 10.228.67.42:/srv/share1 /mnt/share1`

```bash
/etc/fstab
	192.168.56.101:/srv/share1	/mnt/share1	nfs	defaults	0	0
```

Samba:
==============================================================================
* `apt-get install samba smbclient cifs-utils` - pokud otevře manuál, dát q

* `smbpasswd -a pepa` - přidá uživatele pepa (musí být v systému) !!!!!

* `addgroup --gid 6001 cifs`
* `usermod -G cifs pepa`

```bash
/etc/samba/smb.conf
[share1]
	comment = Prvni share co jsem kdy vyrobili
        path = /srv/share1
        browsable = yes
        writable = yes  #neni read-only, alternativne read only = yes místo writable = yes
        guest ok = yes  #nebude chtit heslo
        create mask = 0600 #maska prav souboru
        directory mask = 0700 #maska prav adresaru
	valid users = vita #kdo smi sdilet
	hosts allow = 127.0.0.1 192.168.2.0/24 192.168.3.0/24 #pro jake site bude sdileni fungovat
    		hosts deny = 0.0.0.0/0

- znovu bez komentářů - lze celé zkopírovat
[share1]
	comment = muj share
        path = /srv/share1
        browsable = yes
        writable = yes
        guest ok = yes
        create mask = 0777
        directory mask = 0777
	valid users = user
	hosts allow = 127.0.0.1 192.168.2.0/24 192.168.3.0/24
    		hosts deny = 0.0.0.0/0
```
* `smbtree` - ukáže co se sdílí

Ze stroje na který sdílím:
`mount -t cifs //192.168.0.1/share1 /mnt/share -o username=pepa,pass=pepa` - share1 je název sharu

```bash
/etc/fstab
	//192.168.56.101/share1	/mnt/share1	cifs	defaults,username=pepa,pass=pepa	0	0
```

Portsentry
==============================================================================

* `apt-get install portsentry` - kontrola skenovani portu jen pro tcp a nastaveni blokace:

```bash
/etc/portsentry/portsentry.conf:
	BLOCK_UDP="0"
	BLOCK_TCP="1"
```

```bash
/etc/default/portsentry:
	TCP_MODE="tcp"
	#UDP_MODE="udp"
```

* `service portsentry restart`

z jiného stroje:
* `nmap -A 147.228.67.XX` - pro TCP
* `nmap -A -sU 147.228.67.XX` - pro UDP

odblokování:
* `cd /var/lib/portsentry`
* `echo > portsentry.history`
* `echo > portsentry.blocked.tcp (pro UDP echo > portsentry.blocked.udp)`
* `ip route del 147.228.63.10`
smazat příslušnou řádku z `/etc/hosts.deny`


========================================================================================
https://docs.google.com/document/d/1FqpkcpNQ2D_qpxtNHkaDrL5fKAU2qmzZShi-5V7bmWg/edit?fbclid=IwAR0bsq8f_7ANaPc1hRDc1nCUIEam8FXloBQMRwONR5QbvFAhQjJVyuEKz1Q

DNS
--------------------------------------------------

Přidání záznamu do /`etc/hosts`
```bash
nano /etc/hosts
	127.0.0.2 	test.spos
```
* `apt install bind9` - Instalace nástroje bind9
* `system bind9 restart` nebo `systemctl restart bind9.service` - Restartování bind9
* `ps aux | grep bind ` - Kontrola, jestli bind9 běží
* `tail -n 50 /var/log/syslog` - Zobrazení posledních řádků systémových logů
* `cd /etc/bind` - Přesun do adresáře bind9
* `cp db.empty db.test.spos` - Vytvoření kopie souboru podle šablony

Úprava souboru db.test.spos
```bash
nano db.test.spos
	test.spos.	IN	A		127.0.0.2
	mail.test.spos.	IN	A		127.0.0.2
	test.spos. 	IN 	NS 		test.spos.
			IN	MX	6	mail.test.spos.
```

Vytvoření zóny
```bash
nano named.conf.local
zone "test.spos" {
	type: master;
	file "/etc/bind/db.test.spos";
	allow-transfer {"any";};
};
```
* `host -t NS test.spos localhost` -Ověření funkčnosti


Nastavení poštovního serveru
--------------------------------------------------
* `apt install postfix bsd-mailx` - Instalace balíčku postfix a bsd-mailx

```bash
Postfix Configuration
	Internet Site
	System mail name: mail.test.spos
```

Úprava nastavení `/etc/postfix/main.cf`

```bash
nano /etc/postfix/main.cf
	myhostname = mail.test.spos
```

* `service postfix reload` - Restartování služby po úpravě main.cf

Přidání uživatelů a nastavení aliasů
--------------------------------------------------
Vypsání existujících uživatelů
`cat /etc/passwd`
Přidání uživatelů user1 a user2
`useradd user1 && useradd user2`
Nastavení hesla pro uživatele user1
`passwd user1`
Nastavení aliasů
```bash
nano /etc/aliases
	hostmaster: user1
	webmaster: user2
```
Načtení aliasů po změně /etc/aliases
`newaliases`


Odeslání a čtení e-mailů 
--------------------------------------------------
Odeslání pošty na hostmaster@mail.test.spos
`echo "TELO ZPRAVY" | mail -s "PREDMET" hostmaster@mail.test.spos`
Instalace mailového klienta mutt
`apt install mutt`
Otevření pošty v nástroji mutt
`mutt -f /var/spool/mail/user1`

Stahování e-mailů
--------------------------------------------------
* Instalace balíčku dovecot pro stahování pošty
	* `apt install dovecot-imapd` - Protokol IMAP (modernější)
	* `apt install dovecot-pop3d` - Protokol POP3
	
Vytvoření domovského adresáře pro uživatele user1
```bash
mkdir /home/user1
chown user1 user1
```
* `usermod -g mail user1` - Přidání oprávnění pro mail uživateli user1
* `nc localhost 143` - Spojení IMAP
	* `a login user1 heslo` - Přihlášení se jako user1
	* `a list "" "*"` - Seznam
	* `a status inbox (messages)` - Počet zpráv ve schránce
	* `a examine inbox` - Podrobnější přehled (počet zpráv, nedávné, nepřečtené)
	* `a fetch 1 body[]` - Načíst první zprávu
	* `a logout` - Odhlášení se

Webový server s podporou PHP5
--------------------------------------------------

`lsof -i :80` Výpis procesů, které využívají port 80 (služba HTTP)

```bash
lighttpd	  545   www-data   4u   IPv4   12760   0t0   TCP   *:http (LISTEN)
# Na stroji je již nainstalován www server lighttpd.
```

* Přidání starších zdrojů s balíčkem php5
```bash
nano /etc/apt/sources.list
	deb http://ftp.cz.debian.org/debian/ jessie main
```

`apt update` - Aktualizace zdrojů softwaru po změně sources.list

`apt install php5 php5-cgi` - Instalace balíčků php5

Vytvoření soubor index.php
1. `cd /var/www/html` - Přesun do adresáře
2. `mv index.lighttpd.html index.html.backup` - Záloha stávajícího indexu
3. `echo "<h1>BEZI</h1><?php phpinfo(); ?>" > index.php` - Vytvoření index.php
4. `lighttpd-enable-mod fastcgi fastcgi-php` - Povolení PHP
5. `service lighttpd force-reload` - Restartování www serveru

* Vypsání stránky
* Pomocí curl do terminálu
	* `curl test.spos` - 

* Pomocí prohlížeče links
	* `apt install links`
	* `links http://test.spos`

* Zákaz funkce phpinfo()
	* `nano /etc/php5/cgi/php.ini`
	* `disable_functions = phpinfo`
* Restartování www serveru
	* `service lighttpd force-reload`
* Vypsání stránky
	* `curl test.spos`

Muj test
===========================================
netstat -tulpn \| grep :80
Nastavení raidu 5 ze všech dostupných disků
------------------------

* `lsblk` - Ukaze vsechne disky

```bash
sdb    	8:16	0    	200M	0  	disk
sdc    	8:32	0    	200M  	0 	disk
sdd    	8:48	0		200M  	0  	disk
```

* `apt install mdadm` - Instalace balíčku mdadm

* `mdadm -Cv /dev/md0 -l5 -n3 /dev/sd[bcd]` - Založení raidu 5

* `mdadm --detail /dev/md0` - Vypíše stav raidu (kapacita, typ, počet zařízení…)

Naformátování svazku jako EXT4 a automatické připojení jako /home
-----------------------------------------------------------------

* `mkfs.ext4 /dev/md0` - Naformátování share na ext4

* `nano /etc/fstab` - Nastavení připojení share jako /home

```bash
/dev/md0	/home	ext4	defaults		0	2
```

* `mount -a` - Připojení všech oddílů z fstab

* `lsblk -f` - Vypsání diskových oddílů včetně použitých FS a přípojných bodů

MySql
-----

* `apt-get install mysql-server`

1. Vytvorit tabulku "score". [id], [jmeno], [datum], [score]

```sql
CREATE TABLE score( 
	id INT(6) UNSIGNED AUTO_INCREMENT PRIMARY KEY, 
	jmeno VARCHAR(30), 
	score INT(10), 
	datum DATETIME 
);

```

2. Vytvoreni uzivatele "spos" s heslem "spos\*2018", krery bude mit vsechna prava na novou databazi "db" a bude se moct pripojit pouze z 10.0.0.41

```sql
CREATE USER 'spos'@'10.0.0.41' IDENTIFIED BY 'spos\*2018';

GRANT ALL PRIVILEGES ON db.* to 'spos'@'10.0.0.1' IDENTIFIED BY 'spos*2018';
```
3. Naplnit tabulku 30 daty

```bash
#!/bin/bash
for i in `seq 1 50`; do
	echo "INSERT INTO score (jmeno, score, datum) VALUES ('name"$i"', '"$i"', null);"
done | mysql -u root -p1234 db
```
Web server
----------

1. Nainstalujte WEB server s podporou php

* `apt-get install apache2` 
* `apt-get install php5`
* `apt-get install libapache2-mod-php5`
* `apt-get install curl`

2. Zmenit port WEBu na 8080

* `vim /etc/apache2/ports.conf` - Zmenit Listen

```bash
Listen 8080
#Listen 192.168.1.101:8090

<IfModule ssl_module>
        Listen 443
</IfModule>

<IfModule mod_gnutls.c>
        Listen 443
</IfModule>                                                    
```

* `vim /etc/apache2/sites-enabled/000-default.conf` - Zmenit VirtualHost

```bash
<VirtualHost *:8080>
```

* `service apache2 restart`
* `netstat -tulpn | grep :8080` - kontrola

3. Vyrvorit script php, ktery bude zobrazovat obsah tabulky sestupne

```php
<?php
$servername = "localhost";
$username = "root";
$password = "1234";
$dbname = "db";

$conn = new mysqli($servername, $username, $password, $dbname);    
if ($conn->connect_error) {
    die("Connection failed: " . $conn->connect_error);
}

$sql = "SELECT * from score ORDER BY score DESC";
$result = $conn->query($sql);

if ($result->num_rows > 0) {
    while($row = $result->fetch_assoc()) {
        echo "id: " . $row["id"]. " - Name: " . $row["jmeno"]. " " . "\n";
    }
} else {
    echo "0 results";
}
$conn->close();
?>
```
* `vim /etc/apache2/mods-available/dir.conf` - pridat index.php na zacatek
* `apt-get install php5-mysql`
* `apt-get install php-mysqli`
* `curl localhost:8080`

4. Vytvorit 2 virtualni servery na portech 8181 a 8282, poslouchajici na privatni adrese 10.0.0.41

* `/etc/apache2/ports.conf`

```bash
<VirtualHost *:8181>
        DocumentRoot "/var/www/www1"
        <Location />
                Require ip 10.0.0.41
        </Location>
</VirtualHost>

<VirtualHost *:8282>
        DocumentRoot "/var/www/www2"
        <Location />
                Require ip 10.0.0.41
        </Location>
</VirtualHost>

```

* `cp 000-default.conf www1.conf`
* `cp 000-default.conf www2.conf`
* `vim www1.conf `
* `vim www2.conf `
* `service apache2 restart`
* `mkdir /var/www/www1`
* `mkdir /var/www/www2`
* `echo "8181" > /var/www/www1/index.html`
* `echo "8282" > /var/www/www2/index.html`
* `a2ensite www1`
* `a2ensite www2`
* `ip a add 10.0.0.41 dev eth0`


systemctl status nginx
systemctl restart nginx
nginx -t -kontrola
lynx -dump http://10.0.0.41

NGinx reverzni-proxy / load balanceru s podporou https.
-----------------------------------------------------------------------------------

1. Zpistupnete oba weby pomoci NGinx reverzni-proxy / load balanceru s podporou https.

* `apt-get install nginx`
* `touch /etc/nginx/sites-available/www.conf`
* `vim /etc/nginx/sites-available/www.conf`
```bash
upstream 10.0.0.41 {
    server  10.0.0.41:8181;
    server  10.0.0.41:8282;
}

server {
               listen  80; #port, na kterem ma nginx poslouchat
               server_name 10.0.0.41; #jmeno stranky, kterou chceme balancovat

                location / {
                       proxy_pass http://10.0.0.41;
                }
}           
```

* `ln -s /etc/nginx/sites-available/www.conf /etc/nginx/sites-enabled/`
* `systemctl enable nginx`
* `systemctl restart nginx`
* `systemctl status nginx.service`
* `nginx -t`


===================================================================================================

1. Nastavte v Linuxu sdileni slozky /mnt pro Linux i Windows, sdileni ad je viditelne pod nazvem TEST a je dostupne pouze pro cteni. Zabespeceni at kontroluje server.

* `apt-get install samba smbclient cifs-utils` instalace Samby

* `vim /etc/samba/smb.conf`

```bash
[TEST]
	comment = Prvni share co jsem kdy vyrobili
    path = /srv/share1
    browsable = yes
    read only = yes 
    guest ok = yes  
```

* `smbtree` - kontrola

2. Pripojte ten to svazek do linuxu (pro pripojeni IP 10.0.0.41), tak aby byl pripojen i po startu.
* `mount -t cifs //10.0.0.41/TEST /mnt/share -o username=pepa,pass=pepa`
....

3. Nainstalujte PostgreSQL databazi. Zajistete, aby databaze pouzivala pouze IP 10.0.0.41 a TCP port 3333.
* `apt-get	install	postgresql`


4. Vytvorte databazi spos a nastavte uzivatele spos, ktery s heslem ahoj bude mit pristup do teto Databazi pres IP 10.0.0.41

5. Tabulka datum se sloupecky id a datum. Vytvorte script, ktery po zavolani bez zasahu uzivatele vlozi do tabulky udaj o aktualnim case.

6. Na adrese 10.0.0.41 a portech 8888 a 8080 spuste Apache server, zobrazujici stranku s cislem portu.

7. Pomoci reverzni proxy / balanceru zpristupnete obsah WWW serveru na verejne IP linuxu a portu 443/https pouzijte self-signed certifikat.

8. Nastavte DNS pro domenu test.spos tak, aby z interni site 10.0.0.0/24 ukazovala IP adresu 10.0.0.41 a pro ostatni ukazovala verejnou IP. Nastavte reverzni zaznamy pro obe IP adresy a TXT zaznam oznamujici bla bla bla..
