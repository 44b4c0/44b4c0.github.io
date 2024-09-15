---
title: "XChat დეველოპმენტი Part #4"
date: 2024-09-15 14:35:00 +0400
categories: [Programming]
tags: [Coding]
---

##  კლიენტის მხარეს პარამეტრებთან მუშაობა

დღეს კლიენტის მხარესაც უნდა მივხედო. საერთოდ არ დამიწერია არაფერი და დროა რაღაც ლოგიკა მაინც ავუყწო კლიენტის მხარეს.

კლიენტის მხარე შემდეგნაირად იმუშავებს: `თავდაპირველად მთავარი წრედი იქნება, შემდეგ კი კოდის ნაკადი ორ წრედად გაიყოფა: ერთი სერვერიდან მიიღებს ინფორმაციას(მესიჯებს), ხოლო მეორე პირიქით, სერვერს გაუგზავნის ინფორმაციას.`.

პირველ რიგში, `client.hpp` ფაილში უნდა გავაკეთო `Log(char* message, unsigned int verbose_level)` ფუნქცია, რომელსაც ზუსტად იგივე ლოგიკა ექნება, რაც სერვერის `Log()` ფუნქციას. ერთადერთი განსხვავება აქ ის იქნება, რომ კლიენტის ფუნქციას არ ექნება `bool is_initial` პარამეტრი, რადგან კლიენტი, სერვერისგან განსხვავებით, არ ინიციალიზდება.

ასე, რომ ამ ეტაპზე `client.hpp` ფაილი ასეთი იქნება:
```cpp
#include <fstream>
#include <cstring>

#include <time.h>

#define LOG_FILE_PATH "logs/client.log"

#define CLIENT_USAGE "[USAGE]: "

#define CLIENT_INFO "[INFO]: "
#define CLIENT_WARNING "[WARNING]: "
#define CLIENT_ERROR "[ERROR]: "

#define LOG_MESSAGE_BUFFER_SIZE 8096
#define NUGGET_MESSAGE_BUFFER_SIZE 512

int Log(char* message, unsigned int verbose_level){
    std::ofstream file(LOG_FILE_PATH, std::ios::app);

    char log_message_buffer[LOG_MESSAGE_BUFFER_SIZE];

    time_t current_time;
    struct tm *time_info;
    char time_buffer[NUGGET_MESSAGE_BUFFER_SIZE];

    time(&current_time);
    time_info = localtime(&current_time);
    strftime(time_buffer, std::strlen(time_buffer), "%Y-%m-%d %H:%M:%S", time_info);

    if(verbose_level == 0){
        sprintf(log_message_buffer, "[%s] %s%s\n", time_buffer, CLIENT_INFO, message);
    }
    else if(verbose_level == 1){
        sprintf(log_message_buffer, "[%s] %s%s\n", time_buffer, CLIENT_WARNING, message);
    }
    else if(verbose_level == 2){
        sprintf(log_message_buffer, "[%s] %s%s\n", time_buffer, CLIENT_ERROR, message);
    }

    file.write(log_message_buffer, std::strlen(log_message_buffer));
    file.close();

    return 0;
}
```

შემდეგ, საჭიროა `client.cpp`-ში ლოგიკის აწყობა. მაგალითად, პროგრამის გამოყენება ასე უნდა ხდებოდეს:

	./client 192.168.200.100 8080 44b4c0

ამ შემთხვევაში, `./client` არის რიცხვობრივად მე-0 პარამეტრი, `192.168.200.100` 1-ელი და ა.შ. .

> **📝 საინტერესო ფაქტი:**  
> კომპიუტერები ათვლას 0-დან იწყებენ. ეს იმიტომ, რომ ისინი ბინარულ თვლის სისტემაში მუშაობენ და მეხსიერების დაზოგვის მიზნით, უმჯობესია თუ ყველანაირ შესაძლებლობას რიცხვად ჩავთვლით. ანუ, დავუშვათ მოცემულია ერთი ბაიტი(რვა ბიტი, ანუ რვა ელექტრონის ჩაწერის საშუალება), ჩვენ შეგვიძლია დავიწყოთ `0000 0000`-დან და ავიდეთ `1111 1111`-მდე. მეტს ფიზიკურად ვერ გამოვსახავთ, რადგან მხოლოდ რვა ბიტის გამოყენების უფლება გვაქვს ამ შემთხვევაში. ასე, რომ მეხსიერების დასაზოგად, `0000 0000`-ც გამოიყენება ათვლისთვის.

მჭირდება ისეთი კოდის დაწერა, რომელიც `argv[1]`-ს დააყენებს სერვერის IP მისამართად, ხოლო `argv[2]`-ს სერვერის პორტად.

> **📝 საინტერესო ფაქტი:**  
> `char* argv[]` საინტერესო კონცეფციაა. შესაძლებელია `int main(int argc, char* argv[])` ფუნქციის აწყობა, სადაც `argc` არგუმენტების რაოდენობა, ხოლო `argv[]` არგუმენტების დამკავებელი მასივი იქნება. ასე, შეგვიძლია ტერმინალიდან გადავცეთ გარკვეული არგუმენტები პროგრამას და მათთან ვიმუშაოთ. თუნდაც `nmap`-ის მაგალითში, როდესაც ვწერთ `nmap -sC -sV -A scanme.nmap.org`, აქ ყველა ტექსტი პარამეტრია. გასათვალისწინებელია ის ფაქტი, რომ რიცხვების გადაცემისას ისინი ტექსტად აღიქმება და მათი `int` მონაცემთა ტიპად გადაქცევისათვის საჭიროა `atoi()` ფუნქციის გამოყენება, რომელიც `#include <stdlib.h>`(C ვერსია) ან `#include <cstdlib>`(C++ ვერსია) ბიბლიოთეკაშია გაწერილი.

## კლიენტის მთავარი ლოგიკა

კლიენტის მთავარი ლოგიკა მდგომარეობს იმაში, რომ კლიენტმა უნდა ნახოს არის თუ არა `argc` `3`-ზე ნაკლები. თუ ასეა, მაშინ კოდი გამოიტანს ტექსტს:

	[USAGE]: argv[0] <IPv4> <PORT> <UserName>

`argv[0]`-ის ადგილზე, მისი მნიშვნელობა იქნება.

ამ ეტაპზე, `client.cpp` ფაილი ასე გამოიყურება:

```cpp
#include <stdio.h>
#include <stdlib.h>

#include <sys/socket.h>
#include <sys/types.h>
#include <arpa/inet.h>

#include <thread>

#include "client.hpp"

int main(int argc, char* argv[]){
    if(argc < 3){
        char local_message_buffer[LOG_MESSAGE_BUFFER_SIZE];
        sprintf(local_message_buffer, "%s%s <IPv4> <PORT> <UserName>\n", CLIENT_USAGE, argv[0]);

        printf(local_message_buffer);
        return 1;
    }

    int client_socket = socket(AF_INET, SOCK_STREAM, 0);
    sockaddr_in server_address;

    server_address.sin_family = AF_INET;
    server_address.sin_port = htons(atoi(argv[2]));
    server_address.sin_addr.s_addr = inet_addr(argv[1]);
}
```

ახლა დრო მოვიდა რაღაც-რაღაცების გადამოწმებისათვის საჭირო ლოგიკა ჩავაშენო კოდში. ანუ, მუშაობს თუ არა სოკეტი და შესაძლებელია თუ არა კავშირზე გასვლა. გარკვეული ლოგიკის დამატების შემდეგ, `client.cpp` ფაილი ასე გამოიყურება:

```cpp
#include <stdio.h>
#include <stdlib.h>

#include <sys/socket.h>
#include <sys/types.h>
#include <arpa/inet.h>

#include <unistd.h>

#include <thread>

#include "client.hpp"

int main(int argc, char* argv[]){
    if(argc < 3){
        char local_message_buffer[LOG_MESSAGE_BUFFER_SIZE];
        sprintf(local_message_buffer, "%s%s <IPv4> <PORT> <UserName>\n", CLIENT_USAGE, argv[0]);

        printf(local_message_buffer);
        return 1;
    }

    int client_socket = socket(AF_INET, SOCK_STREAM, 0);

    if(client_socket < 0){
        Log((char*)"Unable to define socket", 2);
        return 1;
    }

    sockaddr_in server_address;
    socklen_t szof_server_address = sizeof(server_address);

    server_address.sin_family = AF_INET;
    server_address.sin_port = htons(atoi(argv[2]));
    server_address.sin_addr.s_addr = inet_addr(argv[1]);

    if(connect(client_socket, reinterpret_cast<sockaddr*>(&server_address), szof_server_address) < 0){
        Log((char*)"Unable to connect to the server", 1);
        close(client_socket);
        return 1;
    }

    char username_buffer[USERNAME_BUFFER_SIZE];
    sprintf(username_buffer, "%s", argv[3]);

    send(client_socket, username_buffer, USERNAME_BUFFER_SIZE, 0);
}
```

`<unistd.h>` ბიბლიოთეკა `close();` ფუნქციისთვისაა შემოტანილი. ახლა ყველაზე მთავარი, წრედების შექმნაა.

ეს ორი ფუნქციაა `client.hpp` ფაილში:
```cpp
void ReadFromServer(int client_socket){
    while(true == true){
        char message_buffer[MESSAGE_BUFFER_SIZE];

        int received_bytes = recv(client_socket, message_buffer, MESSAGE_BUFFER_SIZE - 1, 0);

        if(received_bytes < 0){
            Log((char*)"Unable to receive message from the server", 1);
        }
        else if(received_bytes == 0){
            Log((char*)"Disconnected from the server(Lost connection)", 2);
            close(client_socket);
            break;
        }
        else{
            message_buffer[received_bytes] = '\0';
        }
    }
}

void SendToServer(int client_socket, char* message_to_be_sent){
    int result = send(client_socket, message_to_be_sent, strlen(message_to_be_sent), 0);

    if(result < 0){
        Log((char*)"Sending message to the server has failed", 2);
    }
    else if(result == 0){
        Log((char*)"Server closed the connection", 1);
        close(client_socket);
        return;
    }
    else{
        Log((char*)"Message sent successfully", 0);
    }
}
```

ორი წრედი უნდა გავაკეთო ახლა `client.cpp` ფაილში და ეს ფუნქციები გადავცე არგუმენტებად, მაგრამ ამ ყველაფერს ვერ გავაკეთებ სანამ `ncurses.h` ბიბლიოთეკით TUI-ს არ ავაწყობ, რომ მომხმარებელს ინფორმაციის შეყვანის საშუალება ჰქონდეს.

## საბოლოო კომპილაცია

როდესაც კოდი კომპილატორს გადავაწოდე, ასეთი გაფრთხილება დამიწერა:
![1.png](https://44b4c0.github.io/assets/img/posts/4/1.png)

ზოგადად უმჯობესია თუ ყოველგვარ გაფრთხილებას მოვიშორებ თავიდან კომპილაციის დროს, რადგან ეს ყველაფერი მომავალში შეიძლება დიდ პრობლემად გადაიქცეს.

ერთმა გაფრთხილებამაც კი შეიძლება საქმე შეუქმნას კოდის მუშაობას. ეს გაფრთხილება ამბობს, რომ უმჯობესია `printf();` ფუნქციას ფორმატით გადავცეთ ცვლადი:

	printf("%s", local_message_buffer);

ამის გასწორების შემდეგ, ისევ გადავაკომპილირე კოდი და ამჟამად ყველაფერმა კარგად იმუშავა(ასევე მომიწია `-pthread` flag-ის დამატება კომპილატორის ოფციებში, სტატიკური დალინკვისთვის).
