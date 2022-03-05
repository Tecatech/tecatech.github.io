---
layout: post
title: Разбор месяца про ret2libc и ASLR
---

В этой статье мы разберем пример эксплуатации переполнения буфера на стеке посредством составления ROP-цепочки. Данная задача была опубликована в сообществе [SPbCTF](https://pwn.spbctf.ru/tasks) в рамках разбора, посвященного атакам типа ret2libc и методам обхода ASLR.

![_config.yml]({{ site.baseurl }}/images/pwn-ret2libc-aslr/1.png)

Вначале отключим рандомизацию адресного пространства в текущей операционной системе:

![_config.yml]({{ site.baseurl }}/images/pwn-ret2libc-aslr/2.png)

Загрузим исполняемый файл в IDA Pro и изучим листинг функции `main`:

![_config.yml]({{ site.baseurl }}/images/pwn-ret2libc-aslr/3.png)

Таким образом, исходный код программы содержит ряд уязвимостей:

- функция форматированного вывода `printf` используется без спецификаторов формата, то есть имеется уязвимость форматной строки;
- для считывания данных из стандартного ввода применяется функция `gets`, которая не позволяет установить ограничение на размер считываемой строки, то есть имеется уязвимость переполнения буфера на стеке.

Отобразим кадр стека функции main в IDA Pro:

![_config.yml]({{ site.baseurl }}/images/pwn-ret2libc-aslr/4.png)

Теперь запустим исполняемый файл и прочитаем значение из стека с помощью функции `printf`:

![_config.yml]({{ site.baseurl }}/images/pwn-ret2libc-aslr/5.png)

Для проведения атаки ret2libc необходимо получить адрес какой-либо функции из библиотеки libc:

![_config.yml]({{ site.baseurl }}/images/pwn-ret2libc-aslr/6.png)

Откроем ассемблерный листинг функции `main` в отладчике gdb:

![_config.yml]({{ site.baseurl }}/images/pwn-ret2libc-aslr/7.png)

Запустим программу в отладчике для нахождения требуемого адреса:

![_config.yml]({{ site.baseurl }}/images/pwn-ret2libc-aslr/8.png)

В качестве седьмого параметра на стеке используется адрес, указывающий на инструкцию `xor r8d, r8d` функции `__GI__IO_setvbuf`:

![_config.yml]({{ site.baseurl }}/images/pwn-ret2libc-aslr/9.png)

Загрузим файл динамической библиотеки в IDA Pro и убедимся в корректности этого адреса памяти:

![_config.yml]({{ site.baseurl }}/images/pwn-ret2libc-aslr/10.png)

Создадим первую версию эксплойта и вычислим базовый адрес libc через уязвимость форматной строки:

![_config.yml]({{ site.baseurl }}/images/pwn-ret2libc-aslr/11.png)

![_config.yml]({{ site.baseurl }}/images/pwn-ret2libc-aslr/12.png)

Таким образом, полезная нагрузка имеет следующую структуру:

- `128` байтов «‎мусора» (заполнение буфера `format`)
- `8` байтов «‎мусора» (перезапись значения регистра `rbp`)
- адрес инструкции `ret` (выравнивание стека)
- адрес гаджета `pop rdi; ret` (вызов функции `system` в `cdecl`)
- адрес строки `/bin/sh` (передача аргумента в функцию `system`)
- адрес функции `system` (запуск оболочки)

С помощью консольных утилит получим адреса гаджетов и строки `/bin/sh`:

![_config.yml]({{ site.baseurl }}/images/pwn-ret2libc-aslr/13.png)

Имеем аналогичный результат при использовании IDA Pro:

![_config.yml]({{ site.baseurl }}/images/pwn-ret2libc-aslr/14.png)

![_config.yml]({{ site.baseurl }}/images/pwn-ret2libc-aslr/15.png)

![_config.yml]({{ site.baseurl }}/images/pwn-ret2libc-aslr/16.png)

Затем найдем адрес функции `system` в файле динамической библиотеки:

![_config.yml]({{ site.baseurl }}/images/pwn-ret2libc-aslr/17.png)

Итак, соберем вместе все этапы эксплуатации уязвимости:

![_config.yml]({{ site.baseurl }}/images/pwn-ret2libc-aslr/18.png)

Запустим эксплойт и получим shell на удаленной машине:

![_config.yml]({{ site.baseurl }}/images/pwn-ret2libc-aslr/19.png)

Полный код эксплойта доступен на [GitHub](https://github.com/Tecatech/information-security-training/blob/main/2020/SPbCTF/Binary%20Exploitation/ret2libc_ASLR/cleaker/cleaker_exploit.py)