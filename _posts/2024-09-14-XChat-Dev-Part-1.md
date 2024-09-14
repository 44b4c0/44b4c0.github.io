---
title: "XChat დეველოპმენტი Part #1"
date: 2024-09-14 16:37:00 +0400
categories: [Programming]
tags: [Coding]
---

# XChat დეველოპმენტი Part #1

## შესავალი

რამდენიმე წლის წინ დავწერე Terminal based ჩატი პითონში(Python) და რამდენიმე დღის წინ გამახსენდა, რომ ეგ პროექტი კიდევ მქონდა გადანახული ძველი GitHub-ის ექაუნთზე. წავედი და დავკლონე მთელი რეპოზიტორია. თვითონ ჩატის კონცეფცია საკმაოდ საინტერესოა, მაგრამ სამწუხაროდ ბევრი ნაკლი აქვს.

## ძველი პროექტის ანალიზი

მოდი ჯერ ძველი პროექტი გავაანალიზოთ:

`client.py:`
```python
# Tring to import libraries
try:
    import socket
    import threading
except:
    # ERROR MSG
    print(f'[ERROR]: Can\'t import libraries...')
    exit()

# Adding SysINFO Texts
nickIsAlreadyUsed = '[ERROR]: Nickname is already used...'
youJoinedTheServer = '[INFO]: You joined the server!'
exitINFO = '//EXIT//'

# Adding vars
HOST = input('Enter IPv4: ')

# Checking HOST
if HOST[0:7] == '192.168' or HOST[0:3] == '127' or HOST[0:2] == '10':
    PORT = 4444
else:
    PORT = input('Enter Port: ')

Nickname = input('Enter nickname: ')
Client_Socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)

# Tring to connect to the server
try:
    Client_Socket.connect((str(HOST), int(PORT)))
except:
    # ERROR MSG
    print(f'[ERROR]: Can\'t find the server...')
    exit()

# Sending nickname to the server
Client_Socket.send(Nickname.encode('utf-8'))

# Receiving Answer
receivedAnswer = Client_Socket.recv(2048)
decodedReceivedAnswer = receivedAnswer.decode('utf-8')

# Checking decodedReceivedAnswer
if decodedReceivedAnswer == nickIsAlreadyUsed:
    # Printing decodedReceivedAnswer
    print(decodedReceivedAnswer)
    exit()
elif decodedReceivedAnswer == youJoinedTheServer:
    # Printing decodedReceivedAnswer
    print(decodedReceivedAnswer)
else:
    # ERROR MSG
    print('[ERROR]: Did not understand SysINFO...')
    exit()

# Write function (To write and send MSGs)
def Write():
    while True:
        # userInput (Input MSG to send to the server)
        userInput = input()

        # Checking userInput
        if userInput[0] == '/':
            if userInput == '/exit':
                # Sending userInput to the server
                Client_Socket.send(userInput.encode('utf-8'))
                exit()
            elif userInput == '/upload':
                try:
                    # Sending userInput to the server
                    Client_Socket.send(userInput.encode('utf-8'))

                    # userInput (Choose file to upload)
                    userInputFile = input()

                    try:
                        # Opening file
                        fileToOpen = open(f'{userInputFile}', 'r')
                        dataToSend = fileToOpen.read()
                        fileToOpen.close()

                        # Sending dataToSend
                        Client_Socket.send(dataToSend.encode('utf-8'))

                        # Printing INFO
                        print(f'[INFO]: File uploaded!')
                    except:
                        # ERROR MSG
                        print(f'[ERROR]: Can\'t find file...')
                except:
                    # ERROR MSG
                    print(f'[ERROR]: Can\'t upload file...')
            elif userInput == '/download':
                # Sending userInput to the server
                Client_Socket.send(userInput.encode('utf-8'))

                try:
                    # Downloading File
                    receivedDataForFile = Client_Socket.recv(10000000)
                    decodedReceivedDataForFile = receivedDataForFile.decode('utf-8')

                    # Storing decodedReceivedDataForFile
                    fileToStore = open('fileToStore', 'w')
                    fileToStore.write(decodedReceivedDataForFile)
                    fileToStore.close()

                    # Printing INFO
                    print(f'[INFO]: File downloaded!')
                except:
                    # ERROR MSG
                    print(f'[ERROR]: Can\'t download file...')
            elif userInput == '/help':
                # INFO MSG
                print('/upload    -- Upload file to server')
                print('/download  -- Download file from server')
                print('/exit      -- Exit the server')
                print('/help      -- This MSG')
            elif userInput == '//EXIT//':
                # WARNING MSG
                print('[WARNING]: You can not do that! You may demage server!')
                print('[WARNING]: Disconnecting from server for server\'s security...')

                # Disconnecting from the server
                Client_Socket.send('/exit'.encode('utf-8'))
                exit()
            else:
                # ERROR MSG
                print(f'[ERROR]: Command does not exist...')
        else:
            # Sending userInput to the server
            Client_Socket.send(userInput.encode('utf-8'))

# Receive function (To receive MSGs)
def Receive():
    while True:
        # Receiving MSG
        receivedMSG = Client_Socket.recv(10000)
        decodedReceivedMSG = receivedMSG.decode('utf-8')

        # Checking decodedReceivedMSG
        if decodedReceivedMSG == exitINFO:
            exit()
        else:
            print(decodedReceivedMSG)

# Making threads
writeThread = threading.Thread(target=Write)
receciveThread = threading.Thread(target=Receive)

# Staring Theads
writeThread.start()
receciveThread.start()
```

კლიენტის მთელი მუღამი იმაშია, რომ სოკეტ კავშირის გაბმის შემდეგ, ორ წრედს(Thread) ქმნის, რომლებიდანაც ერთი სერვერისთვის ინფორმაციის გაგზავნას ემსახურება, ხოლო მეორე ინფორმაციის სერვერიდან მიღებას და მის STDOUT-ში გადასროლას, ანუ ამ შემთხვევაში ტერმინალში გამოტანას.

> **📝 საინტერესო ფაქტი:**  
> 'stdout' ნიშნავს სტანდარტულ output-ს. ყველა პროცესს(Linux process, Windows process იქნება ეს თუ სხვა, არ აქვს არსებითი მნიშვნელობა) აქვს stdin, stdout და stderr, სადაც, stdin ნიშნავს სტანდარტულ input-ს, stdout, როგორც უკვე ვთქვით, ნიშნავს სტანდარტულ output-ს, ხოლოდ stderr ნიშნავს სტანდარტულ error-ს.

თუ გსურს უფრო დეტალური კოდის ანალიზი შეგიძლია მოცემული კოდი გატესტო. ასევე მოცემულია სერვერის მხარე.

`server.py:`
```python
# Tring to import libraries
try:
    import socket
    import threading
    import os
    from time import sleep as slp
except:
    # ERROR MSG
    print(f'[ERROR]: Can\'t import libraries...')
    exit()

# Adding SysINFO Texts
exitINFO = '//EXIT//'

# Adding vars
HOST = '0.0.0.0' # Change this if you want specific IPv4 address
PORT = 4444
Server_Socket = socket.socket(socket.AF_INET, socket.SOCK_STREAM)
counter1 = 1
MAX_USERS = 5

# Adding lists
clients = []
threads = []
nicknames = []

# Tring to listen
try:
    # Binding Server_Socket
    Server_Socket.bind((HOST, PORT))
    Server_Socket.listen()
except:
    # ERROR MSG
    print('[ERROR]: Can\'t start server...')
    exit()

# MSG when server starts
print(f'[START]: Server listening on {HOST}:{PORT}')

# Handle function (This function will handle one client at the time)
def Handle():
    while True:
        while True:
            # Receiving INFO about client
            client, address = Server_Socket.accept()

            # BroadcastMSG function (This will send MSGs to other clients)
            def BroadcastMSG(msg):
                for Client in clients:
                    if Client == client:
                        pass
                    else:
                        Client.send(msg.encode('utf-8'))

            # Receiving Nickname
            receivedNickname = client.recv(16)
            decodedReceivedNickname = receivedNickname.decode('utf-8')

            # Checking decodedReceivedNickname
            if decodedReceivedNickname in nicknames:
                # Sending ERROR MSG To the client
                client.send(f'[ERROR]: Nickname is already used...'.encode('utf-8'))
                break
            else:
                # Adding decodedReceivedNickname in nicknames list
                nicknames.append(decodedReceivedNickname)

            # Adding clinet in clients list
            clients.append(client)

            # Sending INFO MSG to the client
            client.send(f'[INFO]: You joined the server!'.encode('utf-8'))

            # Printing INFO
            print(f'[INFO]: {decodedReceivedNickname} joined!')

            slp(0.2)

            # Sending USERS to the client
            if len(clients) == 5:
                client.send(f'[USERS]: {nicknames[0]}, {nicknames[1]}, {nicknames[2]}, {nicknames[3]}, {nicknames[4]}'.encode('utf-8'))
            elif len(clients) == 4:
                client.send(f'[USERS]: {nicknames[0]}, {nicknames[1]}, {nicknames[2]}, {nicknames[3]}'.encode('utf-8'))
            elif len(clients) == 3:
                client.send(f'[USERS]: {nicknames[0]}, {nicknames[1]}, {nicknames[2]}'.encode('utf-8'))
            elif len(clients) == 2:
                client.send(f'[USERS]: {nicknames[0]}, {nicknames[1]}'.encode('utf-8'))
            elif len(clients) == 1:
                client.send(f'[USERS]: {nicknames[0]}'.encode('utf-8'))
            else:
                pass

            # Sending MSG to other clients
            BroadcastMSG(f'{decodedReceivedNickname} joined!')

            # Main loop
            while True:
                # Receiving MSG from client
                receivedMSG = client.recv(10000)
                decodedReceivedMSG = receivedMSG.decode('utf-8')

                # Checking decodedReceivedMSG
                if decodedReceivedMSG[0] == '/':
                    if decodedReceivedMSG == exitINFO:
                        # Sending WARNING INFO to the client
                        client.send(f'[WARNING]: You can not do that!'.encode('utf-8'))

                        # Printing WARNING MSG
                        print(f'[WARNING]: {decodedReceivedNickname} tried to leak EXIT_INFO!')
                    elif decodedReceivedMSG == '/exit':
                        # Sending EXIT_INFO to the client (This MSG will turn off 'Receive' function on client side)
                        client.send(exitINFO.encode('utf-8'))

                        # Removing client's INFO from lists
                        nicknames.remove(decodedReceivedNickname)
                        clients.remove(client)

                        # Printing INFO
                        print(f'[INFO]: {decodedReceivedNickname} left!')

                        # Sending MSG to other clients
                        BroadcastMSG(f'{decodedReceivedNickname} left!')
                        break
                    elif decodedReceivedMSG == '/upload':
                        # Receiving file
                        receivedDataForFile = client.recv(10000000)
                        decodedReceivedDataForFile = receivedDataForFile.decode('utf-8')

                        # Storing decodedReceivedDataForFile in File
                        uploadedFile = open('uploadedFile', 'w')
                        uploadedFile.write(decodedReceivedDataForFile)
                        uploadedFile.close()

                        # Printing INFO
                        print(f'[INFO]: {decodedReceivedNickname} uploaded file!')

                        # Sending INFO to other clients
                        BroadcastMSG(f'[INFO]: {decodedReceivedNickname} uploaded file!')
                    elif decodedReceivedMSG == '/download':
                        # Reading File
                        toDownloadFile = open('uploadedFile', 'r')
                        dataToSend = toDownloadFile.read()
                        toDownloadFile.close()

                        # Sending dataToSend to the client
                        client.send(dataToSend.encode('utf-8'))

                        # Printing INFO
                        print(f'[INFO]: {decodedReceivedNickname} downloaded file!')

                        # Sending INFO to other clients
                        BroadcastMSG(f'[INFO]: {decodedReceivedNickname} downloaded file!')
                else:
                    # Sending MSG to other clients
                    BroadcastMSG(f'{decodedReceivedNickname}: {decodedReceivedMSG}')

# Making threads
while counter1 <= MAX_USERS:
    handleThread = threading.Thread(target=Handle)

    threads.append(handleThread)

    counter1 = counter1 + 1

# Starting threads
for thread in threads:
    thread.start()
```

ამ შემთხვევაში, იქმნება ხუთი წრედი თითოეული კლიენტისთვის. საქმე ისაა, რომ thread pool წესივრად არცაა ინტეგრირებული ამ კოდში და ეს პრობლემაც მოსაგვარებელია.

ჩემი აზრით, ყველაზე დიდი პრობლემა ამ პროექტში ისაა, რომ კავშირი არის დაუცველი(ანუ, SSL/TLS არ გვაქვს, კრიპტოგრაფიის მექანიზმები საერთოდ არ არის გამოყენებული და ინფორმაციის უსაფრთხოება საერთოდ არ არის გათვალისწინებული) და კლიენტის მხარეს ე.წ. TUI არ გვაქვს.

> **📝 საინტერესო ფაქტი:**  
> TUI(Terminal User Interface) არის ტერმინალში ინტეგრირებული გრაფიკული მომხმარებლის ინტერფეისი, ანუ GUI.

## ახალი პროექტი

პირველ რიგში, ვფიქრობ საჭიროა `C++` ენის გამოყენება, რადგან ეს მეტ კონტროლს მომცემს კომპიუტერის ქვედა დონეზე(Lower levels). ასე, რომ შევქმნათ ფაილი, Git-ის ინიციალიზაცია მოვახდინოთ და დავიწყოთ კოდის წერა.

- თავიდან შევქმნით `XChat` ფაილს, სადაც ჩავყრით `server` და `client` ფაილებს. ასევე, სწორედ ამ ფაილში გავაკეთებთ Git-ის ინიციალიზაციას:
![1.png](https://44b4c0.github.io/assets/img/posts/1/1.png)
- შემდეგ, გამოვიყენოთ `VSCode` კოდის საწერად. ეგ რომ შევძლო, ამ დირექტორიაშივე `code .` ბრძანება უნდა დავწერო.

მოკლედ, `VSCode` ჩაირთო და შემიძლია უკვე სერვერის წერაზე გადავიდე. ორი ფაილია `server` დირექტორიაში მოცემული(`server.cpp` და `server.hpp`). პირველ ფაილში იქნება `int main()`, ანუ პროგრამის ე.წ "შესავალი"(Entry Point), ხოლო მეორე ფაილში მაკროებს და გარკვეულ ფუნქციებს გავწერ.

> **📝 საინტერესო ფაქტი:**  
> ყველა პროგრამას აქვს "Entry Point". ეს მიუთითებს იმ ლოკაციაზე თუ საიდან უნდა დაიწყოს პროგრამის execution, ანუ საიდან უნდა დაიწყოს CPU-მ კოდის გაშვება.

ჯერჯერობით ეს ბიბლიოთეკები შემოვიტანე:
![2.png](https://44b4c0.github.io/assets/img/posts/1/2.png)

ხო, ზოგადად ლინუქსზე უფრო რთულია სოკეტების წერა, ვიდრე ვინდოუსზე, რადგან ლინუქსზე დამატებით ბიბლიოთეკების შემოტანაა საჭირო, მაგალითად ეს ორი:

```cpp
#include <sys/types.h>
#include <arpa/inet.h>
```

ეს ბიბლიოთეკები იმიტომ მჭირდება ამ შემთხვევაში, რომ პირველი დამატებით data ტიპებზე მომცემს წვდომას, მეორე კი გარკვეული სახის კონვერტაციებს გამიმარტივებს. ვინდოუსის შემთხვევაში საქმე ასე იქნებოდა:

```cpp
#include <WinSock2.h>

#pragma Comment(lib, "ws2_32.lib")
```

ვინდოუსის შემთხვევაში მხოლოდ ერთი ბიბლიოთეკაა საჭირო, მაგრამ მეორე ხაზით ლინკერს უნდა ვუთხრათ თუ რომელი სტატიკური ბიბლიოთეკის დალინკვა გვსურს `WinSock2.h`-სთვის.

სხვა ბიბლიოთეკებს რაც შეეხება, მოგვიანებით განვიხილავ მათ.

მოდი ახლა სერვერს შევქმნი, რომელიც `8080/tcp` პორტზე იმუშავებს და ეს პორტი იქნება `XChat`-ის სტანდარტი:

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

მთელი იდეა იმაში მდგომარეობს, რომ იქნება სერვერი, რომელიც მოისმენს `8080/tcp` პორტზე შემომავალ კავშირს და შემდეგ თითოეულ კლიენტს მოემსახურება როგორც წინასწარ გაწერილ `client` კლასს.
