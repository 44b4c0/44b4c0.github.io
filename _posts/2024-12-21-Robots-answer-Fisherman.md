---
title: "რობოტები მეთევზის წინააღმდეგ"
date: 2024-12-21 17:24:00 +0400
categories: [CyberSecurity,Hacking]
tags: [Malware]
---

## დასაწყისი

წინა ბლოგში განვიხილე `Full Attack Chain`, რომელიც საკმაოდ კარგად იყო გათვლილი თითქმის ყველა კუთხით. ახლა ვეცდები ამ ინფრასტრუქტურის უსაფრთხოება შევამოწმო და ვნახო თუ რას ჰოსტავს გუნდი(ის დაჯგუფება, რომელიც მართავდა ამ ფიშინგ კამპანიას).

## Power up the machine!

ჩავრთე ჩემი ვირტუალური მანქანა და მზად ვარ დავიწყო ჰაკერების წინააღმდეგ მოქმედება. მოკლედ, როგორც წინა პოსტში ვთქვი, მოცემულია რამდენიმე დომენი:

1. hxxps://saaadnesss.shop
2. hxxps://polovoiinspektor.shop

`nslookup` ბრძანებით, შესაძლებელია იმ IP მისამართების ამოღება, რომლებიც დგანან ამ დომენების უკან.

![1.png](https://44b4c0.github.io/assets/img/posts/23/1.png)

საინტერესოა, ერთი და იგივე IP მისამართია. ანუ გამოდის, რომ ერთ IP მისამართზე ორი დომენია რეგისტრირებული. დაახლოებით ასეთი რამ გამოდის:

```
   ┌───────────────┐                            
   │saaadnesss.shop├─────┐                      
   └───────────────┘     │     ┌───────────────┐
                         ├─────►185.121.235.167│
┌─────────────────────┐  │     └───────────────┘
│polovoiinspektor.shop├──┘                      
└─────────────────────┘                         
```

კარგი, მაშინ ავდგები და `nmap`-ს გამოვიყენებ ამ IP მისამართის დასასკანირებლად:

```bash
nmap -sC -sV -A 185.121.235.167 -T5 > nmapScan.txt
```

გასათვალისწინებელია ის ფაქტი, რომ `GodResponsibility.exe` ვირუსი აღარ იძებნება გუშინდელი ამბის შემდეგ(წინა პოსტს ვგულისხმობ). ეს იმიტომ, რომ ზოგადად, ჰაკერები ხშირად აკვირდებიან თავიანთ ინფრასტრუქტურას და თუ დააფიქსირეს ვინმე, რომელიც არაავტორიზებულად ცდილობს ზედ ძრომიალს, მაშინვე აჩერებენ ოპერაციას, რათა დაიცვან თავიანთი მონაპოვარი და `payload`-ები. ამის გამო ვეღარ ვახერხებ ამ ვირუსის შოვნას.

```
Starting Nmap 7.95 ( https://nmap.org ) at 2024-12-21 04:30 PST
Nmap scan report for v200070.hosted-by-vdsina.com (185.121.235.167)
Host is up (0.11s latency).
Not shown: 989 filtered tcp ports (no-response), 2 filtered tcp ports (host-unreach)
PORT     STATE SERVICE        VERSION
21/tcp   open  ftp            Microsoft ftpd
| ftp-syst: 
|_  SYST: Windows_NT
80/tcp   open  http           nginx 1.26.2
|_http-server-header: nginx/1.26.2
|_http-cors: HEAD GET POST PUT DELETE OPTIONS PATCH
|_http-title: 404 Not Found
135/tcp  open  msrpc          Microsoft Windows RPC
139/tcp  open  netbios-ssn    Microsoft Windows netbios-ssn
443/tcp  open  ssl/http       nginx 1.26.2
| ssl-cert: Subject: commonName=saaadnesss.shop
| Subject Alternative Name: DNS:ddeapeaceofmind.shop, DNS:minimeh.shop, DNS:polovoiinspektor.shop, DNS:saaadnesss.shop, DNS:uuukaraokeboss.shop
| Not valid before: 2024-11-24T21:14:14
|_Not valid after:  2025-02-22T21:14:13
|_http-cors: HEAD GET POST PUT DELETE OPTIONS PATCH
|_ssl-date: TLS randomness does not represent time
|_http-title: 404 Not Found
|_http-server-header: nginx/1.26.2
445/tcp  open  microsoft-ds?
3389/tcp open  ms-wbt-server  Microsoft Terminal Services
| rdp-ntlm-info: 
|   Target_Name: WIN-2513OKBPOH9
|   NetBIOS_Domain_Name: WIN-2513OKBPOH9
|   NetBIOS_Computer_Name: WIN-2513OKBPOH9
|   DNS_Domain_Name: WIN-2513OKBPOH9
|   DNS_Computer_Name: WIN-2513OKBPOH9
|   Product_Version: 10.0.20348
|_  System_Time: 2024-12-21T12:32:24+00:00
|_ssl-date: 2024-12-21T12:33:02+00:00; 0s from scanner time.
| ssl-cert: Subject: commonName=WIN-2513OKBPOH9
| Not valid before: 2024-11-17T17:13:30
|_Not valid after:  2025-05-19T17:13:30
5001/tcp open  commplex-link?
| fingerprint-strings: 
|   DNSStatusRequestTCP, DNSVersionBindReqTCP: 
|     HTTP/1.1 400 
|     content-length: 0
|     connection: close
|     date: Sat, 21 Dec 2024 12:31:33 GMT
|     server: hypercorn-h11
|   GenericLines, WMSRequest, ZendJavaBridge: 
|     HTTP/1.1 400 
|     content-length: 0
|     connection: close
|     date: Sat, 21 Dec 2024 12:31:26 GMT
|     server: hypercorn-h11
|   GetRequest, HTTPOptions: 
|     HTTP/1.1 404 
|     content-type: text/html; charset=utf-8
|     content-length: 207
|     vary: Origin
|     date: Sat, 21 Dec 2024 12:31:27 GMT
|     server: hypercorn-h11
|     Connection: close
|     <!doctype html>
|     <html lang=en>
|     <title>404 Not Found</title>
|     <h1>Not Found</h1>
|     <p>The requested URL was not found on the server. If you entered the URL manually please check your spelling and try again.</p>
|   RTSPRequest: 
|     HTTP/1.1 400 
|     content-length: 0
|     connection: close
|     date: Sat, 21 Dec 2024 12:31:27 GMT
|_    server: hypercorn-h11
5357/tcp open  http           Microsoft HTTPAPI httpd 2.0 (SSDP/UPnP)
|_http-title: Service Unavailable
|_http-server-header: Microsoft-HTTPAPI/2.0
1 service unrecognized despite returning data. If you know the service/version, please submit the following fingerprint at https://nmap.org/cgi-bin/submit.cgi?new-service :
SF-Port5001-TCP:V=7.95%I=7%D=12/21%Time=6766B51E%P=x86_64-unknown-linux-gn
SF:u%r(WMSRequest,73,"HTTP/1\.1\x20400\x20\r\ncontent-length:\x200\r\nconn
SF:ection:\x20close\r\ndate:\x20Sat,\x2021\x20Dec\x202024\x2012:31:26\x20G
SF:MT\r\nserver:\x20hypercorn-h11\r\n\r\n")%r(ZendJavaBridge,73,"HTTP/1\.1
SF:\x20400\x20\r\ncontent-length:\x200\r\nconnection:\x20close\r\ndate:\x2
SF:0Sat,\x2021\x20Dec\x202024\x2012:31:26\x20GMT\r\nserver:\x20hypercorn-h
SF:11\r\n\r\n")%r(GenericLines,73,"HTTP/1\.1\x20400\x20\r\ncontent-length:
SF:\x200\r\nconnection:\x20close\r\ndate:\x20Sat,\x2021\x20Dec\x202024\x20
SF:12:31:26\x20GMT\r\nserver:\x20hypercorn-h11\r\n\r\n")%r(GetRequest,17A,
SF:"HTTP/1\.1\x20404\x20\r\ncontent-type:\x20text/html;\x20charset=utf-8\r
SF:\ncontent-length:\x20207\r\nvary:\x20Origin\r\ndate:\x20Sat,\x2021\x20D
SF:ec\x202024\x2012:31:27\x20GMT\r\nserver:\x20hypercorn-h11\r\nConnection
SF::\x20close\r\n\r\n<!doctype\x20html>\n<html\x20lang=en>\n<title>404\x20
SF:Not\x20Found</title>\n<h1>Not\x20Found</h1>\n<p>The\x20requested\x20URL
SF:\x20was\x20not\x20found\x20on\x20the\x20server\.\x20If\x20you\x20entere
SF:d\x20the\x20URL\x20manually\x20please\x20check\x20your\x20spelling\x20a
SF:nd\x20try\x20again\.</p>\n")%r(HTTPOptions,17A,"HTTP/1\.1\x20404\x20\r\
SF:ncontent-type:\x20text/html;\x20charset=utf-8\r\ncontent-length:\x20207
SF:\r\nvary:\x20Origin\r\ndate:\x20Sat,\x2021\x20Dec\x202024\x2012:31:27\x
SF:20GMT\r\nserver:\x20hypercorn-h11\r\nConnection:\x20close\r\n\r\n<!doct
SF:ype\x20html>\n<html\x20lang=en>\n<title>404\x20Not\x20Found</title>\n<h
SF:1>Not\x20Found</h1>\n<p>The\x20requested\x20URL\x20was\x20not\x20found\
SF:x20on\x20the\x20server\.\x20If\x20you\x20entered\x20the\x20URL\x20manua
SF:lly\x20please\x20check\x20your\x20spelling\x20and\x20try\x20again\.</p>
SF:\n")%r(RTSPRequest,73,"HTTP/1\.1\x20400\x20\r\ncontent-length:\x200\r\n
SF:connection:\x20close\r\ndate:\x20Sat,\x2021\x20Dec\x202024\x2012:31:27\
SF:x20GMT\r\nserver:\x20hypercorn-h11\r\n\r\n")%r(DNSVersionBindReqTCP,73,
SF:"HTTP/1\.1\x20400\x20\r\ncontent-length:\x200\r\nconnection:\x20close\r
SF:\ndate:\x20Sat,\x2021\x20Dec\x202024\x2012:31:33\x20GMT\r\nserver:\x20h
SF:ypercorn-h11\r\n\r\n")%r(DNSStatusRequestTCP,73,"HTTP/1\.1\x20400\x20\r
SF:\ncontent-length:\x200\r\nconnection:\x20close\r\ndate:\x20Sat,\x2021\x
SF:20Dec\x202024\x2012:31:33\x20GMT\r\nserver:\x20hypercorn-h11\r\n\r\n");
Service Info: OS: Windows; CPE: cpe:/o:microsoft:windows

Host script results:
| smb2-security-mode: 
|   3:1:1: 
|_    Message signing enabled but not required
| smb2-time: 
|   date: 2024-12-21T12:32:27
|_  start_date: N/A

Service detection performed. Please report any incorrect results at https://nmap.org/submit/ .
Nmap done: 1 IP address (1 host up) scanned in 155.94 seconds
```

ეს არის `nmap`-ის `output`. როგორც ჩანს, `smb` და `rdp` გვაქვს მოცემული. ეს ძალიან კარგია. ახლა ვეცდები `rdp` პროტოკოლით სისტემაში შესვლას. სამწუხაროდ მომხმარებლის სახელს და პაროლს ითხოვს. მაგრამ ის ვიცი ახლა, რომ `Microsoft` სერვერია.

აქ ყველაზე საინტერესო არის ეს:

```
Message signing enabled but not required
```

ზოგადად, როდესაც `SMB Signing` არ ხდება, `SMB Relay` შეტევის საშიშროება ჩნდება. მაგრამ ამ შემთხვევაში ამ შეტევის შესაძლებლობას აზრი ეკარგება, რადგან `VPS` ინტერნეტში დგას და არა `LAN`-ში. იმისათვის, რომ ამ შეტევამ იმუშაოს, მე უნდა ვწვდებოდე ორივეს, ჰაკერის კომპიუტერსაც და სერვერსაც, შემდეგ კი ჰაკერის `authentication`-ის მცდელობა უნდა დავიჭირო, რაც `WAN`-ში შეუძლებელია. ასე, რომ `SMB Relay` შეტევა გამოირიცხა.

ახლა კიდე ვცდილობ სხვადასხვა შეტევების განხორციელებას და სხვადასხვა პორტების ენუმერაციას, მაგრამ როგორც ჩანს, გუშინ ყველაფერი წაშალეს და ცარიელი `VPS` დატოვეს. სამწუხაროა, რომ ვერ ვახერხებ ვერაფერს. პორტები კი ღიაა, მაგრამ მათ უკან ფიზიკურად არაფერია, რომ დავაექსპლოიტო. ასე, რომ მომიწევს აქ გაჩერება.

## მაინც ვიპოვე რაღაც

სკანირების შედეგად კიდევ რამდენიმე დომენი ვიპოვე, რომელიც ამ სერვერთან იყო კავშირში. ეტყობა, საკმაოდ დიდი სქემა ქონდათ გაშლილი.

1. hxxps://saaadnesss.shop
2. hxxps://ddeapeaceofmind.shop
3. hxxps://minimeh.shop
4. hxxps://polovoiinspektor.shop
5. hxxps://uuukaraokeboss.shop

გამოდის, 5 დომენი მოვაშორე ონლაინ სივრციდან, რომლებიც ზიანს აყენებდნენ ყველას. ეს უკვე წინსვლაა.
