---
title: "XChat development ნაწილი #5"
date: 2024-09-18 13:16:00 +0400
categories: [Programming]
tags: [Coding]
---

## ისევ TUI და თავის ტკივილი

ისევ TUI-ზე მუშაობა მიწევს. ამჯერად ყველაფერს ფუნქციებში გადავანაწილებ, ყველაფერი რომ გამარტივდეს. ცოტა წინსვლა არის უკვე:

![1.gif](https://44b4c0.github.io/assets/img/posts/5/1.gif)

ტექსტი გამოდის `chat_window`-ში. ამას `wprintw();` ფუნქციას ვაკეთებინებ. საქმე ისაა, რომ ორი წრედი დამჭირდება, რომლებიც `chat_window`-ზე იმუშავებენ და მგონი მომიწევს `Mutex`-ების დამატებაც.

ახლა `scroll`-ის პრობლემა გავასწორე. საქმე ისაა, რომ როდესაც `chat_window` ვერტიკალურად ივსებოდა ტექსტით, დამატებითი ინფორმაციის შეყვანისას ახალი ტექსტი წინას გვერდით აჯდებოდა, ანუ ჰორიზონტალურად იწყებდა ტექსტი გამოჩენას. ეს, რა თქმა უნდა, დიდ პრობლემას ქმნიდა. არავის უნდა მესიჯების ჰორიზონტალურად `წერა/მიღება`. ასე, რომ მომიწია `scroll` ტექნიკის იმპლემენტაცია და `window size`-ის რეალოკაცია. საბოლოოდ, ყველაფერი კარგად გამოვიდა:

![2.gif](https://44b4c0.github.io/assets/img/posts/5/2.gif)

აქ ქართული ფონტის გამოყენება ვცადე, მაგრამ არ აღიქვა. ასე, რომ ფონტის იმპლემენტაციაც მომიწევს.

## Input-ის პარსვა

როგორც ჩანს, ყველაფერი კარგად მუშაობს დიზაინის მხრივ, ასე, რომ დრო მოვიდა ლოგიკის წერაზე გადავიდე. ლოგიკა უმნიშვნელოვანესია, მის გარეშე არაფერი გამოვა. ახლა ვცდილობ ისე გავაკეთო, რომ ბრძანებები და ჩვეულებრივი მესიჯები ერთმანეთისგან გამოვყო. ასე ვიზამ, თუ მომხმარებლის მიერ შეყვანილი ტექსტის პირველი სიმბოლო იქნება `/`, მაშინ `ParseCommand(char* message);` ფუნქციას გამოვიძახებ, ხოლო თუ პირველი სიმბოლო არ იქნება `/`, მაშინ ჩვეულებრივად გავაგზავნი სერვერთან.

საქმე ისაა, რომ დიდად ბრძანების პარსვაც არ მჭირდება. გააჩნია ბრძანებას. იმ შემთხვევაში, თუ ბრძანება მხოლოდ `client side` ლოგიკას ეხება და სერვერთან არაფერ შუაშია(მაგალითად, `theme`-ის შეცვლა), მაშინ გამოვიყენებ `ParseCommand();`-ს. ან შესაძლოა ამ ფუნქციამ შეამოწმოს არის თუ არა ბრძანება `client side`.

ახლა ვცდილობ რაღაც ბრძანებების ინტეგრაციას(რა თქმა უნდა, მათი პარსვით) და ჯერჯერობით ეს მივიღე:
```cpp
if(strncmp(input_buffer, "/connect", 7) == 0){
	char* command = strtok(input_buffer, " ");
	char* ip_address = strtok(nullptr, " ");
	char* port = strtok(nullptr, " ");

	char test_buffer[MESSAGE_SIZE];
	sprintf(test_buffer, "%s:%s:%s", command, ip_address, port);
	wprintw(chat_window, test_buffer);

	memset(input_buffer, 0, MESSAGE_SIZE);
}
else if(strncmp(input_buffer, "/clear", 5) == 0){
	wclear(chat_window);

	memset(input_buffer, 0, MESSAGE_SIZE);
}
```

ვცადე პირველი ბრძანების პარსვის ტესტირება, აღმოჩნდა, რომ მუშაობს, ასე რომ ლოგიკას გადავწერ და ისე ვიზამ, რომ სერვერთან კავშირის გაბმა სცადოს.

საბოლოო ჯამში, `input`-ის პარსვა ასეთი გამოვიდა(ჯერჯერობით რაც დავამატე):
```cpp
#include <stdio.h>
#include <stdlib.h>

#include <sys/socket.h>
#include <sys/types.h>
#include <arpa/inet.h>

#include <thread>
#include <vector>

#include "client.hpp"

int main(){
    int client_socket;
    sockaddr_in server_address;

    SSL_library_init();
    OpenSSL_add_all_algorithms();

    const SSL_METHOD* ssl_method = SSLv23_client_method();
    SSL_CTX* ssl_context = SSL_CTX_new(ssl_method);

    if(!ssl_context){
        return 1;
    }

    signal(SIGWINCH, HandleGlobalResize);

    int screen_init_res = CreateScreen();

    if(screen_init_res != 0){
        return 1;
    }

    while(true == true){
        char input_buffer[MESSAGE_SIZE];
        wclear(message_window);
        wrefresh(message_window);

        wgetstr(message_window, input_buffer);

        if(strlen(input_buffer) > MESSAGE_SIZE){
            continue;
        }

        if(strncmp(input_buffer, "/connect", 7) == 0){
            char* command = strtok(input_buffer, " ");
            char* ip_address = strtok(nullptr, " ");
            char* port = strtok(nullptr, " ");

            if(ip_address == nullptr){
                wprintw(chat_window, "[ERROR]: IP address required!\n");
                wrefresh(chat_window);
                continue;
            }
            else if(port == nullptr){
                wprintw(chat_window, "[ERROR]: Port required!\n");
                wrefresh(chat_window);
                continue;
            }

            if(strlen(username_buffer) <= 0){
                wprintw(chat_window, "[ERROR]: Username not set!\n");
                wrefresh(chat_window);
                continue;
            }

            memset(input_buffer, 0, MESSAGE_SIZE);

            client_socket = socket(AF_INET, SOCK_STREAM, 0);

            if(client_socket < 0){
                wprintw(chat_window, "[ERROR]: Unable to open socket\n");
                wrefresh(chat_window);
                continue;
            }

            server_address.sin_family = AF_INET;
            server_address.sin_addr.s_addr = inet_addr(ip_address);
            int int_port = atoi(port);
            server_address.sin_port = htons(int_port);

            socklen_t sizeof_server_address = sizeof(server_address);

            if(connect(client_socket, reinterpret_cast<sockaddr*>(&server_address), sizeof_server_address) < 0){
                wprintw(chat_window, "[ERROR]: Unable to find the server\n");
                wrefresh(chat_window);
                continue;
            }

            SSL* ssl_fd = SSL_new(ssl_context);
            SSL_set_fd(ssl_fd, client_socket);

            if(SSL_connect(ssl_fd) <= 0){
                wprintw(chat_window, "[ERROR]: Unable to connect SSL\n");
                wrefresh(chat_window);
                continue;
            }

            SSL_write(ssl_fd, username_buffer, USERNAME_SIZE);

            while(true == true){
                wgetstr(message_window, input_buffer);

                SSL_write(ssl_fd, input_buffer, MESSAGE_SIZE);

                memset(input_buffer, 0, MESSAGE_SIZE);
            }
        }
        else if(strncmp(input_buffer, "/username", 8) == 0){
            char* command = strtok(input_buffer, " ");
            char* username = strtok(nullptr, " ");

            if(strlen(username) > USERNAME_SIZE){
                wprintw(chat_window, "[ERROR]: Username must be less than 32 characters in size!\n");
            }

            sprintf(username_buffer, "%s", username);
        }
        else if(strncmp(input_buffer, "/clear", 5) == 0){
            wclear(chat_window);

            memset(input_buffer, 0, MESSAGE_SIZE);
        }

        wrefresh(chat_window);
    }

    int screen_del_res = DeleteScreen();

    return 0;
}
```

## ქსელური პრობლემები

სამწუხაროდ, ჯერჯერობით არ ვიცი რატომ, მაგრამ `127.0.0.1:8080`-ზე მეუბნება, რომ ქსელი მიუწვდომელია. არადა `./server` გაშვებულია. ალბათ დამჭირდება დრო ამ პრობლემის გადასაჭრელად. რომ შევამოწმე `ip_address` და `port`(`int_port`), ყველა კარგად გამოიყურება. სერვერიც გამართულად მუშაობს და ამ ეტაპზე არ ვიცი რა შეიძლება იყოს პრობლემა.

საინტერესოა თუ როგორ ვტესტავ `/connect` ბრძანებას. რა თქმა უნდა, თუ `username_buffer` ცარიელია, მაშინ კავშირი არ შედგება, რადგან კლიენტმა პირველივე კავშირისას უნდა გააგზავნოს მისი `username` სერვერთან, ხოლო თუ კლიენტს არ აქვს ის გამზადებული, მაშინ შეუძლებელია კავშირის შედგენა სერვერთან და კომუნიკაცია დაიშლება.
