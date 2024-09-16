---
title: "XChat development ნაწილი #3"
date: 2024-09-16 19:29:00 +0400
categories: [Programming]
tags: [Coding]
---

## რის გამოყენებას ვაპირებ(პროექტის ზოგადი სტრუქტურა)

როდესაც საქმე კლიენტის მხარეს ეხება, აქ ბევრად სხვანაირადაა საქმე. პირველ რიგში, რა თქმა უნდა, მჭირდება `SSL/socket`, რათა სერვერთან კომუნიკაცია შევძლო. ასე, რომ ჯერ ამის ლოგიკას ავაწყობ. მაგრამ, როდესაც საქმე `TUI` ინტერფეისს ეხება, აქ `ncurses.h` ბიბლიოთეკა შემოდის საქმეში. მარტივად რომ ვთქვა:
```
                                               ┌─────────┐  ┌────────────────┐ 
                                       ┌──────►│Thread #1├─►│WriteToServer();│ 
                                       │       └─────────┘  └────────────────┘ 
┌────┐           ┌─────────┐           │                                       
│User├──────────►│ncurses.h├───────────┤           ┌────Output───────┐         
└────┘           └────┬────┘           │           ▼                 │         
                      │                │       ┌─────────┐  ┌────────┴────────┐
                      │                └──────►│Thread #2├─►│ReadFromServer();│
                      │                        └───┬─────┘  └─────────────────┘
                      │                            │                           
                      │                            │                           
                      │                            │                           
                      ▼                            │                           
             ┌──────────────────┐                  │                           
             │TUI based graphics│◄─────────────────┘                           
             └──────────────────┘                                              
```

დაახლოებით ასეთ რამესთან მაქვს საქმე, კლიენტის მონახაზი რომ გავაკეთე. ეს შედარებით მარტივი იქნება, უბრალოდ `ncurses.h`-ით `TUI` ინტერფეისის აწყობა ბევრად ართულებს საქმეს. მაგრამ ეს ყველაფერი მოგვარებადია.

## კოდის წერა

ბევრი რამ ემთხვევა როდესაც საქმე `server.cpp`-ს წერასა და `client.cpp`-ს წერას ეხება. მაგალითად, ზუსტად ემთხვევა სოკეტის და `SSL`-ის ინიციალიზაციის მეთოდები და ზოგადი ლოგიკის პრინციპი, მაგრამ განსხვავება აქ ისაა, რომ სერვერისგან განსხვავებით, კლიენტს სხვა მეთოდების გამოყენება მოუწევს ბიბლიოთეკებიდან.

`client.cpp:`
```cpp
#include <stdio.h>
#include <stdlib.h>

#include <sys/socket.h>
#include <sys/types.h>
#include <arpa/inet.h>

#include <ncurses.h>

#include <thread>

#include "client.hpp"

int main(){
    return 0;
}
```

საწყის ეტაპზე ეს არის. ახლა მთავარია `TUI` ავაწყო.

## TUI `ncurses.h` ბიბლიოთეკით

რა თქმა უნდა, ამ ბიბლიოთეკას საკუთარი მეთოდები გააჩნია. ზოგადად, ნებისმიერი ბიბლიოთეკის გამოყენების დროს არსებობს განსხვავებული მეთოდები და ფუნქციები, რომელთა სწავლაც მიწევს, რადგან ამის გარეშე კოდს ვერ დავწერ. რა თქმა უნდა, შეიძლება საკუთარი ბიბლიოთეკის აწყობა, მაგრამ ეს ყველაფერი დროს ძალიან გაწელავს. ასე, რომ, საქმეს `ncurses.h` ბიბლიოთეკით შევუდგები:

```cpp
int main(){
    int client_socket;
    sockaddr_in server_address;

    WINDOW* chat_window;
    WINDOW* message_window;
    WINDOW* users_window;

    int screen_init_res = CreateScreen(chat_window, message_window, users_window);

    if(screen_init_res != 0){
        return 1;
    }

    char input[512];
    while(true == true){
        wclear(message_window);
        mvwprintw(message_window, 0, 0, "Type here: ");
        wrefresh(message_window);

        wgetstr(message_window, input);
        if(strcmp(input, "/quit") == 0){
            break;
        }

        wrefresh(chat_window);
    }

    int screen_del_res = DeleteScreen(chat_window, message_window, users_window);

    return 0;
}
```

თქმა არ სჭირდება იმას, რომ `CreateScreen` და `DeleteScreen` ფუნქციების კულისებს მიღმა ბევრად მეტი რამ არსებობს ვიდრე ეს ერთი შეხედვით ჩანს:

`client.hpp:`
```cpp
int CreateScreen(WINDOW* chat_window, WINDOW* message_window, WINDOW* users_window){
    initscr();
    noecho();
    keypad(stdscr, TRUE);

    int screen_height, screen_width;
    getmaxyx(stdscr, screen_height, screen_width);

    int chat_window_height = screen_height - 3;

    chat_window = newwin(chat_window_height, screen_width, 0, 0);
    message_window = newwin(1, screen_width, screen_height - 2, 0);
    users_window = newwin(1, screen_width, screen_height - 1, 0);

	return 0;
}

int DeleteScreen(WINDOW* chat_window, WINDOW* message_window, WINDOW* users_window){
    int screen_height, screen_width;
    getmaxyx(stdscr, screen_height, screen_width);

    int chat_window_height = screen_height - 3;

    chat_window = newwin(chat_window_height, screen_width, 0, 0);
    message_window = newwin(1, screen_width, screen_height - 2, 0);
    users_window = newwin(1, screen_width, screen_height - 1, 0);

    delwin(chat_window);
    delwin(message_window);
    delwin(users_window);

    endwin();

	return 0;
}
```

ამჟამად ეს სატესტო მოდელია, რომელიც მარტივ ფანჯრებს ქმნის და შიგნით მუშაობის საშუალებას მაძლევს. საკმაოდ საინტერესო და მარტივი სინტაქსი აქვს `ncurses.h` ბიბლიოთეკის ფუნქციებსა და მეთოდებს.

როდესაც სოკეტი და `SSL` ამუშავდება(ანუ კავშირი შედგება), შეიქმნება ორი `Thread`, რომლებიც გაუშვებენ `ReadFromServer();` და `WriteToServer();` ფუნქციებს. დიდი ალბათობით აქაც მომიწევს `Mutex`-ების გამოყენება, რათა `chat_window` გავაკონტროლო, მაგრამ შეიძლება ეს საერთოდაც არ იყოს საჭირო.

## უსასრულო ციკლი

კლიენტის მხარეს, თავიდანვე უსასრულო ციკლის გამოყენება მიწევს, `ncurses.h` ბიბლიოთეკის გამო. ასევე, საჭიროა `inline` ბრძანებების `parsing`, ასე, რომ მომიწევს კავშირის გაბმამდე ბრძანებები გავპარსო.
```cpp
while(true == true){
    char input_buffer[MESSAGE_SIZE];
    wclear(message_window);
    mvwprintw(message_window, 0, 0, "Type here: ");
    wrefresh(message_window);

    wgetstr(message_window, input_buffer);

    wrefresh(chat_window);
}
```

სწორედ აქ ჩაშენდება ყველაფერი. სამომავლოდ, გავწერ რამდენიმე ფუნქციას, რომლებიც ბრძანებების შესაბამისად იმოქმედებენ.
