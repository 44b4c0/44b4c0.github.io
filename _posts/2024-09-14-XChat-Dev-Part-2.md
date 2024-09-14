---
title: "XChat დეველოპმენტი Part #2"
date: 2024-09-14 18:20:00 +0400
categories: [Programming]
tags: [Coding]
---

## სერვერის საბაზისო მხარე

ამ ვლოგში, სერვერის მხარეზე `Log(char* message, unsigned int verbose_level, bool is_initial)` ფუნქციას დავამატებ.

დროა რაღაც-რაღაცები გადავასწორო და თავისი ადგილი მივუჩინო. მაგალითად, მაკროები, კლასები და სხვადასხვა სახის ფუნქციები ყველა `*.hpp` ფაილებში უნდა იყოს გადანაწილებული. ასე, რომ ავდგები და `server.cpp` ფაილში რამდენიმე ცვლილებას შევიტან:

`server.cpp:`
```cpp
#include <stdio.h>
#include <stdlib.h>

#include <sys/socket.h>
#include <sys/types.h>
#include <arpa/inet.h>

#include <unistd.h>
#include <time.h>
#include <string.h>

#include <vector>
#include <string>

#include "server.hpp"

int main(void){
    std::vector<Client> client_vector;
    int server_socket; sockaddr_in server_address;

    server_socket = socket(AF_INET, SOCK_STREAM, 0);

    if(server_socket < 0){
        // ლოგირების ლოგიკა იქნება აქ
        return 1;
    }

    server_address.sin_family = AF_INET;
    server_address.sin_port = htons(SERVER_PORT);
    server_address.sin_addr.s_addr = inet_addr(SERVER_ADDRESS) | INADDR_ANY;

    if(bind(server_socket, reinterpret_cast<sockaddr*>(&server_address), sizeof(server_address)) < 0){
        // ლოგირების ლოგიკა იქნება აქ
        return 1;
    }

    if(listen(server_socket, 5) < 0){
        // ლოგირების ლოგიკა იქნება აქ
        return 1;
    }

    return 0;
}
```

`server.hpp:`
```cpp
#define SERVER_PORT 8080
#define SERVER_ADDRESS "0.0.0.0"

class Client {
    public:
    char* username;
    char* ip_address;
    unsigned int port;

    int client_fd;

    ~Client(){
        delete[] username;
        delete[] ip_address;
        if(username != nullptr){
            username = nullptr;
        }
        if(ip_address != nullptr){
            ip_address = nullptr;
        }
    }
};
```

კლიენტის კლასია აქ ყველაზე საინტერესო. საჭიროა, რომ ობიექტმა შეინახოს კლიენტის `username`(ანუ ზედმეტსახელი), `ip_address`(ანუ IP მისამართი, საიდანაც კავშირი შემოვიდა), `port`(ანუ პორტი, რომელიც კლიენტმა დასაკავშირებლად გამოიყენა) და `client_fd`(რომელიც არის client file descriptor და საშუალებას მომცემს კლიენტს რაღაცები გავუგზავნო ან პირიქით, მისგან მივიღო ინფორმაცია). დესტრუქტორი რაცაა(`~Client()`), ეგ `char*` პოინტერებს გაანადგურებს როდესაც კლიენტის წაშლა მომინდება.

დრო მოვიდა `Log()` ფუნქციაზე დავიწყო მუშაობა. საქმე ისაა, რომ სერვერის ლოგი სამი სახის ლოგ ჩანაწერს უნდა შეიცავდეს:

- `[INFO]:`
- `[WARNING]:`
- `[ERROR]:`

თითოეული მათგანი სხვადასხვა რამეს აღნიშნავს, მაგალითად: თუ კლიენტი წარმატებით დაკავშირდება სერვერთან, გამოვიყენებ `[INFO]:`-ს, რომელიც იქნება უბრალო სახის ინფორმაცია. იმ შემთხვევაში, თუ მაგალითად კლიენტისგან `username`-ის მიღება შეუძლებელია, ან კლიენტი უბრალოდ წყვეტს კავშირს ყოველგვარი ნორმის გარეშე, მაშინ გამოვიყენებ `[WARNING]:`-ს, ხოლო თუ სერვერზე კრიტიკული ხარვეზია, მაშინ `[ERROR]:`-ის გამოყენება იქნება საჭირო.

საბოლოო ჯამში, ასეთი სახის ფუნქციას მივიღებთ:

```cpp
int Log(char* message, unsigned int verbose_level, bool is_initial){
    std::ofstream file(LOG_FILE_PATH, std::ios::app);

    char log_message_buffer[LOG_MESSAGE_BUFFER_SIZE];

    time_t current_time;
    struct tm *time_info;
    char time_buffer[NUGGET_MESSAGE_BUFFER_SIZE];

    time(&current_time);
    time_info = localtime(&current_time);
    strftime(time_buffer, std::strlen(time_buffer), "%Y-%m-%d %H:%M:%S", time_info);

    if(is_initial == true){
        sprintf(log_message_buffer, "[%s] %sServer started listening...\n", time_buffer, SERVER_START);
        file.write(log_message_buffer, std::strlen(log_message_buffer));
        file.close();
        return 0;
    }
    if(verbose_level == 0){
        sprintf(log_message_buffer, "[%s] %s%s\n", time_buffer, SERVER_INFO, message);
    }
    else if(verbose_level == 1){
        sprintf(log_message_buffer, "[%s] %s%s\n", time_buffer, SERVER_WARNING, message);
    }
    else if(verbose_level == 2){
        sprintf(log_message_buffer, "[%s] %s%s\n", time_buffer, SERVER_ERROR, message);
    }

    file.write(log_message_buffer, std::strlen(log_message_buffer));
    file.close();

    return 0;
}
```

აქ რამდენიმე რამ ხდება. პირველ რიგში, მოცემულია `message` პარამეტრი, რომელიც შეიცავს იმ ტექსტს, რომლის Log ფაილში ჩაწერაც მინდა. შემდეგ მოცემულია `verbose_level` პარამეტრი, რომელიც გადაწყვეტს რა სახის Log ჩანაწერი იქნება ტექსტი:

- 0 --> INFO ℹ️
- 1 --> WARNING ☣️
- 2 --> ERROR 💥

საბოლოოდ, დავამატე `is_initial` პარამეტრი, რომელიც გვეტყვის ახლა ირთვება თუ არა სერვერი. თუ სერვერი თავიდან ეშვება, მაშინ ლოგ ფაილში დაემატება ახალი ინფორმაციის მაიდენტიფიცირებელი: `[START]:`, რომელიც მიუთითებს იმაზე, რომ სერვერი ახლა ჩაირთო. საბოლოო ჯამში, `server.hpp` იქნება ასეთი:

```cpp
#include <fstream>
#include <cstring>
#include <time.h>

#define LOG_FILE_PATH "./logs/server.log"

#define SERVER_PORT 8080
#define SERVER_ADDRESS "0.0.0.0"

#define SERVER_START "[START]: "

#define SERVER_INFO "[INFO]: "
#define SERVER_WARNING "[WARNING]: "
#define SERVER_ERROR "[ERROR]: "

#define LOG_MESSAGE_BUFFER_SIZE 8096
#define NUGGET_MESSAGE_BUFFER_SIZE 512

class Client {
    public:
    char* username;
    char* ip_address;
    unsigned int port;

    int client_fd;

    ~Client(){
        delete[] username;
        delete[] ip_address;
        if(username != nullptr){
            username = nullptr;
        }
        if(ip_address != nullptr){
            ip_address = nullptr;
        }
    }
};

int Log(char* message, unsigned int verbose_level, bool is_initial){
    std::ofstream file(LOG_FILE_PATH, std::ios::app);

    char log_message_buffer[LOG_MESSAGE_BUFFER_SIZE];

    time_t current_time;
    struct tm *time_info;
    char time_buffer[NUGGET_MESSAGE_BUFFER_SIZE];

    time(&current_time);
    time_info = localtime(&current_time);
    strftime(time_buffer, std::strlen(time_buffer), "%Y-%m-%d %H:%M:%S", time_info);

    if(is_initial == true){
        sprintf(log_message_buffer, "[%s] %sServer started listening...\n", time_buffer, SERVER_START);
        file.write(log_message_buffer, std::strlen(log_message_buffer));
        file.close();
        return 0;
    }
    if(verbose_level == 0){
        sprintf(log_message_buffer, "[%s] %s%s\n", time_buffer, SERVER_INFO, message);
    }
    else if(verbose_level == 1){
        sprintf(log_message_buffer, "[%s] %s%s\n", time_buffer, SERVER_WARNING, message);
    }
    else if(verbose_level == 2){
        sprintf(log_message_buffer, "[%s] %s%s\n", time_buffer, SERVER_ERROR, message);
    }

    file.write(log_message_buffer, std::strlen(log_message_buffer));
    file.close();

    return 0;
}
```

## server.cpp

რაც შეეხება მთავარ ფაილს, აქ რამდენიმე ადგილას ლოგ ჩანაწერის გენერატორია დასამატებელი. წინა პოსტში, ეს ადგილები კომენტარებით ამოვავსე და მარტო `return 1;` statement-ები დავურთე, ახლა კი დროა ეგ ნაწილი დავასრულო:

```cpp
#include <stdio.h>
#include <stdlib.h>

#include <sys/socket.h>
#include <sys/types.h>
#include <arpa/inet.h>

#include <unistd.h>
#include <string.h>

#include <vector>
#include <string>

#include "server.hpp"

int main(void){
    std::vector<Client> client_vector;
    int server_socket; sockaddr_in server_address;

    server_socket = socket(AF_INET, SOCK_STREAM, 0);

    if(server_socket < 0){
        Log((char*)"Unable to define socket", 2, false);
        return 1;
    }

    server_address.sin_family = AF_INET;
    server_address.sin_port = htons(SERVER_PORT);
    server_address.sin_addr.s_addr = inet_addr(SERVER_ADDRESS) | INADDR_ANY;

    if(bind(server_socket, reinterpret_cast<sockaddr*>(&server_address), sizeof(server_address)) < 0){
        Log((char*)"Unable to bind server", 2, false);
        return 1;
    }

    if(listen(server_socket, 5) < 0){
        char message_buffer[LOG_MESSAGE_BUFFER_SIZE];
        sprintf(message_buffer, "Unable to start listening on %s:%d", SERVER_ADDRESS, SERVER_PORT);
        Log(message_buffer, 2, false);
        return 1;
    }

    Log((char*)"NULL", 0, true);

    return 0;
}
```

კოდის კომპილაცია ვცადე და ყველაფერმა იდეალურად იმუშავა:
![1.png](https://44b4c0.github.io/assets/img/posts/2/1.png)

მწვანედ რომაა დაწერილი, `server`, ეგ არის executable `.ELF` ფაილი.
