---
title: "XChat development ნაწილი #1"
date: 2024-09-15 22:29:00 +0400
categories: [Programming]
tags: [Coding]
---

## შესავალი

რამდენიმე წლის წინ გავაკეთე `Terminal based` ჩატი პითონში, რომელიც არც ისე კარგად მუშაობდა მაგრამ საბოლოო ჯამში თავის საქმეს აკეთებდა.

ახლა ვფიქრობ, ხომ არ ჯობია იგივე გავაკეთო `C++` ენით, მაგრამ ამჯერად ბევრად უკეთესი. წინა პროექტის უარყოფითი მხარეები იყო:

- **დაუცველი კავშირი სერვერსა და კლიენტებს შორის** ⛓️‍💥
- **შეზღუდული კლიენტების რაოდენობა** 👨‍👩‍👧‍👦
- **TUI-ს არ ქონა** 🎨

ამ ყველაფერმა ძალიან დიდი დისკომფორტი გამოიწვია, ასე რომ ახლა ვიმუშავებ პროექტზე, რომელიც ამ პრობლემებს აანაზღაურებს.

სანამ კოდის წერას დავიწყებ, უნდა მოვიფიქრო თუ რა სახის სისტემა იქნება. რა თქმა უნდა, იქნება სერვერისა და კლიენტის მხარე. ანუ, იქნება ორი ფაილი და ამ ფაილებიდან ერთი მოიქცევა როგორც სერვერი, ხოლო მეორე კლიენტი.

## რის გამოყენებას ვაპირებ?

საუკეთესო ვარიანტი ალბათ `sockets/OpenSSL` იქნება. სოკეტების დახმარებით კავშირი შედგება კლიენტებსა და სერვერს შორის, ხოლო `OpenSSL` მათ ტრაფიკს დაშიფრავს. სერვერზე თითოეულ კლიენტს ცალკე `Thread` გამოეყოფა და იქნება `client_vector` ვექტორი, რომელიც ინფორმაციას ყველა კლიენტზე შეინახავს. ამ ინფორმაციას `Client` კლასი დაიჭერს. საბოლოო ჯამში, სისტემა ასე ლაგდება:

```
                         ┌───────────┐                                              
       ┌────────────────►│Main Thread├────────┬──────────────┬──────────────┐       
       │                 └───────────┘        │              │              │       
       │                                      │              │              │       
       │                                      │              │              │       
       │                                      │              │              │       
       │                                      │              │              │       
       │                                      │              │              │       
       │                                      │              │              │       
┌──────┴──────┐                        ┌─────────────┐┌─────────────┐┌─────────────┐
│Client Vector│◄───────────────────────┼Client Thread││Client Thread││Client Thread│
└─────────────┘                        └─────────────┘└─────────────┘└─────────────┘
       ▲                                                                            
       │                                                                            
       │                                                                            
       │                                                                            
┌──────┴─────┐                         ┌─────────────────────┐                      
│Client class│◄────────────────────────┤#include "server.hpp"│                      
└────────────┘                         └─────────────────────┘                      
```

ცოტა კომპლექსური გამოვიდა დიაგრამა, მაგრამ არაუშავს. გრაფიკას რაც შეეხება, `TUI based` იქნება ჩატი და ამ ყველაფრისთვის `ncurses.h` ბიბლიოთეკას გამოვიყენებ დიდი ალბათობით.

## პროექტის შექმნა და კოდირების დაწყება

რა თქმა უნდა, პირველ რიგში `Git init` ბრძანება და შემდეგ ვუდგები საქმეს. შევქმენი ორი ფოლდერი(დირექტორია):

- **client 💻**
- **server 💽**

თითოეული მათგანი შეიცავს ორ ფაილს:

- **.cpp**
- **.hpp**

ანუ, საბოლოო ჯამში, პროექტის სტრუქტურა ასეთია:

```
                      ┌─────┐                      
                      │XChat│                      
                      └──┬──┘                      
                         │                         
                         │                         
                         │                         
                         │                         
                         │                         
                 ┌───────┴───────┐                 
                 │               │                 
                 │               │                 
                 │               │                 
             ┌───┴──┐        ┌───┴──┐              
             │Client│        │Server│              
             └───┬──┘        └───┬──┘              
┌──────────┐     │               │     ┌──────────┐
│client.cpp├─────┤               ├─────┤server.cpp│
└──────────┘     │               │     └──────────┘
                 │               │                 
                 │               │                 
                 │               │                 
┌──────────┐     │               │     ┌──────────┐
│client.hpp├─────┘               └─────│server.hpp│
└──────────┘                           └──────────┘
```

თავდაპირველად, სერვერს დავწერ, რადგან როდესაც კლიენტის წერაზე გადავალ უკვე მექნება გარკვეული საფუძველი, ანუ ის, რასაც კლიენტი უნდა დაუკავშირდეს, რომ ინფორმაცია გაუცვალოს.

თავდაპირველად შევქმენი სერვერი `server.cpp` ფაილში:
```cpp
#include <stdio.h>
#include <stdlib.h>

#include <sys/socket.h>
#include <sys/types.h>
#include <arpa/inet.h>

#include <openssl/ssl.h>

#include "server.hpp"

int main(void){
    int server_socket; sockaddr_in server_address;

    server_address.sin_family = AF_INET;
    server_address.sin_addr.s_addr = inet_addr(SERVER_ADDRESS);
    server_address.sin_port = htons(SERVER_PORT);

    socklen_t sizeof_server_address = sizeof(server_address);

    server_socket = socket(AF_INET, SOCK_STREAM, 0);

    if(server_socket < 0){
        return 1;
    }

    if(bind(server_socket, reinterpret_cast<sockaddr*>(&server_address), sizeof_server_address) < 0){
        return 1;
    }

    if(listen(server_socket, 5) < 0){
        return 1;
    }

    return 0;
}
```

რა თქმა უნდა, `server.hpp` ფაილშიც დავამატე რაღაცები:
```cpp
#pragma once

#define SERVER_ADDRESS "0.0.0.0"
#define SERVER_PORT 8080
```

ახლა, დრო მოვიდა `OpenSSL` გადავატარო სოკეტს. ეს იმიტომ, რომ ტრაფიკი თავიდანვე დაშიფრული იყოს და მერე აღარ მომიწიოს კოდში ქექვა:

```cpp
#include <stdio.h>
#include <stdlib.h>

#include <sys/socket.h>
#include <sys/types.h>
#include <arpa/inet.h>

#include <openssl/ssl.h>

#include "server.hpp"

int main(void){
    int server_socket; sockaddr_in server_address;

    server_address.sin_family = AF_INET;
    server_address.sin_addr.s_addr = inet_addr(SERVER_ADDRESS);
    server_address.sin_port = htons(SERVER_PORT);

    socklen_t sizeof_server_address = sizeof(server_address);

    server_socket = socket(AF_INET, SOCK_STREAM, 0);

    if(server_socket < 0){
        return 1;
    }

    if(bind(server_socket, reinterpret_cast<sockaddr*>(&server_address), sizeof_server_address) < 0){
        return 1;
    }

    if(listen(server_socket, 5) < 0){
        return 1;
    }

    SSL_library_init();
    OpenSSL_add_all_algorithms();

    const SSL_METHOD* ssl_method = SSLv23_server_method();
    SSL_CTX* ssl_context = SSL_CTX_new(ssl_method);

    if(!ssl_context){
        return 1;
    }

    if (SSL_CTX_use_certificate_file(ssl_context, "server.crt", SSL_FILETYPE_PEM) <= 0 || SSL_CTX_use_PrivateKey_file(ssl_context, "server.key", SSL_FILETYPE_PEM) <= 0) {
        return 1;
    }

    return 0;
}
```

აქ ყველაზე საინტერესო ისაა, რომ ვამოწმებ `server.crt` და `server.key` ფაილებს. ეს იმიტომ, რომ სერვერს აუცილებლად უნდა ჰქონდეს თავისი `public` და `private` გასაღებები.

> **📰 საინტერესო ფაქტი:**
> 
> კრიპტოგრაფიაში არსებობს გასაღებების კონცეფცია, რაც ითვალისწინებს იმას, რომ დაშიფრული ტუნელის გაყვანისას, ერთმა გასაღებმა უნდა დაშიფროს ინფორმაცია, ხოლო მეორემ უნდა გაშიფროს. `public`(ანუ `საჯარო`) გასაღები დაშიფრავს ინფორმაციას(წაუკითხავს გახდის მას), ხოლო `private`(ანუ `კერძო`) გასაღები გაშიფრავს მას(წაკითხვადს გახდის მას). ზოგადად, ასეთი სიტუაციებისას, `public` გასაღები მოსაუბრეს გადაეცემა, რათა მან ჩვენთვის სასურველი ინფორმაცია დაშიფროს, ხოლო როდესაც ჩვენ მას მივიღებთ, ის ჩვენივე `private` გასაღებით გაიშიფრება. ეს ყველაფერი ჩუმად მოსმენის შესაძლებლობას გამორიცხავს და ტრაფიკიც დაცულია ჰაკერებისგან და მავნე აქტორებისგან.

## კლიენტის კლასი

ახლა ავაწყე კლიენტის კლასი. ეს უნდა მოთავსდეს ვექტორში და უნდა შეინახოს გარკვეული ინფორმაცია კლიენტის შესახებ.
```cpp
class Client {
    public:
    char* username;

    char* ip_address;
    unsigned int port;

    int client_fd;
    SSL* ssl_fd;
};
```

მოცემულია `username`, `ip_address`, `port`, `client_fd` და `ssl_fd` ცვლადები. პირველი სამი ალბათ ძალიან მარტივად გასაგებია, მაგრამ ბოლო ორს შედარებით მეტი დაკვირვება სჭირდება.

ბოლო ორი ცვლადი არის კლიენტის ე.წ. `Handle`. ისინი საშუალებას მომცემენ კლიენტს პირდაპირ ვექტორიდან მივმართო. მაგალითად, თუ მომინდა კლიენტებისთვის `SSL`-ით დაშიფრული ტრაფიკით ინფორმაცია გადავცე, შემიძლია ასე გამოვიძახო კლიენტები:
```cpp
for(const &auto client : client_vector){
	SSL_write(ssl_fd, message_buffer, std::strlen(message_buffer));
}
```

ეს ყველაფერს გაამარტივებს. ანუ, აღარ მომიწევს კლიენტზე მისაწვდომი ინფორმაციის, ანუ `Handle`-ის ცალკე გატანა და დამატებითი ალგორითმების წერა მათ გამოსათვლელად. რაც შეეხება `client_fd` ცვლადს, ის უბრალო `socket`-ის `Handle` არის, რომელიც საშუალებას მომცემს კლიენტს დაუშიფრავად ვეკონტაქტო(ამის საჭიროება არ არსებობს და არც არასოდეს იარსებებს).
