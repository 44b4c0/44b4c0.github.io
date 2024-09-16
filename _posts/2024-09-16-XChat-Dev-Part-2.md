---
title: "XChat development ნაწილი #2"
date: 2024-09-16 14:06:00 +0400
categories: [Programming]
tags: [Coding]
---

## წრედების სინქრონიზაცია

პრობლემა იმაში მდგომარეობს, რომ თითოეულმა წრედმა უნდა შეძლოს `client_vector`-თან მუშაობა, რამაც პრობლემები შეიძლება გამოიწვიოს. ამის გამო, მე მჭირდება ისეთი რამის გაკეთება, რაც წრედებს სინქრონულად ამუშავებს `client_vector`-თან მიმართებაში.

ამაში შემიძლია Mutex დავიხმარო. ზოგადად, `Mutex` ნიშნავს `Mutual Exclusion`-ს და `C++` ენაში წრედების რესურსების გასაკონტროლებლად გამოიყენება(ძალიან პრიმიტიული ახსნაა). დაახლოებით ასეთი  რამ ხდება, როდესაც ერთი ობიექტზე რამდენიმე წრედს უნდა მუშაობა:
```
   ┌─────────┐                                         
   │Thread #2│                                         
   └─────────┘                                         
        │                                              
        │                                              
        │                                              
        │                                              
        │                                              
        │                                              
        │                                              
        │                                              
        │                                              
        x                                              
┌───────────────┐               ┌─────────┐            
│Protector Mutex│◄──────────────┤Thread #1│            
└───────┬───────┘               └─────────┘            
        │                                              
        │                                              
        │                                              
        ▼                                              
 ┌─────────────┐             ┌────────────────────────┐
 │client_vector├────────────►│{Item1, Item2, Item3...}│
 └─────────────┘             └────────────────────────┘
```

ამ შემთხვევაში,  `Thread #1` მუშაობს `client_vector`-თან, რომელიც `Protector Mutex`-ს ჰყავს ჩაკეტილი. როდესაც `Thread #1` ვექტორთან მუშაობას მორჩება, `Protector Mutex` გაიხსნება და `Thread #2` შეძლებს `client_vector`-ზე მუშაობის გაგრძელებას.

ეს ყველაფერი რომ შევძლო, საჭიროა ამ ბიბლიოთეკის `server.hpp` ფაილში გაწერა:
```cpp
#include <mutex>
```

ახლა უკვე შემიძლია შევქმნა `Mutex` ობიექტი, რომელიც `client_vector`-ს დაიცავს. თავდაპირველად ასეთ რამეს გავაკეთებ `main()` ფუნქციაში, რომელიც, რა თქმა უნდა, `server.cpp` ფაილში იქნება მოთავსებული:
```cpp
std::mutex client_vector_mutex;
```

სწორედ ეს დაიცავს `client_vector` ვექტორს. ახლა, საჭიროა სერვერის ბირთვის აწყობა, ანუ ის ნაწილი, რომელიც უშუალოდ კავშირის მიღებაზე იქნება პასუხისმგებელი. რა თქმა უნდა, სერვერმა გაუჩერებლად უნდა იმუშაოს, ასე რომ მომიწევს უსასრულოდ მომუშავე ციკლის შექმნა:
```cpp
while(true == true){
	if(client_vector.size() >= MAX_CLIENT){
	}
	else{
		std::thread new_thread(ReadFromClient, client_vector_mutex);
		new_thread.detach();
	}
}
```

პირველ ნაწილში, მივიღებ კლიენტს და უბრალოდ ვეტყვი, რომ სერვერი სავსეა. მეორე შემთხვევაში, შეიქმნება წრედი, რომელიც მოემსახურება კლიენტს. რა თქმა უნდა, ჯერჯერობით ეს უბრალო მონახაზია, სინამდვილეში კოდი განსხვავებული იქნება, მაგრამ დაახლოებით ასეთი რამის გაკეთებას ვაპირებ.

## HandleClient(); ფუნქცია

რაც შეეხება ამ ფუნქციას, ის კლიენტს `SSL/socket`-ის გამოყენებით მოემსახურება, რაც გულისხმობს იმას, რომ სოკეტი `SSL Tunnel`-ში გაივლის და ტრაფიკი დაშიფრული იქნება.
```cpp
void ReadFromClient(std::mutex client_vector_mutex){
    std::lock_guard<std::mutex> lock(client_vector_mutex);
}
```

რა თქმა უნდა, თავდაპირველად საჭიროა `Mutex`-ის ჩაკეტვა, რათა `client_vector`-ზე მუშაობა შევძლო. შემდეგ, საჭიროა `std::vector<Client> client_vector`-ის პარამეტრებში დამატება:
```cpp
void ReadFromClient(std::mutex client_vector_mutex, std::vector<Client> client_vector){
    std::lock_guard<std::mutex> lock(client_vector_mutex);
}
```

გარკვეული მოდიფიკაციების შედეგად, საბოლოო ჯამში, ასეთი `server.hpp` ფაილი მივიღე:
```cpp
#pragma once

#include <mutex>
#include <algorithm>

#include <openssl/ssl.h>

/* Server Bind config */

#define SERVER_ADDRESS "0.0.0.0"
#define SERVER_PORT 8080

/* Server Logic config */
#define MAX_CLIENT 200

/* Communication config */
#define MESSAGE_SIZE 1024

class Client {
    public:
    char* username;

    char* ip_address;
    unsigned int port;

    int client_fd;
    SSL* ssl_fd;
};

void RemoveClient(std::vector<Client>& client_vector, int client_fd, std::mutex& client_vector_mutex) {
    std::lock_guard<std::mutex> lock(client_vector_mutex);

    client_vector.erase(
        std::remove_if(client_vector.begin(), client_vector.end(),
            [client_fd](const Client& client) {
                return client.client_fd == client_fd;
            }),
        client_vector.end());
}

void ReadFromClient(SSL* ssl_fd, int client_fd, sockaddr_in client_address, char* username, std::mutex& client_vector_mutex, std::vector<Client>& client_vector){
    {
        std::lock_guard<std::mutex> lock(client_vector_mutex);

        Client new_client;

        new_client.client_fd = client_fd;
        new_client.ssl_fd = ssl_fd;
        new_client.username = username;
        new_client.ip_address = inet_ntoa(client_address.sin_addr);
        new_client.port = ntohs(client_address.sin_port);

        client_vector.push_back(new_client);  
    }  

    while(true == true){
        char message_buffer[MESSAGE_SIZE];

        int received_bytes = SSL_read(ssl_fd, message_buffer, MESSAGE_SIZE - 1);

        if(received_bytes <= 0){
            continue;
        }
        else{
            message_buffer[received_bytes] = '\0';

            {
                std::lock_guard<std::mutex> lock(client_vector_mutex);

                for(const auto& client : client_vector){
                    SSL_write(client.ssl_fd, message_buffer, MESSAGE_SIZE);
                }
            }
        }
    }

    {
        std::lock_guard<std::mutex> lock(client_vector_mutex);
        RemoveClient(client_vector, client_fd, client_vector_mutex);
    }
}
```

აქ ყველაზე მნიშვნელოვანი `scope`-ების ანალიზია.

> **📰 საინტერესო ფაქტი:**
> 
> როდესაც `std::lock_guard`-ით `Mutex` იხურება, ეს იმას ნიშნავს, რომ მხოლოდ კონკრეტულ წრედს შეუძლია სამიზნესთან მუშაობა. საქმე ისაა, რომ როდესაც ეს ხდება, ფუნქციაში მას ცალკე `scope` უნდა გამოეყოს, რათა `scope`-ის დასრულების ადგილზე ისევ, ავტომატურად, გაიხსნას `Mutex`.

რა თქმა უნდა, ეს ფუნქცია ჯერ კიდევ მონახაზია და ბევრ ცვლილებას განიცდის სამომავლოდ. ამჟამად, დროა შევამოწმო სწორად მუშაობს თუ არა სერვერ:

	g++ src/server.cpp -o server -pthread -lssl -lcrypto

ამ ბრძანებით შევძლებ კოდის კომპილაციას. სანამ კოდს გადავაკომპილირებ, `server.cpp` ფაილიც გასასწორებელია.

## კოდის მთავარი ნაწილი

ახლა, კოდის მთავარ ნაწილზე გადავალ. იმ შემთხვევაში თუ სერვერი სავსე იქნება, კლიენტს მიიღებს და უპასუხებს, რომ ადგილი არ დარჩა მისთვის, შემდეგ კი კავშირს დახურავს. თუ სერვერი სავსე არ იქნება, მაშინ კლიენტს მიიღებს და მისთვის ცალკე წრედს გამოყოფს. ამ ყველაფრის გაკეთება, შემიძლია იმ კოდის გამოყენებით, რომელიც `main();` ფუნქციაში დავამატე:
```cpp
while(true == true){
    if(client_vector.size() >= MAX_CLIENT){
        SSL* ssl_fd = SSL_new(ssl_context);

        if(!ssl_fd){
            SSL_free(ssl_fd);
            continue;
        }

        sockaddr_in client_address;
        socklen_t client_address_size;
        int client_fd = accept(server_socket, reinterpret_cast<sockaddr*>(&client_address), &client_address_size);

        if(client_fd < 0){
            SSL_free(ssl_fd);
            close(client_fd);
            continue;
        }

        char username_buffer[USERNAME_SIZE];
        int recevied_bytes = SSL_read(ssl_fd, username_buffer, USERNAME_SIZE - 1);

        if(recevied_bytes <= 0){
            SSL_free(ssl_fd);
            close(client_fd);
            continue;
        }

        username_buffer[recevied_bytes] = '\0';

        SSL_write(ssl_fd, CODE_SERVER_FULL, strlen(CODE_SERVER_FULL));

        SSL_free(ssl_fd);
        close(client_fd);
        continue;
    }
    else{
        SSL* ssl_fd = SSL_new(ssl_context);

        if(!ssl_fd){
            SSL_free(ssl_fd);
            continue;
        }

        sockaddr_in client_address;
        socklen_t client_address_size;
        int client_fd = accept(server_socket, reinterpret_cast<sockaddr*>(&client_address), &client_address_size);

        if(client_fd < 0){
            close(client_fd);
            continue;
        }

        SSL_set_fd(ssl_fd, client_fd);

        if(SSL_accept(ssl_fd) < 0 ){
            SSL_free(ssl_fd);
            close(client_fd);
            continue;
        }

        char username_buffer[USERNAME_SIZE];

        int received_bytes = SSL_read(ssl_fd, username_buffer, USERNAME_SIZE - 1);

        if(received_bytes <= 0){
            SSL_free(ssl_fd);
            close(client_fd);
            continue;
        }

        username_buffer[received_bytes] = '\0';

        std::thread new_thread(ReadFromClient, ssl_fd, client_fd, client_address, username_buffer, std::ref(client_vector_mutex), std::ref(client_vector));
        new_thread.detach();
    }
}
```
