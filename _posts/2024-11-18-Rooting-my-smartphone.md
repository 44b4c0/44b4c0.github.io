---
title: "Samsung J3 (SM-J330L) დარუთვა"
date: 2024-11-18 16:32:00 +0400
categories: [Miscellaneous]
tags: [Root]
---

## ⚠️გაფრთხილება⚠️

**არ დარუთოთ ტელეფონი თუ არ იცით რას აკეთებთ. ამან შეიძლება დააზიანოს ან საერთოდ გაანადგუროს/გააფუჭოს თქვენი სმარტფონი.**

## დასაწყისი

მოკლედ, ხელში ჩამივარდა, `Samsung J3 (SM-J330L)` და მისი დარუთვა გადავწყვიტე.

#### რა არის ტელეფონის დარუთვა? ზოგადად რუთი (root) ?

`Root`-ის სისტემაზე ქონა ნიშნავს სისტემაზე სრული წვდომის მოპოვებას. ნაგულისხმევად, ანდროიდი ასეთი მეთოდით მუშაობს: არის კერნელი და არის თითოეული აპლიკაციისათვის გამოყოფილი სივრცე, სადაც შეიძლება, რომ ეს აპლიკაციები ჩაიტვირთონ და გაეშვან. პრობლემა ისაა, რომ როდესაც აპლიკაციიდან რაღაც ისეთის გაკეთებას ვცდილობ, რომელიც მაღალ პრივილეგიებს მოითხოვს, არ გამომდის, რადგან ნაგულისხმევად ტელეფონები ამის საშუალებას არ იძლევიან. კერნელი ბლოკავს ყველა ასეთ მოთხოვნას `Kernel API`-ზე.

ამის გამო, მომიწია ტელეფონის დარუთვა.

#### რატომ გადავწყვიტე დარუთვა?

პირველ რიგში იმიტომ, რომ `Termux`-ის გამოყენებას დიდად აზრი არ აქვს `Root`-ის გარეშე. რა თქმა უნდა, იმას არ ვამბობ, რომ `Termux` მხოლოდ პენტესტინგისა და `Red Teaming`-სთვის გამოიყენება, მაგრამ მე უმეტეს დროს სწორედ ამ საქმეს ვუთმობ. ლეპტოპი კი როგორი კომპაქტურიც არ უნდა მქონდეს(`Lenovo Thinkpad E14 Gen5`), ტელეფონთან ახლოსაც ვერ მივა როდესაც საქმე კომპაქტურობას ეხება. `16GB` შიდა მეხსიერება აქვს ტელეფონს, მაგრამ მაგის მოგვარება `SD Card`-ით შეიძლება, რომელიც `1 TB`-იც კი შეიძლება იყოს.

`Root` მჭირდება `Wireless` შეტევებისათვის, როგორიცაა `Bluetooth` და `Wi-Fi` შეტევები. თუ გარე ადაპტერი ვიშოვე, `RFID` შეტევებიც შესაძლებელი იქნება.

## რა მჭირდება?

მჭირდება სულ ორი რამ.

1. TWRP
2. Magisk.zip

მოკლედ, `TWRP` არის აღდგენის(Recovery) ხელსაწყო, რომლიდანაც აიტვირთება `Magisk.zip`, სადაც დევს `Magisk image`.

#### TWRP

თავდაპირველად, კომპიუტერში დავაყენე `Odin`, მივაერთე ტელეფონი, `TWRP.tar` დავაყენე `AP` სექციაში ოფციად, ბოლოს კი ტელეფონში ავტვირთე.

ტელეფონში `TWRP` ასე იტვირთება: საჭიროა ტელეფონში დეველოპერის ოფციების გახსნა და `OEM Unlock` სექციის ჩართვა. ამის შემდეგ, უნდა გამოირთოს ტელეფონი და ამ კონკრეტული მოდელის შემთხვევაში ჩართვისათვის უნდა დავაჭირო `Power` + `Home` + `Volume Down` ღილაკებს. ამის შემდეგ, ტელეფონი გადადის `Download mode`-ში, რის შემდეგაც უნდა გაკეთდეს ზემოთ მოცემული ქმედებები, `Odin`-დან.

#### Magisk

როდესაც `TWRP`-ს ჩაწერას მოვრჩი, ტელეფონი გამოვრთე და იგივენაირად ჩავრთე, მაგრამ ახლა `Volume Down` ღილაკი `Volume Up`-ით ჩავანაცვლე და გადავედი `TWRP` პანელში. აქ თავდაპირველად `Backup`-ის შექმნაა საჭირო. მოვნიშნე `System` და `EFS` `backup`-სთვის და დავიწყე მათი შენახვა. ამის შემდეგ, `TWRP`-ში ჩავრთე `adb sideloading` და `Windows`-დან გავუშვი ეს ბრძანება:

```bash
adb sideload "C:\\Magisk.zip"
```

ამ ბრძანების შემდეგ, ტელეფონმა `Magisk.zip` დაწერა სისტემაზე. ეს კი იმას ნიშნავს, რომ სტანდარტული სისტემა ჩანაცვლდა მოდიფიცირებული, `Magisk`-ის სისტემით.

ტელეფონი გამოირთო და თავიდან ჩავრთე. 10-15 წუთი დასჭირდა ბოლომდე ჩართვას და `Factory Reset` გააკეთა, რის გამოც ყველანაირი ინფორმაცია წაიშალა ტელეფონიდან.

ტელეფონში სხვა ნაგულისხმევი აპლიკაციების გვერდით, გამოჩნდა `Magisk` აპლიკაცია, რომელიც `Root` მენეჯერია. სწორედ აქედან ხდება მისი მენეჯმენტი და იმ აპლიკაციებზე წვდომის მიცემა, რომელთა გამოყენებაც `Root`-ის სახელით მსურს. დანარჩენი აპლიკაციები სტანდარტულად, თავიანთ "კოლბაში" გაეშვებიან.

## დარუთვის შემდეგ...

გადმოვწერე `Termux` და გავუშვი შემდეგი ბრძანებები:

```bash
pkg update && pkg upgrade
pkg install tsu
pkg install root-repo
pkg update && pkg upgrade
tsu
```

ბოლო ბრძანების გაშვების შემდეგ, პრომპტი შეიცვალა და გავხდი `root` მომხმარებელი. ახლა ნებისმიერი რამის გაკეთება შემიძლია ტელეფონზე.
