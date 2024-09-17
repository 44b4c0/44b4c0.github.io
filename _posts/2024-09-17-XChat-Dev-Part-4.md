---
title: "XChat development ნაწილი #4"
date: 2024-09-17 23:08:00 +0400
categories: [Programming]
tags: [Coding]
---

## TUI ინტერფეისის შენება

ძალიან არ მიყვარს ეს პროცესი, რადგან გრაფიკასთან მუშაობა არაა სასიამოვნო, მაგრამ კლიენტმა სერვერთან კომუნიკაცია კომფორტულად რომ შეძლოს, მომიწევს `TUI` ინტერფეისის აშენება.

ამ ეტაპზე ბრძანებების პარსინგიც არ მაინტერესებს, მთავარია კოდმა გამართულად იმუშაოს. ასე, რომ კოდის დიდი ნაწილის წაშლა მომიწევს(მაინც დაუმთავრებელია და არ მჭირდება), რომელიც თითქოსდა უნდა პარსავდეს მომხმარებლის მიერ შეყვანილ მონაცემებს. ერთადერთი რაც მჭირდება, არის მომხმარებლის მიერ შეყვანილი ინფორმაციის შემოწმება. ეს იმიტომ, რომ `Buffer Overflow` შეტევები ავირიდო თავიდან.
```
          Malicious shellcode/asm instructions
                           │                  
    Input buffer  EIP      │                  
         │         │       │                  
         ▼         │       │                  
┌─────────────────┐▼       ▼                  
│██████████████████████████████████...        
└─────────────────┘                           
                   ▲                          
                   │                          
             Overflowed data                  
```

> **📰 საინტერესო ფაქტი:**
> 
> `EIP` არის `32 bit` CPU register, რომელიც იშიფრება როგორც `Instruction Pointer`. ეს რეგისტრატორი იმ `asm` ინსტრუქციაზე(ანუ მანქანური კოდის `nugget`-ზე) მიუთითებს, რომელიც უნდა შესრულდეს. ანუ, გამოდის, რომ თუ მომხმარებლის მიერ შეყვანილ კოდს სწორად არ მოვექცევი, შესაძლოა `Buffer overflow` გამოვიწვიო, რამაც შესაძლოა კოდში ბოროტი `shellcode`-ის ინექცია გამოიწვიოს.

```cpp
while(true == true){
	char input_buffer[MESSAGE_SIZE];
	wclear(message_window);
	wrefresh(message_window);

	wgetstr(message_window, input_buffer);

	if(strlen(input_buffer) > MESSAGE_SIZE){
		continue;
	}

	wrefresh(chat_window);
}
```

ასე გამოიყურება კოდი. რა თქმა უნდა, ვამოწმებ არის თუ არა `input_buffer`-ში ჩაწერილი ინფორმაცია `MESSAGE_SIZE` მაკროზე მეტი(`#define MESSAGE_SIZE 1024`).

## Pointer პრობლემები

მოკლედ ბევრს აწუხებს `pointer`-ები, მაგრამ იდეაში არც ისე რთული ტოპიკია. პოინტერი უბრალო ცვლადია, რომელიც სხვა ცვლადის მეხისერების მისამართს იკავებს. მაგალითად:
```cpp
int x = 10;
int* x_ptr = &x;
```

ამის გამო, ასეთი რამ ხდება:
```
                          Random Access Memory                                
                                   │                                          
                                   │                                          
                                   │                                          
                                   │                                          
      ┌────────┐                   │                                          
      │        │                   │                                          
      │        │                   ▼                                          
┌───┬─▼─┬────┬─┼─┬───────────────────────────────────────────────────────────┐
│   │   │    │ │ │                                                           │
│   │1 0│    │ │ │                                                           │
│   │   │    │   │                                                           │
│   ├───┤    ├───┤                                                           │
│   │ x │    │ y │                                                           │
└───┴───┴────┴───┴───────────────────────────────────────────────────────────┘
```

აქ ის პრობლემა იქმნება, რომ როდესაც `WINDOW*` ცვლადები გადავეცი `CreateScreen();` და `DeleteScreen();` ფუნქციებს, მათ მიერ დაპროცესებული ცვლადების ცვლილება არ აისახა `main();` ფუნქციაში და `Segmentation fault(core dumped)` დამიწერა ტერმინალში. ამ პრობლემის მოსაგვარებლად, ამ ორი ფუნქციის პარამეტრებში, `WINDOW**` გახდება ყველა პარამეტრი, ხოლო ფუნქციებს ისინი ასე გადაეცემიან: `&client_window`. ასევე, ფუნქციებში მათი გამოყენებისას, მათ წინ `*` ექნებათ დართული: `*client_window`.

საბოლოო ჯამში, კოდი ასეთი გამოვიდა:
```cpp
int CreateScreen(WINDOW** chat_window, WINDOW** message_window, WINDOW** users_window){
    initscr();
    echo();
    keypad(stdscr, TRUE);

    int screen_height, screen_width;
    getmaxyx(stdscr, screen_height, screen_width);
    start_color();
    init_pair(1, COLOR_WHITE, COLOR_BLUE);

    int chat_window_height = screen_height - 3;

    *chat_window = newwin(chat_window_height, screen_width, 0, 0);
    *message_window = newwin(1, screen_width, screen_height - 2, 0);
    *users_window = newwin(1, screen_width, screen_height - 1, 0);

    wbkgd(*chat_window, COLOR_PAIR(1));
    wbkgd(*message_window, COLOR_PAIR(1));
    wbkgd(*users_window, COLOR_PAIR(1));

    // Draw borders around windows
    wborder(*chat_window, '|', '|', '-', '-', '+', '+', '+', '+');
    wborder(*message_window, '|', '|', '-', '-', '+', '+', '+', '+');
    wborder(*users_window, '|', '|', '-', '-', '+', '+', '+', '+');

    return 0;
}

int DeleteScreen(WINDOW** chat_window, WINDOW** message_window, WINDOW** users_window){
    int screen_height, screen_width;
    getmaxyx(stdscr, screen_height, screen_width);

    int chat_window_height = screen_height - 3;

    delwin(*chat_window);
    delwin(*message_window);
    delwin(*users_window);

    endwin();

    return 0;
}
```

![1.gif](https:///44b4c0.github.io/assets/img/posts/4/1.gif)

ხო, `.gif` ფაილი ცოტა ცუდ ხარისხშია(არ ვიცი რატომ), მაგრამ არაუშავს. ეს უბრალო დემონსტრაციაა იმის თუ რა ხდება, როდესაც კლიენტი ირთვება. როდესაც კლიენტი ამუშავდება, გამოჩნდება წარწერა, სადაც კლიენტი პროგრამის შესახებ შესაბამის ინფორმაციას გაიგებს. ფერებიც შესაცვლელია ჩატისთვის. შესაძლოა `theme`-ებიც დავამატო. ეს მოგვიანებით. ახლა მთავარია ფანჯრები ავაწყო.

## დინამიური რეზოლუცია

რა თქმა უნდა, საჭიროა დინამიური რეზოლუციის ქონა. ეს პრობლემა სულ ახლახანს აღმოვაჩნიე. გამოდის, რომ როდესაც ტერმინალის ფანჯრის ზომას ვცვლი, `client`-ის შიდა ფანჯრები ირევა. მინდა ისე იყოს ყველაფერი, რომ როდესაც ტერმინალის ფანჯარა გაიზრდება/დაპატარავდება, `client` ჩვეულებრივად აჰყვეს რეზოლუციის დინამიურ ცვლილებას.

ამის გაკეთებას `signal`-ით შევძლებ. `<signal.h>` ბიბლიოთეკაა ამისათვის საჭირო. შემდეგ, შემიძლია ავიღო ნებისმიერი ფუნქცია, და სიგნალს გადავცე ის. ასევე, შემიძლია მივუთითო თუ რა უნდა იყოს სიგნალი. ამ შემთხვევაში, სიგნალი ფანჯრის ზომის შეცვლა იქნება:
```cpp
signal(SIGWINCH, HandleGlobalResize);
```

ხოლო, `HandleGlobalResize` ფუნქცია იქნება ეს:
```cpp
void HandleGlobalResize(int resize_signal){
    endwin();

    clear();
    refresh();

    start_color();
    init_pair(1, COLOR_WHITE, COLOR_BLUE);

    int screen_height, screen_width;
    getmaxyx(stdscr, screen_height, screen_width);

    int chat_window_height = screen_height - 3;
    chat_window = newwin(chat_window_height, screen_width, 0, 0);
    message_window = newwin(1, screen_width, screen_height - 2, 0);
}
```

ამის გამო, გარკვეული ცვლადების `main();` ფუნქციიდან გამოტანა და მათი `client.hpp` ფაილში ჩასმა დამჭირდა. მაგალითად, ფანჯრის პოინტერების:
```cpp
WINDOW* chat_window;
WINDOW* message_window;
```

რამდენჯერმე დამჭირდა ტესტირების ჩატარება და საბოლოოდ, სიგნალი გამართულად მუშაობს. დინამიური რეზოლუცია ავაწყე. საბოლოოდ, ასეთი კოდი მივიღე:

`client.cpp:`
```cpp
#include <stdio.h>
#include <stdlib.h>

#include <sys/socket.h>
#include <sys/types.h>
#include <arpa/inet.h>

#include <string.h>

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

    int screen_init_res = CreateScreen(&chat_window, &message_window);

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

        wrefresh(chat_window);
    }

    int screen_del_res = DeleteScreen(&chat_window, &message_window);

    return 0;
}
```

`client.hpp:`
```cpp
#pragma once

#include <ncurses.h>
#include <signal.h>
#include <openssl/ssl.h>

/* Communication config */
#define MESSAGE_SIZE 1024

WINDOW* chat_window;
WINDOW* message_window;

int CreateScreen(WINDOW** chat_window, WINDOW** message_window){
    initscr();
    echo();
    keypad(stdscr, TRUE);

    int screen_height, screen_width;
    getmaxyx(stdscr, screen_height, screen_width);
    start_color();
    init_pair(1, COLOR_WHITE, COLOR_BLUE);

    int chat_window_height = screen_height - 3;

    *chat_window = newwin(chat_window_height, screen_width, 0, 0);
    *message_window = newwin(1, screen_width, screen_height - 2, 0);

    wbkgd(*message_window, COLOR_PAIR(1));

    return 0;
}

int DeleteScreen(WINDOW** chat_window, WINDOW** message_window){
    int screen_height, screen_width;
    getmaxyx(stdscr, screen_height, screen_width);

    int chat_window_height = screen_height - 3;

    delwin(*chat_window);
    delwin(*message_window);

    endwin();

    return 0;
}

void HandleGlobalResize(int resize_signal){
    endwin();

    // clear();
    refresh();

    start_color();
    init_pair(1, COLOR_WHITE, COLOR_BLUE);

    int screen_height, screen_width;
    getmaxyx(stdscr, screen_height, screen_width);

    int chat_window_height = screen_height - 3;
    chat_window = newwin(chat_window_height, screen_width, 0, 0);
    message_window = newwin(1, screen_width, screen_height - 2, 0);

    wbkgd(message_window, COLOR_PAIR(1));
    refresh();
}

int ReadFromServer(WINDOW* chat_window, int client_socket){
    while(true == true){

    }

    return 0;
}
```
