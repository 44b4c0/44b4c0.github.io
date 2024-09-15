---
title: "XChat დეველოპმენტი Part #3"
date: 2024-09-14 23:24:00 +0400
categories: [Programming]
tags: [Coding]
---

## კლიენტის მიღება/დაპროცესება

დრო მოვიდა კლიენტები მივიღო სერვერით. სანამ `OpenSSL`-ს გამოვიყენებ, ტრაფიკი დაუშიფრავი იქნება, მაგრამ როდესაც მთავარ ლოგიკას მოვრჩები მას აუცილებლად დავამატებ კოდში. ასევე ასაწყობია TUI ინტერფეისი `ncurses.h` ბიბლიოთეკის დახმარებით.

ჯერჯერობით ყურადღებას მაინც კლიენტისადა სერვერის კომუნიკაციაზე გავამახვილებ. მოკლედ, `server.hpp` ფაილში უნდა შეიქმნას `HandleClient()` და `ReadFromClient()` ფუნქციები:

```cpp
int ReadFromClient(int client_fd, sockaddr_in server_address, char* &recv_message_buffer){
    return 0;
}

int HandleClient(int server_socket, sockaddr_in server_address, std::vector<Client> client_vector){
    return 0;
}
```

აქ მთავარი `HandleClient` ფუნქციაა. მაგრამ სანამ ამ ფუნქციას გავუშვებ, საჭიროა მთავარ ფუნქციაში შემოწმდეს შესაძლებელია თუ არა ახალი კლიენტის მიღება და ამისდა მიხედვით უნდა მივიღო შესაბამისი ზომები:

```cpp
while(true == true){
    if(client_vector.size() >= 200){
        sockaddr_in client_address;
        socklen_t szof_client_address = sizeof(client_address);
        int client_fd = accept(server_socket, reinterpret_cast<sockaddr*>(&client_address), &szof_client_address);
            
        char data_buffer[NICKNAME_BUFFER_SIZE];

        int received_bytes = recv(client_fd, data_buffer, NICKNAME_BUFFER_SIZE - 1, 0);

        if(received_bytes < 0){
            Log((char*)"Couldn't receive nickname from the client", 1, false);
            close(client_fd);
        }
        else if(received_bytes == 0){
            Log((char*)"Client disconnected", 1, false);
            close(client_fd);
        }
        else{
            data_buffer[received_bytes] = '\0';
            char data_send_buffer[SPEC_CODE_BUFFER_SIZE];

            send(client_fd, data_send_buffer, SPEC_CODE_BUFFER_SIZE, 0);
            close(client_fd);
        }
    }
    else{
        std::thread handle_client_thread(HandleClient, server_socket, server_address);
    }
}
```

ეს კოდის ნაწილი შეამოწმებს სავსეა თუ არა კლიენტთა სია და იმის მიხედვით მიიღებს გადაწყვეტილებას.

## შეცდომების გასწორება?!

კოდი გადავაკომპილირე და იმუშავა, მაგრამ Runtime-ში მქონდა ასეთი პრობლემა:
![1.png](https://44b4c0.github.io/assets/img/posts/3/1.png)

ბევრი ვიფიქრე და დიდი ხანი ვიყავი ამაზე გაჭედილი. მეგონა, რომ ბიბლიოთეკებში გარკვეული ფუნქციები სწორად არ მუშაობდა და მაგიტომ ხდებოდა ეს, მაგრამ შემდეგ მივხვდი, რომ კოდის ლოგიკაა არეული. საქმე ისაა, რომ პროგრამა უსასრულოდ ამოწმებს `client_vector` ვექტორს და თუ ის სავსე არაა, მაშინ ის ახალ წრედს ქმნის. გამოდის, რომ უსასრულოდ ბევრი წრედი იქმნება და ამის გამო კერნელი პროცესს მიკლავს, იმისთვის, რომ რაღაც პრობლემა არ შეექმნას ოპერაციულ სისტემას. აქედან გამომდინარე საჭიროა ამ პრობლემის გამოსწორება:

```cpp
while(true == true){
    if(client_vector.size() >= 200){
        sockaddr_in client_address;
        socklen_t szof_client_address = sizeof(client_address);
        int client_fd = accept(server_socket, reinterpret_cast<sockaddr*>(&client_address), &szof_client_address);

        if(client_fd < 0){
            Log((char*)"Unable to accept client", 2, false);
        }
            
        char data_buffer[USERNAME_BUFFER_SIZE];

        int received_bytes = recv(client_fd, data_buffer, USERNAME_BUFFER_SIZE - 1, 0);

        if(received_bytes < 0){
            Log((char*)"Couldn't receive nickname from the client", 1, false);
            close(client_fd);
        }
        else if(received_bytes == 0){
            Log((char*)"Client disconnected", 1, false);
            close(client_fd);
        }
        else{
            data_buffer[received_bytes] = '\0';
            char data_send_buffer[SPEC_CODE_BUFFER_SIZE];

            send(client_fd, data_send_buffer, SPEC_CODE_BUFFER_SIZE, 0);
            close(client_fd);
        }
    }
    else{
        sockaddr_in client_address;
        socklen_t szof_client_address = sizeof(client_address);
        int client_fd = accept(server_socket, reinterpret_cast<sockaddr*>(&client_address), &szof_client_address);

        char username_buffer[USERNAME_BUFFER_SIZE];

        Client new_client;
        new_client.client_fd = client_fd;
        new_client.username = username_buffer;
        new_client.ip_address = inet_ntoa(client_address.sin_addr);
        new_client.port = ntohs(client_address.sin_port);

        client_vector.push_back(new_client);

        std::thread handle_client_thread(HandleClient, server_socket, server_address);
    }
}
```

ასევე `server.hpp` ფაილში მომიწია ფუნქციების ჩასწორება:

```cpp
int ReadFromClient(int client_fd, sockaddr_in server_address, char* &recv_message_buffer){
    return 0;
}

int HandleClient(int client_fd, sockaddr_in server_address){
    return 0;
}
```

შედარებით კარგი აღწერისთვის, `NICKNAME_BUFFER_SIZE` შევცვალე `USERNAME_BUFFER_SIZE`-ით. მაგრამ ამ მხრივ ყველაფერი იგივე რჩება სახელის გარდა.

რეკომპილაციის შემდეგ, სერვერი გამართულად მუშაობს. თეორიულად, ახლა აღარც `Thread pool`-ის დაწერა მომიწევს და აღარც ხელით მომიწევს `Thread vector`-ის გაწერა, რომლის კონტროლიც განცალკევებული `thread`-დან მომიწევს.

ახლა ერთადერთ გამოწვევად რჩება ისეთი რამის მოფიქრება, რაც `client_vector`-ს გაწმენდს. გაწმენდა იმიტომაა საჭირო, რომ ისეთი კლიენტები(მათი ობიექტები) მოვაშორო სერვერის ვექტორს თავიდან, რომლებმაც სისტემასთან გაწყვიტეს კავშირი. მსგავსი რამ დასაშვებია. შესაძლოა კლიენტების ვექტორი გადავაწოდო ორივე ფუნქციას და თანმიმდევრულად შევამოწმო თითოეული ვექტორის წევრი.

> **📝 საინტერესო ფაქტი:**  
> `for` ციკლით შესაძლებელია მასივში, ვექტორში, სიაში არსებული ნივთების გადამოწმება. ანუ, შესაძლებელია თითოეული ნივთის სათითაოდ შემოწმება. მაგალითად, პითონს აქვს ასეთი ხერხი: `for client in clients:` და ამის შემდეგ `client` ხდება ობიექტი, რომელზე მუშაობაც შეგვიძლია. ეს მაგალითი ამ სერიის პირველ პოსტშია მოყვანილი, სადაც ჩემი ძველი რეპოზიტორიიდან წამოღებული პითონში დაწერილი პროექტი განვიხილე: [XChat დეველოპმენტი ნაწილი #1](https://44b4c0.github.io/posts/XChat-Dev-Part-1/)

## კიდევ უფრო მეტი ცვლილება

საუკეთესო ვარიანტი მაინც ახალი სისტემა იქნება. მხოლოდ ერთი ფუნქცია დავტოვე, `ReadFromClient()`, რადგან მთავარ ფუნქციაში ვაკეთებ ახლა `HandleClient()` ფუნქციის გასაკეთებელს. ასევე ბევრი საინტერესო რამ დავამატე ფუნქციებს.

ეს არის განახლებული კოდის ნაწილი `main()` ფუნქციაში:
```cpp
while(true == true){
    if(client_vector.size() >= 200){
        sockaddr_in client_address;
        socklen_t szof_client_address = sizeof(client_address);
        int client_fd = accept(server_socket, reinterpret_cast<sockaddr*>(&client_address), &szof_client_address);

        if(client_fd < 0){
            Log((char*)"Unable to accept client", 2, false);
        }
            
        char data_buffer[USERNAME_BUFFER_SIZE];

        int received_bytes = recv(client_fd, data_buffer, USERNAME_BUFFER_SIZE - 1, 0);

        if(received_bytes < 0){
            Log((char*)"Couldn't receive nickname from the client", 1, false);
            close(client_fd);
        }
        else if(received_bytes == 0){
            Log((char*)"Client disconnected", 1, false);
            close(client_fd);
        }
        else{
            data_buffer[received_bytes] = '\0';
            char data_send_buffer[SPEC_CODE_BUFFER_SIZE];

            send(client_fd, data_send_buffer, SPEC_CODE_BUFFER_SIZE, 0);
            close(client_fd);
        }
    }
    else{
        sockaddr_in client_address;
        socklen_t szof_client_address = sizeof(client_address);
        int client_fd = accept(server_socket, reinterpret_cast<sockaddr*>(&client_address), &szof_client_address);

        char username_buffer[USERNAME_BUFFER_SIZE];

        Client new_client;
        new_client.client_fd = client_fd;
        new_client.username = username_buffer;
        new_client.ip_address = inet_ntoa(client_address.sin_addr);
        new_client.port = ntohs(client_address.sin_port);

        client_vector.push_back(new_client);

        std::thread handle_client_thread(ReadFromClient, client_fd, server_address, std::ref(client_vector));
    }
}
```

შემდეგ, `server.hpp` ფაილიდან, როგორც უკვე ვთქვი, წავშალე `HandleClient()` ფუნქცია და მხოლოდ `ReadFromClient()` ფუნქცია დავტოვე. რაც ნიშნავს, რომ შემცირდა ე.წ. `Code Base` და ამით ყველაფერი ბევრად უკეთესობისკენ წავა.

აი ეს ცვლილებები შევიტანე `server.hpp` ფაილში:
```cpp
void RemoveClient(std::vector<Client> client_vector, int client_fd){
    auto it = std::remove_if(client_vector.begin(), client_vector.end(),
    [client_fd](const Client& client) {
        return client.client_fd == client_fd;
    });
    client_vector.erase(it, client_vector.end());
}

int ReadFromClient(int client_fd, sockaddr_in server_address, std::vector<Client> client_vector){
    while(true == true){
        char message_buffer[MESSAGE_BUFFER_SIZE];
        int received_bytes = recv(client_fd, message_buffer, MESSAGE_BUFFER_SIZE - 1, 0);
    
        if(received_bytes < 0){
            Log((char*)"Unable to receive message from the client", 1, false);
            close(client_fd);
            break;
        }
        else if(received_bytes == 0){
            Log((char*)"Client disconnected", 1, false);
            close(client_fd);
            break;
        }
        else{
            message_buffer[received_bytes] = '\0';

            for(const auto& client : client_vector){
                if(client.client_fd != client_fd){
                    send(client.client_fd, message_buffer, MESSAGE_BUFFER_SIZE, 0);
                }
            }
        }
    }

    RemoveClient(std::ref(client_vector), client_fd);

    close(client_fd);

    return 0;
}
```

პირველი ფუნქცია, `RemoveClient()` იღებს ორ პარამეტრს და `client_fd`-ზე დაყრდნობით აშორებს კლიენტს ვექტორიდან. შესაძლოა, მოგვიანებით, ეს ბევრად უკეთესი ვარიანტით შევცვალო, რომელიც მომხმარებლის `username`-ით ამოშლას ითვალისწინებს.
