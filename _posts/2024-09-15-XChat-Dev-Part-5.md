---
title: "XChat დეველოპმენტი Part #5"
date: 2024-09-15 18:55:00 +0400
categories: [Programming]
tags: [Coding]
---


## OpenSSL და შიფრაციის პრინციპები

კი, დრო მოვიდა `OpenSSL`-ის ინტეგრაცია მოვახდინო კოდში. ეს იმიტომ, რომ არ მინდა ვინმემ მარტივი `arp spoofing` შეტევით კლიენტების მესიჯები წაიკითხოს. აქედან გამომდინარე, საჭიროა ტრაფიკის შიფრაცია. დიაგრამის მოშველებაც შემიძლია:

```

+========+                                                                                                                                +========+
| Client |--------------------------------------------------------------------------------------------------------------------------------| Server |
+========+                                                                                                                                +========+

```

მარტივად გასაგებია მგონი ყველაფერი. დაშიფრული ტრაფიკის დროს, არსებობს ტუნელი, სადაც გადის მონაცემები და ამ ტუნელში ჩახედვა მხოლოდ კლიენტსა და სერვერს შეუძლიათ. რა თქმა უნდა, ეს ტუნელი აბსტრაქტული კონცეფციაა.

> **📝 საინტერესო ფაქტი:**  
> კრიპტოგრაფიაში არსებობს `გასაღებების` ცნება. თითოეულ ჰოსტზე(მანქანაზე) გენერირდება ორი სახის გასაღები: `public` და `private`(ანუ `საჯარო` და `კერძო`). როდესაც `A` მოწყობილობას `B` მოწყობილობასთან უსაფრთხო კავშირი სურს, ის `B` მოწყობილობას თავის საჯარო გასაღებს უგზავნის(იგივეს იმეორებს `B` მოწყობილობა `A` მოწყობილობასთან მიმართებაში). როდესაც `B` მოწყობილობა მიიღებს `A` მოწყობილობის `public` გასაღებს(ანუ `საჯაროს`), ის `A` მოწყობილობისთვის გასაგზავნ ინფორმაციას ამ გასაღებით დაშიფრავს და ჩვეულებრივად გაუგზავნის დაშიფრულ ინფორმაციას `A` მოწყობილობას. როდესაც `A` მოწყობილობა მიიღებს დაშიფრულ ინფორმაციას, ის თავისი `private` გასაღებით(ანუ `კერძო`) გაშიფრავს დაშიფრულ ინფორმაციას და შესაბამისი ლოგიკისდა მიხედვით დააპროცესებს მას(გააჩნია კოდის სექციაში რა წერია).

კოდში `OpenSSL`-ის ინტეგრაცია საკმაოდ მარტივია. ერთადერთი რაც უნდა გავაკეთო არის ის, რომ `OpenSSL`-ის ინიციალიზაცია მოვახდინო და შემდეგ ის `client_socket`-ს და `server_socket`-ს მოვდო(მივაბა).

ამისათვის, პირველ რიგში, უნდა დავაყენო `libssl-dev` პაკეტი და შემდეგ კოდში უნდა შევიტანო ორი ბიბლიოთეკა:

```cpp
#include <openssl/ssl.h>
#include <openssl/err.h>
```

თავდაპირველად საჭიროა `OpenSSL`-ის ინიციალიზაცია, რისი გაკეთებაც შემდეგნაირად შეიძლება:

```cpp
SSL_library_init();
OpenSSL_add_all_algorithms();
SSL_load_error_strings();
```

პირველი ფუნქცია, ბიბლიოთეკის ინიციალიზაციას ემსახურება, მეორე ალგორითმი კრიპტოგრაფიულ ალგორითმებს ჩატვირთავს, ხოლო ბოლო ფუნქცია `Error` სტრინგებს(ტექსტებს) ჩატვირთავს, იმისთვის რომ `Error`-ების შემთხვევაში მათი გამოყენება შევძლო.

შემდეგ, საჭიროა `SSL_CTX`-ის, ანუ `SSL Context`-ის და მისი მეთოდის შექმნა. უფრო სწორად, `SSL Method`-ს და მისი კონტექსტის. ამ ყველაფერს ასე გავაკეთებ:

```cpp
const SSL_METHOD* ssl_method = SSLv23_client_method();
SSL_CTX *ssl_context = SSL_CTX_new(ssl_method);
```

როდესაც ეს მოხდება, საჭიროა `ssl_context`-ის გადამოწმება:

```cpp
if(!ssl_context){
	Log((char*)"Unable to initialize OpenSSL", 2);
	return 1;
}
```

ანუ, თუ `SSL`-ის კონტექსტის ინიციალიზება ვერ მოხერხდა, მაშინ შეცდომას დავლოკავ და პროგრამას გავაჩერებ, რადგან დაუშიფრავი ტრაფიკით დაშიფრულ ტრაფიკს(სერვერის ტრაფიკს) ვერ მიუერთდება პროგრამა.

## სოკეტის შიფრაცია

ამ ყველაფრის გაკეთების შემდეგ, შემიძლია სოკეტის შექმნა და `SSL`-ის მის ნაკადზე მიმაგრება. სანამ ამას გავაკეთებ, მჭირდება ისეთი რამ რაც სოკეტს მიემაგრება:

```cpp
SSL* ssl = SSL_new(ssl_context);
SSL_set_fd(ssl, client_socket);
```

რა თქმა უნდა, მისი მიბმის შემოწმება არ უნდა გამომრჩეს:

```cpp
if(SSL_connect(ssl) < 0 || SSL_connect(ssl) == 0){
	Log((char*)"Unable to connect SSL to the socket's stream", 2);
	close(client_socket);
	return 1;
}
```

იმ შემთხვევაში თუ ესეც წარმატებით დასრულდა, მაშინ შემიძლია ტუნელში მონაცემების ჩაყრა/ამოყრა დავიწყო. ანუ, სტანდარტულ `send()`/`recv()` ფუნქციებს კი აღარ გამოვიყენებ, არამედ უშუალოდ `SSL`-სთვის განკუთვნილ ფუნქციებს ჩავრთავ საქმეში.

მაგალითად, აი ასე შეიცვლება კოდი:
```cpp
SSL_write(ssl, username_buffer, strlen(username_buffer));
// send(client_socket, username_buffer, USERNAME_BUFFER_SIZE, 0);
```

ქვემოთ რაც იყო ამოვაკომენტარე და ახლით ჩავანაცვლე. იდეაში ორივე იგივე ფუნქციას ასრულებს, მაგრამ ახლა ინფორმაცია დაშიფრულია.

ახლა, კომპილაციის დროს ასეთი ბრძანების დაწერა მიწევს:

	g++ client.cpp -o client -pthread -lssl -lcrypto

მთავარი ფუნქციის ბოლოს, ამას დავამატებ, რათა ყველაფერი გასუფთავდეს პროგრამის გათიშვის წინ:

```cpp
try{
	close(client_socket);
	SSL_shutdown(ssl);
	SSL_free(ssl);
	SSL_CTX_free(ssl_context);
}
catch (...){
	SSL_shutdown(ssl);
	SSL_free(ssl);
	SSL_CTX_free(ssl_context);
}

return 0;
```

## სერვერის მხარე

დროა `OpenSSL` სერვერზეც ავაწყო, რადგან თუ ამას არ გავაკეთებ კავშირი არ შედგება.

სერვერის მხარეს უკვე საჭიროა გასაღებების დაგენერირება:

	openssl req -x509 -nodes -days 365 -newkey rsa:2048 -keyout server.key -out server.crt

`Runtime`-ში გავტესტე ყველაფერი და არ მუშაობს. ასე, რომ სერვერის ლოგიკის თავიდან გადაწერა მომიწევს. ისე უნდა გავაკეთო, რომ გარკვეული `pointer`-ების ახალ წრედში გადაცემა აღარ მომიწიოს.

> **📝 საინტერესო ფაქტი:**  
> ყველა პროცესს საკუთარი `virtual address space` აქვს. ასევე ყველა წრედს გამოეყოფა საკუთარი მეხსიერება(გარკვეული თვალსაზრისით). ასე, რომ შეუძლებელია ერთი წრედიდან მეორე წრედის მეხსიერების მისამართს(`memory address`-ს) პირდაპირ მივწვდე და ვუთხრა CPU-ს, რომ ეს ლოკაცია გაწმინდოს `SSL_free(ssl)` ოპერაციის გამოყენებით.

ფაქტია, სერვერის კოდის თავიდან გადაწერა და მასში დიდი ცვლილებების შეტანა მომიწევს.

დავწერე ახალი `ReadFromClient()` ფუნქცია:

```cpp
int ReadFromClient(int server_socket, sockaddr_in server_address, std::vector<Client> client_vector){
    SSL* ssl;
    if(client_vector.size() >= 200){
        sockaddr_in client_address;
        socklen_t szof_client_address = sizeof(client_address);
        int client_fd = accept(server_socket, reinterpret_cast<sockaddr*>(&client_address), &szof_client_address);

        if(client_fd < 0){
            Log((char*)"Unable to accept client", 2, false);
            SSL_free(ssl);
            return 1;
        }

        SSL_set_fd(ssl, client_fd);

        if(SSL_accept(ssl) <= 0){
            Log((char*)"Unable to accept SSL client", 2, false);
            SSL_free(ssl);
            return 1;
        }

        char username_buffer[USERNAME_BUFFER_SIZE];

        int received_bytes = SSL_read(ssl, username_buffer, USERNAME_BUFFER_SIZE - 1);

        if(received_bytes < 0){
            Log((char*)"Unable to receive nickname from SSL client", 2, false);
            close(client_fd);
            SSL_free(ssl);
            return 1;
        }
        else if(received_bytes == 0){
            Log((char*)"SSL Client disconnected", 1, false);
            close(client_fd);
            SSL_free(ssl);
            return 1;
        }
        else{
            username_buffer[received_bytes] = '\0';
            char data_send_buffer[SPEC_CODE_BUFFER_SIZE];

            sprintf(data_send_buffer, "%s", SERVER_FULL_SPEC_CODE);
            SSL_write(ssl, data_send_buffer, SPEC_CODE_BUFFER_SIZE);
            close(client_fd);
            SSL_free(ssl);
            return 1;
        }
    }
    else{
        sockaddr_in client_address;
        socklen_t szof_client_address = sizeof(client_address);
        int client_fd = accept(server_socket, reinterpret_cast<sockaddr*>(&client_address), &szof_client_address);

        SSL_set_fd(ssl, client_fd);

        if(SSL_accept(ssl) <= 0){
            Log((char*)"Unable to accept SSL client", 2, false);
            close(client_fd);
            SSL_free(ssl);
            return 1;
        }

        char username_buffer[USERNAME_BUFFER_SIZE];

        int received_bytes = SSL_read(ssl, username_buffer, USERNAME_BUFFER_SIZE - 1);

        if(received_bytes < 0){
            Log((char*)"Unable to receive username from SSL client", 2, false);
            close(client_fd);
            SSL_free(ssl);
            return 1;
        }
        else if(received_bytes == 0){
            Log((char*)"SSL client disconnected", 2, false);
            close(client_fd);
            SSL_free(ssl);
            return 1;
        }
        else{
            Client new_client;
            new_client.client_fd = client_fd;
            new_client.username = username_buffer;
            new_client.ip_address = inet_ntoa(client_address.sin_addr);
            new_client.port = ntohs(client_address.sin_port);
            new_client.ssl = ssl;

            client_vector.push_back(new_client);

            while(true == true){
                char message_buffer[MESSAGE_BUFFER_SIZE];
                int received_bytes = SSL_read(ssl, message_buffer, MESSAGE_BUFFER_SIZE - 1);
                if(received_bytes < 0){
                    Log((char*)"Unable to receive message from SSL client", 2, false);
                    close(client_fd);
                    SSL_free(ssl);
                    RemoveClient(client_vector, client_fd);
                    return 1;
                }
                else if(received_bytes == 0){
                    Log((char*)"SSL client disconnected", 1, false);
                    close(client_fd);
                    SSL_free(ssl);
                    RemoveClient(client_vector, client_fd);
                    return 1;
                }
                else{
                    if(strcmp(message_buffer, CLIENT_EXIT)){
                        RemoveClient(client_vector, client_fd);
                        char data[MESSAGE_BUFFER_SIZE];
                        sprintf(data, "%s Disconnected", new_client.username);
                        for(const auto &client : client_vector){
                            if(client.ssl != ssl){
                                SSL_write(client.ssl, data, strlen(data));
                            }
                        }
                        return 1;
                    }
                    for(const auto &client : client_vector){
                        if(client.ssl != ssl){
                            SSL_write(client.ssl, message_buffer, MESSAGE_BUFFER_SIZE);
                        }
                    }
                }
            }
        }
    }

    

    return 0;
}
```

პრობლემა ისევ წრედების შექმნაა.
