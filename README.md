# Домашнее задание к занятию "`Защита сети`" - `Евгений Головенко`
------

### Подготовка к выполнению заданий

1. Подготовка защищаемой системы:

- установите **Suricata**,
- установите **Fail2Ban**.

2. Подготовка системы злоумышленника: установите **nmap** и **thc-hydra** либо скачайте и установите **Kali linux**.

Обе системы должны находится в одной подсети.

------

### Задание 1

Проведите разведку системы и определите, какие сетевые службы запущены на защищаемой системе:

**sudo nmap -sA < ip-адрес >**

**sudo nmap -sT < ip-адрес >**

**sudo nmap -sS < ip-адрес >**

**sudo nmap -sV < ip-адрес >**

По желанию можете поэкспериментировать с опциями: https://nmap.org/man/ru/man-briefoptions.html.


*В качестве ответа пришлите события, которые попали в логи Suricata и Fail2Ban, прокомментируйте результат.*

### Решение 1

**ACK scan: sudo nmap -sA 10.10.10.4**

<img width="466" height="153" alt="net sec_1" src="https://github.com/user-attachments/assets/24969fab-3605-4526-8475-422bc034f72f" />

Данное сканирование помогает определить, имеется ли на атакуемой системе statefull firewall.

Возможные состояния портов:
- unfiltered
- filtered
- ignored

На скрине видно, что все порты ответили ignored. Это может означать, что либо firewall отсутствует, либо пропускает ACK, либо атакуемый хост вообще ресетит все подряд. Фильтрация на этом этапе не обнаружена. 

**TCP connect scan: sudo nmap -sT 10.10.10.4**

<img width="509" height="178" alt="net sec_2" src="https://github.com/user-attachments/assets/9e0ae202-003d-476f-8c30-f2b79a503527" />

На скрине видны два открытых порта:
- 22/tcp (ssh) - сервис доступен, полный TCP handshake прошел, значит можно пробовать аутентификацию
- 80/tcp (http) - есть веб-сервис и он слушает. Если повезет, можно получить shell.

Остальные 998 портов закрыты. Значит хост в эти порты отвечает RST, порты доступны, но сервисы на них не висят. И firewall не пытается блокировать соединения.

**SYN scan: sudo nmap -sS 10.10.10.4**

<img width="522" height="180" alt="net sec_3" src="https://github.com/user-attachments/assets/cc1596fb-d50f-4aff-882c-4859975fc0b7" />

Результат сканирования внешне схож с TCP connect scan. Те же 2 открытых порта. Но 998 закрытых портов в предыдущем случае получили `connection refused`. Сейчас они `reset`. В принципе, результат схож, но получен разными путями.

В случае *-sT* `nMap` говорит ядру ОС установить соединение. ОС направляет в целевой порт `SYN` и получает обратно `RST` (если порт закрыт). ОС интерпретирует это как отказ и сообщает `nMap` `connection refused`.

В случае *-sS* `nMap` уже сам запрашивает соединение `SYN`, не завершает его (ключевое различие) и получает обратно `RST` (если порт закрыт) или `SYN+ACK` (если порт открыт). В итоге, это менее шумный метод сканирования, не оставляет логи соединений.

**Version scan: sudo nmap -sV 10.10.10.4**

<img width="655" height="206" alt="net sec_4" src="https://github.com/user-attachments/assets/db9a62cc-0196-45a7-a9e6-4789b9ca3015" />

На атакуемом хосте имеется:
- ОС: Debian
- Версия OpenSSH: 10.2p2, протокол 2.0 (свежий)
- web сервер ngnix.

------

### Задание 2

Проведите атаку на подбор пароля для службы SSH:

**hydra -L users.txt -P pass.txt < ip-адрес > ssh**

1. Настройка **hydra**: 
 
 - создайте два файла: **users.txt** и **pass.txt**;
 - в каждой строчке первого файла должны быть имена пользователей, второго — пароли. В нашем случае это могут быть случайные строки, но ради эксперимента можете добавить имя и пароль существующего пользователя.

Дополнительная информация по **hydra**: https://kali.tools/?p=1847.

2. Включение защиты SSH для Fail2Ban:

-  открыть файл /etc/fail2ban/jail.conf,
-  найти секцию **ssh**,
-  установить **enabled**  в **true**.

Дополнительная информация по **Fail2Ban**:https://putty.org.ru/articles/fail2ban-ssh.html.

*В качестве ответа пришлите события, которые попали в логи Suricata и Fail2Ban, прокомментируйте результат.*

### Решение 2

1. Настройка и проверка работы **hydra**

Для чистоты эксперимента на жертвенном хосте отключен fail2ban, настройки ssh по умолчению (MaxAuthTries=6, MaxSessions=10).

В `~/hydra/` создаются файлы с рандомными именами пользователей и паролями (users.txt, pass.txt).

Результат тестового брутфорсинга `hydra -L users.txt -P pass.txt -t 1 -V ssh://10.10.10.4`:

<img width="727" height="443" alt="net sec_5" src="https://github.com/user-attachments/assets/3c3db33b-3cd2-4b5e-ae3f-aab0782de7f0" />

2. Включение защиты SSH (Fail2Ban):

*Конфигурация Fail2Ban выполнялась через файл `jail.local`, как рекомендовано официальной документацией, поскольку файл `jail.conf` является шаблонным и не предназначен для прямого редактирования.*

`sudo nano /etc/fail2ban/jail.local`
```cmd
[sshd]
enabled = true
port = ssh
logpath = %(sshd_log)s
backend = systemd
maxretry = 3
findtime = 1m
bantime = 5m

[nginx-http-auth]
enabled = true

[nginx-botsearch]
enabled = true
```
Проверка:

<img width="564" height="75" alt="net sec_6" src="https://github.com/user-attachments/assets/6987b11d-eed5-44e1-8ff9-e758a3590585" />

Запускаем брутфорсинг `ssh`

<img width="728" height="553" alt="net sec_7" src="https://github.com/user-attachments/assets/92069551-684b-4632-9731-239996214d1f" />

Видно, что атака обнаружена и IP банится.

<img width="854" height="188" alt="net sec_8" src="https://github.com/user-attachments/assets/ca773ab7-fd37-4289-9986-c04d4ed2b4fd" />

<img width="997" height="534" alt="net sec_9" src="https://github.com/user-attachments/assets/417daa48-0c3d-4b7b-9d95-f48e5c66c8a6" />

Что можно увидеть в логах `suricate`:
<img width="1094" height="533" alt="net sec_10" src="https://github.com/user-attachments/assets/9ff6ec5a-73c0-4c4b-96d8-03db30a7ece4" />

Незакрашенное - шум от SYN scan, желтое - брутфорсинг в SSH.
