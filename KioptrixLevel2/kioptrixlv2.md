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

\```
nmap 192.168.56.0/24
\```
So i could discover which hosts were available

\```
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
\```
With mysql and http servers open, this host was most likely my tartget machine

\```
nmap -sS -sV -p- -T3 192.168.56.103 
\```
So i could get deeper information on the taget IP's ports and services
