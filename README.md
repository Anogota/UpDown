Pierwszy krok to nmap
```
PORT   STATE SERVICE VERSION
22/tcp open  ssh     OpenSSH 8.2p1 Ubuntu 4ubuntu0.5 (Ubuntu Linux; protocol 2.0)
| ssh-hostkey: 
|   3072 9e1f98d7c8ba61dbf149669d701702e7 (RSA)
|   256 c21cfe1152e3d7e5f759186b68453f62 (ECDSA)
|_  256 5f6e12670a66e8e2b761bec4143ad38e (ED25519)
80/tcp open  http    Apache httpd 2.4.41 ((Ubuntu))
|_http-server-header: Apache/2.4.41 (Ubuntu)
|_http-title: Is my Website up ?
Service Info: OS: Linux; CPE: cpe:/o:linux:linux_kernel
``` 
Nic nie widzimy tutaj ciekawego. Wejdz na stronie aby zobaczy co tam sie znajduje.

![image](https://github.com/Anogota/UpDown/assets/143951834/91b4a5eb-6a57-4cbf-b6e8-8538b4022bf4)

Widzimy tutaj domene, musimy ja zapisac echo "10.10.11.177 siteisup.htb" >> /etc/hosts
Nastpnym krokiem bede vhosty, uzyjemy gobuster
```
gobuster vhost -w /usr/share/seclists/Discovery/DNS/shubs-subdomains.txt -u http://siteisup.htb/
``` 
Znalazlem tylko jedna subdomene. ```Found: dev.siteisup.htb (Status: 403) [Size: 281]``` Lecz nie mozemy wejsc, jest 403.
Nastepnie uzyejmy gobuster dir, aby znalezc ciekawe katalogi.
```
/.hta                 (Status: 403) [Size: 277]
/.htpasswd            (Status: 403) [Size: 277]
/.htaccess            (Status: 403) [Size: 277]
/dev                  (Status: 301) [Size: 310] [--> http://siteisup.htb/dev/]
/index.php            (Status: 200) [Size: 1131]                              
/server-status        (Status: 403) [Size: 277]
```
Mamy cos ciekawego jest dev, ale po wejsciu na strone jest ona cala biala wiec postanowilem jeszcze raz poszukac katalogow na /dev i byl to bardzo dobry wybor.
```
/.hta                 (Status: 403) [Size: 277]
/.git/HEAD            (Status: 200) [Size: 21] 
/.hta.php             (Status: 403) [Size: 277]
/.htaccess            (Status: 403) [Size: 277]
/.htpasswd            (Status: 403) [Size: 277]
/.htaccess.php        (Status: 403) [Size: 277]
/.htpasswd.php        (Status: 403) [Size: 277]
/index.php            (Status: 200) [Size: 0]  
/index.php            (Status: 200) [Size: 0]
```
Musimy uzyc narzedia git dumper.
```git-dumper http://siteisup.htb/dev/.git/ /dev```
Dzieki temu zrzucilo nam caly folder dev.

![image](https://github.com/Anogota/UpDown/assets/143951834/02893b1a-0ece-4387-8db9-593869e1fbe0)

Widzimy tam ciekawe pliki.
W changelog.txt widzimy cos ciekawego, co moze wskazywac na RCE

![image](https://github.com/Anogota/UpDown/assets/143951834/90ff3edc-700c-4b00-af91-0f96dea09692)
