---
title: "საქართველოში 'გაწეული' სისტემები???"
date: 2024-10-27 22:34:00 +0400
categories: [CyberSecurity,Hacking]
tags: [Proxies]
---

## როუტერი ყველაფერს ხედავს

ჩემი სახლის გარე(ანუ იმ როუტერზე, საიდანაც სახლში ინტერნეტი შემომდის) `WAN` ლოგირების სისტემა ჩავრთე. ეს საშუალებას მაძლევს ე.წ. `port scan`-ები და `DoS` შეტევები დავაფიქსირო. ამ ყველაფერმა შედეგი გამოიღო და ბევრი საინტერესო რამ აღმოვაჩინე. როუტერის ლოგების ანალიზისას შევამჩნიე, რომ ქართული `IPv4` მისამართებიდან შემოდიოდა `port scan` მცდელობები.

## Port scan რა არის?

პორტების სკანირება, ანუ `port scan` არის ინფორმაციის მოპოვების(Information gathering) აქტიური მეთოდი.

> **📰 საინტერესო ფაქტი:**
> 
> არსებობს ინფორმაციის მოძიების ორი მეთოდი: `აქტიური` და `პასიური`. `აქტიური` მეთოდი მოისაზრებს სამიზნესთან პირდაპირ შეხებას(შესაძლოა გამოყენებულ იქნას `Tor`/`Proxy` ქსელი, მაგრამ ამ დროს სამიზნე მაინც ხედავს, რომ მასთან ვიღაც მუშაობს(ცუდი გაგებით). `პასიური` მეთოდი კი ნიშნავს იმას, რომ სამიზნესთან პირდაპირი შეხება არ გვაქვს და ის ვერ ხვდება, რომ მასთან მუშაობენ(ცუდი გაგებით). ზოგადად `Blue team`-ს უჭირს ქსელში ისეთი აქტივობების შემჩნევა, რომლებიც პასიურ ინფორმაციის მოძიებაში გადის, ასე, რომ ის კარგი გზაა ქსელში ან თუნდაც აღებულ ჰოსტზე შეუმჩნევლად დარჩენაში.

ანუ გამოდის, რომ მე მაქვს `IPv4` მისამართები, რომელბიც მასკანირებდნენ. ამოვიწერე ეს მისამართები და შევუდექი მათ მოკვლევას. აღმოჩნდა, რომ `IP` მისამართები ქართულია. თავდაპირველად ვიფიქრე, რომ ამ მისამართზე იყო ვიღაც ვინც მემტერებოდა, მაგრამ დეტალური ინფორმაციის მოგროვების შემდეგ აღმოჩნდა, რომ ვცდებოდი. როდესაც ეს მისამართები დავასკანირე, მათზე გახსნილი იყო `HTTP` პორტები, რამაც მაფიქრებინა, რომ ეს იყო მათი როუტერის სამართავი პანელი.

> **📰 საინტერესო ფაქტი:**
> 
> იმ როუტერებს, რომლებსაც `ISP` ანაწილებს სახლებში, გახსნილი აქვთ `HTTP` პორტი ვებპანელისათვის. მასზე შესვლა შესაძლებელია როგორც გარე ქსელიდან, ისე შიდა ქსელიდანაც, ანუ გამოდის, რომ `bind 0.0.0.0` აქვს ოფციებში გაწერილი, თუმცა, ამის შეცვლა შეიძლება. ეს კი იმას ნიშნავს, რომ შესაძლებელია გარე ქსელიდან ამ პორტზე ტრაფიკის დაბლოკვა.

მარტივი `port scan` ის მაგალითია:

```
nmap -sC -sV -A example.com -p- -T5 > nmapScan.txt
```

ეს ბრძანება `example.com`-ის ყველა `TCP` პორტს(1-65535) ამოწმებს(ანუ არის თუ არა რომელიმე მათგანი ღია). თუ პორტი ღიაა, მაშინ `-sV` ოფციის გამო, `nmap` ეცდება ბანერიდან სერვისის სახელი და მისი ვერსია წამოიღოს, მაგალითად: `Apache 2.4.6` ან `Boa/0.93.15`.

ასევე შესაძლებელია ხელით გაიწეროს ავტომატიზირებული ხელსაწყო, საქმის სწრაფად გასაკეთებლად, მაგრამ აქ პრობლემა ისაა, რომ ჰაკერს ხელით მოუწევს ვერსიების ამოღება ბანერებიდან:

```cpp
#include <stdio.h>
#include <stdlib.h>

#include <sys/types.h>
#include <sys/socket.h>
#include <arpa/inet.h>

#include <unistd.h>

#include <vector>

class port_info {
	public:
	char banner[1024];
	unsigned int port;
};

int main(int argc, char* argv[]){
	if(argc < 2){
		printf("Usage: %s <HOST>", argv[1]);
		return 1;
	}

	std::vector<port_info> opened_ports;

	int client_socket = socket(AF_INET, SOCK_STREAM, 0);

	for(unsigned int port = 1; port < 65535; port++){
		sockaddr_in server_addr;
		server_addr.sin_addr.s_addr = inet_addr((const char*) argv[1]);
		server_addr.sin_port = htons(port); server_addr.sin_family = AF_INET;

		int res_con = connect(client_socket, (sockaddr*)&server_addr, sizeof(sockaddr_in));

		if(res_con == -1){
			continue;
		}
		else{
			char* buffer = (char*) malloc(1024*sizeof(char));
			int bytes_recv = recv(client_socket, buffer, 1024 - 1, 0);
			buffer[bytes_recv] = '\0';

			port_info pi;
			pi.port = port;
			strncpy(pi.banner, buffer, 1024 - 1);

			opened_ports.push_back(pi);

			free(buffer);
			close(client_socket);
		}
	}

	// აქ ბანერების პარსვა ხელით.

	return 0;
}
```

დაახლოებით ამ ლოგიკას უნდა მიყვებოდეს კოდი.

## უცნაური აღმოჩენა

 როდესაც პორტები დავასკანირე იმ მისამართებზე, რომლებიც მამოწმებდნენ და ჩემს როუტერში დაუცველობას ეძებდნენ, უცნაური რამ აღმოვაჩინე:

```
PORT    STATE SERVICE   VERSION
80/tcp  open  http      proxygen-bolt
|_http-server-header: proxygen-bolt
| fingerprint-strings: 
|   GetRequest, HTTPOptions: 
|     HTTP/1.1 301 Moved Permanently
|     Location: https:///
|     Content-Type: text/plain
|     Server: proxygen-bolt
|     Date: Wed, 23 Oct 2024 19:03:18 GMT
|     Connection: close
|     Content-Length: 0
|   RTSPRequest: 
|     HTTP/1.1 400 Bad Request
|     Content-Type: text/html; charset=utf-8
|     Date: Wed, 23 Oct 2024 19:03:18 GMT
|     Connection: close
|     Content-Length: 2959
|     <!DOCTYPE html>
|     <html lang="en" id="facebook">
|     <head>
|     <title>Facebook | Error</title>
|     <meta charset="utf-8">
|     <meta http-equiv="cache-control" content="no-cache">
|     <meta http-equiv="cache-control" content="no-store">
|     <meta http-equiv="cache-control" content="max-age=0">
|     <meta http-equiv="expires" content="-1">
|     <meta http-equiv="pragma" content="no-cache">
|     <meta name="robots" content="noindex,nofollow">
|     <style>
|     html, body {
|     color: #141823;
|     background-color: #e9eaed;
|     font-family: Helvetica, Lucida Grande, Arial,
|     Tahoma, Verdana, sans-serif;
|     margin: 0;
|     padding: 0;
|     text-align: center;
|     #header {
|_    height: 30px;
|_http-title: Did not follow redirect to https://host-XX-XXX-XXX-XX.XXXXXXXX.XXXXXXXX.ge/
```

ბოლოში `X`-ები სპეციალურად დავწერე. უცნაური აქ ისაა, რომ `proxygen-bolt` არის გაშვებული. შეიძლება ითქვას ეს `facebook`-ის გუნდის მიწერ დაწერილი `HTTP` ბიბლიოთეკაა, რომელსაც `GitHub`-ზე ძალიან მალე მივაგენი. უცნაურია, რომ გაპროქსილია სისტემა. ანუ გამოდის, რომ ქართველი კი არ მჰაკავდა(უფრო სწორად, ცდილობდა ჩემს დაჰაკვას, რადგან არ გამოუვიდა), არამედ ვიღაც სხვა, რომელმაც ქართველი დაჰაკა და მის სისტემას როგორც `Proxy`-ს, ისე იყენებს. შესაძლოა ქართველმავე დაჰაკა ეს და თავისი იდენტობის დასამალად იყენებს ამას. სამწუხაროდ, არ შემიძლიასა ვნახო ამ მისამართის უკან ვინ დგას, რადგან არ მაქვს `routing` ლოგები და იდეაში ვერც მოვიპოვებ თუ `ISP` არ გავწიე(დავჰაკე), რასაც ნამდვილად არ ვაპირებ. ანუ, იმის თქმა მსურს, რომ ჩემი ქმედებები საქართველოს სისხლის სამართალს არ უნდა კვეთდეს უხეშად. ეს არის თამაშის ერთადერთი წესი.

## რას ვაპირებთ?

ნამდვილად არ ვიცი რას ვაპირებ. შესაძლოა ეს პროქსი საჩემოდ გამოვიყენო. ანუ, თუ ისე მოხდა, რომ საქართველოში ვინმესთან ცუდად თამაში(ანუ "კიბერომი") მომიწევს, მირჩევნია ეს გამოვიყენო და იდენტობა დავმალო, რადგან ჰაკერს პროქსიზე ავთენტიფიკაციის დამატება დაავიწყდა:

![1.png](https://44b4c0.github.io/assets/img/posts/7/1.png)

როგორც ჩანს `proxy` გამართულად მუშაობს და `index.html` ფაილიც კი წამოვიღე `http://example.com`-დან. ერთი კარგი და ერთი ცუდი ამბავი მაქვს. სხვათა შორის, კიდევ მაქვს ერთი ქართული მისამართი სადაც ზუსტად იგივე ხდება. გასათვალისწინებელია ის, რომ არც ჰოსტინგისაა ეს აიპები და ლეგალურად არც პროქსიდაა დარეგისტრირებული. თან უკანონო საქმიანობაში იყვნენ ჩართულები და გამოდის, რომ დაჰაკული სისტემებია. შემიძლია მოვედო ამ სისტემებს და მეთვითონ გავაკონტროლო ან შემიძლია უბრალოდ გამოვიყენო ეს `HTTP Proxy`-ები სხვადასხვა აქტივობებისათვის.
